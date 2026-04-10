<script setup>
import { computed, onBeforeUnmount, reactive, ref, watch } from 'vue'
import '@/components/businesss.css'
import { saveBusinessInterviewSchedule, subscribeToBusinessInterviewSchedules } from '@/lib/business_interviews'
import { updateApplicantJobApplicationStatus } from '@/lib/apply_jobs'

const props = defineProps({
  activeSection: { type: String, default: 'interview-scheduling' },
  businessJobApplications: { type: Array, default: () => [] },
  workspaceOwnerId: { type: String, default: '' },
  businessName: { type: String, default: 'Business Workspace' },
})

const text = (value = '') => String(value ?? '').trim()
const normalizeValue = (value = '') => text(value).toLowerCase()
const formatDateTime = (value = '') => {
  const parsed = Date.parse(String(value || '').trim())
  if (!Number.isFinite(parsed)) return 'Not scheduled'
  return new Date(parsed).toLocaleString('en-US', { month: 'short', day: '2-digit', year: 'numeric', hour: 'numeric', minute: '2-digit' })
}
const formatActivityLabel = (value = '') => {
  const parsed = Date.parse(String(value || '').trim())
  if (!Number.isFinite(parsed)) return 'Live updates pending'

  const diffMs = Math.max(0, Date.now() - parsed)
  const diffMinutes = Math.round(diffMs / (60 * 1000))
  if (diffMinutes < 1) return 'Updated just now'
  if (diffMinutes < 60) return `Updated ${diffMinutes}m ago`

  const diffHours = Math.round(diffMs / (60 * 60 * 1000))
  if (diffHours < 24) return `Updated ${diffHours}h ago`

  return `Updated ${new Date(parsed).toLocaleDateString('en-US', { month: 'short', day: '2-digit', year: 'numeric' })}`
}
const toIsoString = (value = '') => {
  const parsed = Date.parse(String(value || '').trim())
  return Number.isFinite(parsed) ? new Date(parsed).toISOString() : ''
}

const liveInterviewSchedules = ref([])
const isSyncingSchedules = ref(false)
const isSavingSchedule = ref(false)
const activeStatusActionId = ref('')
const isDecisionModalOpen = ref(false)
const decisionModalMode = ref('review')
const selectedInterviewRowId = ref('')
const formError = ref('')
const formMessage = ref('')
const selectedDay = ref(new Date().getDate())
const currentDate = ref(new Date())
const form = reactive({
  applicantId: '',
  schedule: '',
  mode: 'Online',
  locationOrLink: '',
  interviewer: 'Hiring Manager',
  notes: '',
})
const decisionForm = reactive({
  rejectionReason: '',
  availableDates: [''],
  failureReason: '',
})

let stopInterviewSchedulesSubscription = () => {}

const getActivityTime = (record = {}) =>
  Math.max(
    Date.parse(String(record?.updatedAt || record?.updated_at || '').trim()) || 0,
    Date.parse(String(record?.applicantRespondedAt || record?.applicant_responded_at || '').trim()) || 0,
    Date.parse(String(record?.businessDecidedAt || record?.business_decided_at || '').trim()) || 0,
    Date.parse(String(record?.scheduledAt || record?.scheduled_at || '').trim()) || 0,
    Date.parse(String(record?.createdAt || record?.created_at || '').trim()) || 0,
  )

const latestInterviewByApplication = computed(() => {
  const lookup = new Map()
  liveInterviewSchedules.value.forEach((record) => {
    const applicationId = text(record?.applicationId || record?.application_id)
    if (!applicationId) return
    const existing = lookup.get(applicationId)
    if (!existing || getActivityTime(record) >= getActivityTime(existing)) lookup.set(applicationId, record)
  })
  return lookup
})

const resolveInterviewStatus = (record = {}) => {
  const scheduleStatus = normalizeValue(record?.scheduleStatus || record?.schedule_status || 'scheduled')
  const responseStatus = normalizeValue(record?.applicantResponseStatus || record?.applicant_response_status || 'pending')
  if (scheduleStatus === 'completed') return 'Completed'
  if (scheduleStatus === 'cancelled') return 'Cancelled'
  if (responseStatus === 'confirmed') return 'Confirmed'
  if (responseStatus === 'reschedule_requested') return 'Reschedule Requested'
  if (responseStatus === 'reschedule_rejected') return 'Reschedule Rejected'
  return 'Scheduled'
}

const resolveApplicantResponseLabel = (record = {}) => {
  const scheduleStatus = normalizeValue(record?.scheduleStatus || record?.schedule_status || 'scheduled')
  const responseStatus = normalizeValue(record?.applicantResponseStatus || record?.applicant_response_status || 'pending')

  if (scheduleStatus === 'completed') return 'Applicant finished the interview'
  if (scheduleStatus === 'cancelled') return 'Interview closed by business'
  if (responseStatus === 'confirmed') return 'Applicant confirmed attendance'
  if (responseStatus === 'reschedule_requested') return 'Applicant requested a reschedule'
  if (responseStatus === 'reschedule_rejected') return 'Reschedule request was not approved'
  return 'Waiting for applicant response'
}

const resolveApplicantResponseMeta = (record = {}) => {
  const responseStatus = normalizeValue(record?.applicantResponseStatus || record?.applicant_response_status || 'pending')
  const responseReason = text(record?.applicantResponseReason || record?.applicant_response_reason)
  const decisionReason = text(record?.businessDecisionReason || record?.business_decision_reason)
  const requestedScheduleAt = text(record?.requestedScheduleAt || record?.requested_schedule_at)

  if (responseStatus === 'confirmed') {
    return `Confirmed for ${formatDateTime(record?.scheduledAt || record?.scheduled_at)}`
  }
  if (responseStatus === 'reschedule_requested') {
    return responseReason
      ? `${responseReason}${requestedScheduleAt ? ` | Requested: ${formatDateTime(requestedScheduleAt)}` : ''}`
      : requestedScheduleAt
        ? `Requested: ${formatDateTime(requestedScheduleAt)}`
        : 'Applicant asked for a different interview time.'
  }
  if (responseStatus === 'reschedule_rejected') {
    return decisionReason || 'The current interview request will not continue.'
  }
  if (normalizeValue(record?.scheduleStatus || record?.schedule_status) === 'cancelled') {
    return decisionReason || 'This interview was closed in the live workspace.'
  }
  return 'The applicant can confirm or request a reschedule from the Interviews page.'
}

const resolveStatusBadgeClass = (status = '') => {
  const normalized = normalizeValue(status)
  if (normalized === 'completed' || normalized === 'confirmed') return 'is-completed'
  if (normalized === 'cancelled') return 'is-cancelled'
  if (normalized.includes('reschedule')) return 'is-pending'
  if (normalized === 'scheduled') return 'is-scheduled'
  return 'is-ready'
}

const showSchedulingView = computed(() => props.activeSection === 'interview-scheduling')
const showStatusView = computed(() => props.activeSection === 'interview-status')

const normalizeWorkflowStatus = (row = {}) => {
  const applicationStatus = normalizeValue(row?.applicationStatus)
  const scheduleStatus = normalizeValue(row?.scheduleRecord?.scheduleStatus || row?.scheduleRecord?.schedule_status || '')
  const responseStatus = normalizeValue(row?.scheduleRecord?.applicantResponseStatus || row?.scheduleRecord?.applicant_response_status || '')

  if (applicationStatus === 'rejected' || responseStatus === 'reschedule_rejected') return 'Failed'
  if (scheduleStatus === 'completed') return 'Completed'
  if (responseStatus === 'confirmed') return 'Confirmed'
  if (responseStatus === 'reschedule_requested') return 'Resched'
  if (row?.scheduleRecord?.id) return 'Sent'
  return 'Pending'
}

const normalizeWorkflowClass = (status = '') => {
  const normalizedStatus = normalizeValue(status)
  if (normalizedStatus === 'completed' || normalizedStatus === 'confirmed') return 'is-completed'
  if (normalizedStatus === 'failed') return 'is-cancelled'
  if (normalizedStatus === 'resched') return 'is-pending'
  if (normalizedStatus === 'sent') return 'is-scheduled'
  return 'is-ready'
}

const interviewRows = computed(() =>
  (Array.isArray(props.businessJobApplications) ? props.businessJobApplications : [])
    .map((item, index) => {
      const applicationId = text(item?.id || item?.applicationId)
      if (!applicationId) return null

      const liveSchedule = latestInterviewByApplication.value.get(applicationId) || null
      const status = normalizeValue(item?.status)
      if (!(status.includes('approve') || status.includes('interview') || liveSchedule)) return null

      const scheduledAt = text(liveSchedule?.scheduledAt || liveSchedule?.scheduled_at || item?.interviewSchedule || item?.interview_schedule || item?.interviewDate || item?.interview_date)

      return {
        id: applicationId,
        applicationId,
        applicationStatus: status,
        applicantId: text(item?.applicantId || item?.applicant_id || item?.userId || item?.user_id),
        applicantName: text(item?.applicantName || item?.applicant_name || item?.name) || `Applicant ${index + 1}`,
        applicantEmail: text(item?.applicantEmail || item?.applicant_email || item?.email) || 'No email',
        applicantAvatar: text(item?.applicantAvatar || item?.applicant_avatar || item?.avatar || item?.avatar_url),
        jobId: text(item?.jobId || item?.job_id),
        jobTitle: text(item?.jobTitle || item?.job_title || item?.role) || 'Open role',
        workspaceOwnerEmail: text(item?.workspaceOwnerEmail || item?.workspace_owner_email),
        workspaceOwnerName: text(item?.workspaceOwnerName || item?.workspace_owner_name || item?.companyName || item?.company_name || item?.businessName || item?.business_name || props.businessName) || props.businessName,
        scheduleRecord: liveSchedule,
        scheduledAt,
        scheduledAtLabel: formatDateTime(scheduledAt),
        interviewer: text(liveSchedule?.interviewer || item?.interviewer || 'Hiring Manager'),
        modeLabel: text(liveSchedule?.mode || liveSchedule?.interviewMode || item?.interviewMode || item?.interview_mode || 'Online') || 'Online',
        locationLabel: text(liveSchedule?.locationOrLink || liveSchedule?.location_or_link || item?.interviewLocationOrLink || item?.interview_location_or_link) || 'Location or meeting link will appear here.',
        notesLabel: text(liveSchedule?.notes || item?.interviewNotes || item?.interview_notes) || 'No interview notes yet.',
        interviewStatus: liveSchedule ? resolveInterviewStatus(liveSchedule) : scheduledAt ? 'Scheduled' : 'Ready',
        applicantStatusLabel: liveSchedule ? resolveApplicantResponseLabel(liveSchedule) : 'Waiting for applicant response',
        applicantStatusMeta: liveSchedule ? resolveApplicantResponseMeta(liveSchedule) : 'Create a live interview schedule first.',
        updatedAtLabel: formatActivityLabel(
          liveSchedule?.updatedAt
            || liveSchedule?.updated_at
            || liveSchedule?.applicantRespondedAt
            || liveSchedule?.applicant_responded_at
            || liveSchedule?.scheduledAt
            || liveSchedule?.scheduled_at,
        ),
      }
    })
    .filter(Boolean),
)

const interviewStatusRows = computed(() =>
  interviewRows.value
    .filter((row) => row.scheduleRecord?.id)
    .map((row) => {
      const workflowStatus = normalizeWorkflowStatus(row)
      const responseStatus = normalizeValue(row?.scheduleRecord?.applicantResponseStatus || row?.scheduleRecord?.applicant_response_status || '')

      return {
        ...row,
        workflowStatus,
        workflowClass: normalizeWorkflowClass(workflowStatus),
        canReviewReschedule: responseStatus === 'reschedule_requested',
        canMarkOutcome: responseStatus === 'confirmed' && workflowStatus !== 'Completed' && workflowStatus !== 'Failed',
        actionLabel: responseStatus === 'reschedule_requested'
          ? 'View Reason'
          : responseStatus === 'confirmed'
            ? 'Set Outcome'
            : 'View Details',
      }
    }),
)

const selectedInterviewRow = computed(() =>
  interviewStatusRows.value.find((row) => row.id === selectedInterviewRowId.value) || null,
)

const stats = computed(() => ({
  ready: interviewRows.value.filter((item) => item.interviewStatus === 'Ready').length,
  scheduled: interviewRows.value.filter((item) => ['Scheduled', 'Confirmed', 'Reschedule Requested'].includes(item.interviewStatus)).length,
  completed: interviewRows.value.filter((item) => item.interviewStatus === 'Completed').length,
}))

const applicantOptions = computed(() => interviewRows.value.map((item) => ({ value: item.applicationId, label: `${item.applicantName} - ${item.jobTitle}` })))
const selectedApplicant = computed(() => interviewRows.value.find((item) => item.applicationId === text(form.applicantId)) || null)

const monthLabel = computed(() => currentDate.value.toLocaleString('en-US', { month: 'long', year: 'numeric' }))
const weekdayLabels = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat']
const calendarDays = computed(() => {
  const year = currentDate.value.getFullYear()
  const month = currentDate.value.getMonth()
  const firstDay = new Date(year, month, 1).getDay()
  const daysInMonth = new Date(year, month + 1, 0).getDate()
  const days = []
  for (let index = 0; index < firstDay; index += 1) days.push({ key: `empty-${index}`, label: '', muted: true, schedules: 0 })
  for (let day = 1; day <= daysInMonth; day += 1) {
    const schedules = interviewRows.value.filter((item) => {
      const parsed = Date.parse(item?.scheduledAt || '')
      if (!Number.isFinite(parsed)) return false
      const date = new Date(parsed)
      return date.getFullYear() === year && date.getMonth() === month && date.getDate() === day
    }).length
    days.push({ key: `day-${day}`, label: day, muted: false, isToday: day === new Date().getDate() && month === new Date().getMonth() && year === new Date().getFullYear(), isSelected: day === selectedDay.value, schedules })
  }
  return days
})

const selectedDaySchedules = computed(() =>
  interviewRows.value.filter((item) => {
    const parsed = Date.parse(item?.scheduledAt || '')
    return Number.isFinite(parsed) && new Date(parsed).getDate() === selectedDay.value
  }),
)

const resetForm = () => {
  form.applicantId = ''
  form.schedule = ''
  form.mode = 'Online'
  form.locationOrLink = ''
  form.interviewer = 'Hiring Manager'
  form.notes = ''
  formError.value = ''
}

const resetDecisionForm = () => {
  decisionForm.rejectionReason = ''
  decisionForm.availableDates = ['']
  decisionForm.failureReason = ''
}

const openDecisionModal = (row = {}, mode = 'review') => {
  selectedInterviewRowId.value = text(row?.id)
  decisionModalMode.value = mode
  isDecisionModalOpen.value = true
  resetDecisionForm()
}

const closeDecisionModal = (force = false) => {
  if (activeStatusActionId.value && force !== true) return
  isDecisionModalOpen.value = false
  decisionModalMode.value = 'review'
  selectedInterviewRowId.value = ''
  resetDecisionForm()
}

const addAvailableDateOption = () => {
  decisionForm.availableDates = [...decisionForm.availableDates, '']
}

const removeAvailableDateOption = (indexToRemove) => {
  decisionForm.availableDates = decisionForm.availableDates.filter((_, index) => index !== indexToRemove)
  if (!decisionForm.availableDates.length) decisionForm.availableDates = ['']
}

const updateAvailableDateOption = (indexToUpdate, nextValue) => {
  decisionForm.availableDates = decisionForm.availableDates.map((value, index) => index === indexToUpdate ? String(nextValue || '').trim() : value)
}

const upsertLiveInterview = (record = {}) => {
  const normalizedId = text(record?.id)
  const normalizedApplicationId = text(record?.applicationId || record?.application_id)
  const existingIndex = liveInterviewSchedules.value.findIndex((item) => text(item?.id) === normalizedId || text(item?.applicationId || item?.application_id) === normalizedApplicationId)
  if (existingIndex >= 0) {
    liveInterviewSchedules.value = liveInterviewSchedules.value.map((item, index) => index === existingIndex ? record : item)
    return
  }
  liveInterviewSchedules.value = [record, ...liveInterviewSchedules.value]
}

const syncInterviewRecordAndApplication = async (row = {}, nextInterviewPayload = {}, nextApplicationPayload = {}) => {
  const savedInterview = await saveBusinessInterviewSchedule({
    ...(row?.scheduleRecord || {}),
    ...nextInterviewPayload,
  })

  await updateApplicantJobApplicationStatus(row.applicationId, {
    interviewType: 'interview',
    ...nextApplicationPayload,
  })

  upsertLiveInterview(savedInterview)
  return savedInterview
}

const approveRescheduleRequest = async () => {
  const row = selectedInterviewRow.value
  if (!row?.scheduleRecord?.id || activeStatusActionId.value) return

  const availableDates = decisionForm.availableDates
    .map((value) => toIsoString(value))
    .filter(Boolean)

  if (!availableDates.length) {
    formMessage.value = ''
    formError.value = 'Add at least one available date before approving the reschedule request.'
    return
  }

  const primaryDate = availableDates[0]

  try {
    activeStatusActionId.value = row.id
    formError.value = ''
    await syncInterviewRecordAndApplication(
      row,
      {
        scheduledAt: primaryDate,
        scheduleStatus: 'scheduled',
        applicantResponseStatus: 'pending',
        applicantResponseReason: '',
        requestedScheduleAt: '',
        businessDecisionReason: 'Reschedule request approved. Review the new date options and confirm your preferred schedule.',
        businessDecidedAt: new Date().toISOString(),
        availableScheduleOptions: availableDates,
      },
      {
        status: 'interview',
        interviewSchedule: primaryDate,
        interviewDate: primaryDate,
        interviewer: row.interviewer,
        interviewMode: row.modeLabel,
        interviewLocationOrLink: row.locationLabel,
        interviewNotes: row.notesLabel,
        interviewResponseStatus: 'pending',
        interviewRescheduleReason: '',
        interviewRequestedScheduleAt: '',
        interviewDecisionReason: 'Reschedule request approved. Choose one of the new interview dates.',
        interviewRespondedAt: '',
        interviewDecidedAt: new Date().toISOString(),
        interviewScheduleOptions: availableDates,
      },
    )
    formMessage.value = `Reschedule dates sent to ${row.applicantName}. The applicant can now choose and confirm one date in real time.`
    closeDecisionModal(true)
  } catch (error) {
    formError.value = error instanceof Error ? error.message : 'Unable to approve this reschedule request right now.'
  } finally {
    activeStatusActionId.value = ''
  }
}

const rejectRescheduleRequest = async () => {
  const row = selectedInterviewRow.value
  const rejectionReason = text(decisionForm.rejectionReason)
  if (!row?.scheduleRecord?.id || activeStatusActionId.value) return
  if (!rejectionReason) {
    formError.value = 'Enter a rejection reason before declining the reschedule request.'
    return
  }

  try {
    activeStatusActionId.value = row.id
    formError.value = ''
    await syncInterviewRecordAndApplication(
      row,
      {
        scheduleStatus: 'cancelled',
        applicantResponseStatus: 'reschedule_rejected',
        businessDecisionReason: rejectionReason,
        businessDecidedAt: new Date().toISOString(),
        availableScheduleOptions: [],
      },
      {
        status: 'rejected',
        rejectionReason,
        interviewSchedule: text(row?.scheduleRecord?.scheduledAt || row?.scheduleRecord?.scheduled_at),
        interviewDate: text(row?.scheduleRecord?.scheduledAt || row?.scheduleRecord?.scheduled_at),
        interviewer: row.interviewer,
        interviewMode: row.modeLabel,
        interviewLocationOrLink: row.locationLabel,
        interviewNotes: row.notesLabel,
        interviewResponseStatus: 'reschedule_rejected',
        interviewRescheduleReason: text(row?.scheduleRecord?.applicantResponseReason || row?.scheduleRecord?.applicant_response_reason),
        interviewRequestedScheduleAt: text(row?.scheduleRecord?.requestedScheduleAt || row?.scheduleRecord?.requested_schedule_at),
        interviewDecisionReason: rejectionReason,
        interviewDecidedAt: new Date().toISOString(),
        interviewScheduleOptions: [],
      },
    )
    formMessage.value = `Reschedule request rejected for ${row.applicantName}. The applicant will receive the reason in the interview page and notifications.`
    closeDecisionModal(true)
  } catch (error) {
    formError.value = error instanceof Error ? error.message : 'Unable to reject this reschedule request right now.'
  } finally {
    activeStatusActionId.value = ''
  }
}

const markInterviewCompleted = async (row = {}) => {
  if (!row?.scheduleRecord?.id || activeStatusActionId.value) return

  try {
    activeStatusActionId.value = row.id
    formError.value = ''
    await syncInterviewRecordAndApplication(
      row,
      {
        scheduleStatus: 'completed',
        applicantResponseStatus: 'confirmed',
        businessDecisionReason: 'Interview completed successfully.',
        businessDecidedAt: new Date().toISOString(),
      },
      {
        status: 'interview',
        interviewSchedule: text(row?.scheduleRecord?.scheduledAt || row?.scheduleRecord?.scheduled_at),
        interviewDate: text(row?.scheduleRecord?.scheduledAt || row?.scheduleRecord?.scheduled_at),
        interviewer: row.interviewer,
        interviewMode: row.modeLabel,
        interviewLocationOrLink: row.locationLabel,
        interviewNotes: row.notesLabel,
        interviewResponseStatus: 'confirmed',
        interviewDecisionReason: 'Interview completed successfully.',
        interviewDecidedAt: new Date().toISOString(),
      },
    )
    formMessage.value = `Interview marked complete for ${row.applicantName}. The applicant timeline will now show the interview step as complete.`
  } catch (error) {
    formError.value = error instanceof Error ? error.message : 'Unable to complete this interview right now.'
  } finally {
    activeStatusActionId.value = ''
  }
}

const markInterviewFailed = async () => {
  const row = selectedInterviewRow.value
  const failureReason = text(decisionForm.failureReason)
  if (!row?.scheduleRecord?.id || activeStatusActionId.value) return
  if (!failureReason) {
    formError.value = 'Enter a failure reason before marking the interview as failed.'
    return
  }

  try {
    activeStatusActionId.value = row.id
    formError.value = ''
    await syncInterviewRecordAndApplication(
      row,
      {
        scheduleStatus: 'cancelled',
        applicantResponseStatus: 'reschedule_rejected',
        businessDecisionReason: failureReason,
        businessDecidedAt: new Date().toISOString(),
        availableScheduleOptions: [],
      },
      {
        status: 'rejected',
        rejectionReason: failureReason,
        interviewSchedule: text(row?.scheduleRecord?.scheduledAt || row?.scheduleRecord?.scheduled_at),
        interviewDate: text(row?.scheduleRecord?.scheduledAt || row?.scheduleRecord?.scheduled_at),
        interviewer: row.interviewer,
        interviewMode: row.modeLabel,
        interviewLocationOrLink: row.locationLabel,
        interviewNotes: row.notesLabel,
        interviewResponseStatus: 'reschedule_rejected',
        interviewDecisionReason: failureReason,
        interviewDecidedAt: new Date().toISOString(),
        interviewScheduleOptions: [],
      },
    )
    formMessage.value = `Interview marked failed for ${row.applicantName}. The rest of the applicant timeline will stop and turn red in real time.`
    closeDecisionModal(true)
  } catch (error) {
    formError.value = error instanceof Error ? error.message : 'Unable to mark this interview as failed right now.'
  } finally {
    activeStatusActionId.value = ''
  }
}

const createBusinessInterviewSchedule = async () => {
  const targetApplicant = selectedApplicant.value
  const scheduledAt = toIsoString(form.schedule)

  if (!text(props.workspaceOwnerId)) {
    formError.value = 'Missing workspace owner ID for live interview sync.'
    return
  }
  if (!targetApplicant?.applicationId || !targetApplicant?.applicantId || !targetApplicant?.jobId) {
    formError.value = 'Select an approved applicant before creating the interview schedule.'
    return
  }
  if (!scheduledAt) {
    formError.value = 'Choose a valid schedule before creating the interview.'
    return
  }

  formError.value = ''
  formMessage.value = ''

  try {
    isSavingSchedule.value = true
    const payload = {
      id: text(targetApplicant?.scheduleRecord?.id),
      workspaceOwnerId: text(props.workspaceOwnerId),
      workspaceOwnerName: targetApplicant.workspaceOwnerName,
      workspaceOwnerEmail: targetApplicant.workspaceOwnerEmail,
      applicationId: targetApplicant.applicationId,
      applicantId: targetApplicant.applicantId,
      applicantName: targetApplicant.applicantName,
      applicantEmail: targetApplicant.applicantEmail,
      applicantAvatar: targetApplicant.applicantAvatar,
      jobId: targetApplicant.jobId,
      jobTitle: targetApplicant.jobTitle,
      interviewType: 'interview',
      scheduledAt,
      mode: text(form.mode) || 'Online',
      locationOrLink: text(form.locationOrLink),
      interviewer: text(form.interviewer) || 'Hiring Manager',
      notes: text(form.notes),
      scheduleStatus: 'scheduled',
      applicantResponseStatus: 'pending',
      applicantResponseReason: '',
      requestedScheduleAt: '',
      applicantRespondedAt: '',
      businessDecisionReason: '',
      businessDecidedAt: '',
      availableScheduleOptions: [scheduledAt],
    }

    const savedInterview = await saveBusinessInterviewSchedule(payload)
    await updateApplicantJobApplicationStatus(targetApplicant.applicationId, {
      status: 'interview',
      interviewSchedule: scheduledAt,
      interviewDate: scheduledAt,
      interviewType: 'interview',
      interviewer: payload.interviewer,
      interviewMode: payload.mode,
      interviewLocationOrLink: payload.locationOrLink,
      interviewNotes: payload.notes,
      interviewResponseStatus: 'pending',
      interviewRescheduleReason: '',
      interviewRequestedScheduleAt: '',
      interviewDecisionReason: '',
      interviewRespondedAt: '',
      interviewDecidedAt: '',
      interviewScheduleOptions: [scheduledAt],
    })

    upsertLiveInterview(savedInterview)
    formMessage.value = `Interview saved for ${targetApplicant.applicantName}. The applicant interview page and status timeline now sync from Firebase.`
    resetForm()
  } catch (error) {
    formError.value = error instanceof Error ? error.message : 'Unable to create the interview right now.'
  } finally {
    isSavingSchedule.value = false
  }
}

watch(() => props.workspaceOwnerId, () => {
  stopInterviewSchedulesSubscription?.()
  if (!text(props.workspaceOwnerId)) {
    liveInterviewSchedules.value = []
    return
  }
  isSyncingSchedules.value = true
  stopInterviewSchedulesSubscription = subscribeToBusinessInterviewSchedules(
    props.workspaceOwnerId,
    (records = []) => {
      liveInterviewSchedules.value = Array.isArray(records) ? records : []
      isSyncingSchedules.value = false
    },
    () => {
      liveInterviewSchedules.value = []
      isSyncingSchedules.value = false
    },
  )
}, { immediate: true })

watch(interviewRows, (rows = []) => {
  if (!rows.length) {
    form.applicantId = ''
    return
  }
  if (!rows.some((item) => item.applicationId === form.applicantId)) form.applicantId = text(rows[0]?.applicationId)
}, { immediate: true })

onBeforeUnmount(() => {
  stopInterviewSchedulesSubscription?.()
})
</script>

<template>
  <section class="business-interview-scheduling">
    <header class="business-interview-scheduling__header">
      <div class="business-interview-scheduling__header-copy">
        <p class="business-interview-scheduling__kicker">Interview Management</p>
        <h2>Schedule live interviews and sync them to applicants</h2>
        <p>Interview schedules here save to Firebase, appear on the applicant interview page, and light up the applicant status timeline in real time.</p>
      </div>
      <div class="business-interview-scheduling__header-actions">
        <span class="business-interview-scheduling__sync-badge">{{ isSyncingSchedules ? 'Syncing live data...' : 'Live sync active' }}</span>
      </div>
    </header>

    <div class="business-interview-scheduling__stat-strip">
      <span>{{ stats.ready }} ready</span>
      <span>{{ stats.scheduled }} scheduled</span>
      <span>{{ stats.completed }} completed</span>
    </div>

    <div v-if="showSchedulingView" class="business-interview-scheduling__layout">
      <section class="business-interview-scheduling__main">
        <article class="business-interview-scheduling__card">
          <div class="business-interview-scheduling__card-head">
            <div>
              <p class="business-interview-scheduling__card-kicker">Scheduling Form</p>
              <h3>Create interview schedule</h3>
              <p>One interview only. No more initial and final split.</p>
            </div>
          </div>

          <div class="business-interview-scheduling__form-grid">
            <label class="business-interview-scheduling__field business-interview-scheduling__field--full"><span>Applicant</span><select v-model="form.applicantId"><option value="">Select applicant</option><option v-for="option in applicantOptions" :key="option.value" :value="option.value">{{ option.label }}</option></select></label>
            <label class="business-interview-scheduling__field"><span>Interview</span><input type="text" value="Interview" disabled></label>
            <label class="business-interview-scheduling__field"><span>Schedule</span><input v-model="form.schedule" type="datetime-local"></label>
            <label class="business-interview-scheduling__field"><span>Mode</span><select v-model="form.mode"><option>Online</option><option>On-site</option><option>Hybrid</option></select></label>
            <label class="business-interview-scheduling__field"><span>Interviewer</span><input v-model="form.interviewer" type="text"></label>
            <label class="business-interview-scheduling__field business-interview-scheduling__field--full"><span>Location or Meeting Link</span><input v-model="form.locationOrLink" type="text" placeholder="Office meeting room or video call link"></label>
            <label class="business-interview-scheduling__field business-interview-scheduling__field--full"><span>Notes</span><textarea v-model="form.notes" rows="4" placeholder="Add interview notes or reminders"></textarea></label>
          </div>

          <p v-if="selectedApplicant" class="business-interview-scheduling__info">Current applicant: {{ selectedApplicant.applicantName }} | {{ selectedApplicant.jobTitle }} | {{ selectedApplicant.scheduledAt ? selectedApplicant.scheduledAtLabel : 'No interview saved yet' }}</p>
          <p v-if="formError" class="business-interview-scheduling__error">{{ formError }}</p>
          <p v-else-if="formMessage" class="business-interview-scheduling__info business-interview-scheduling__info--success">{{ formMessage }}</p>
          <div class="business-interview-scheduling__actions">
            <button type="button" class="business-interview-scheduling__reset" @click="resetForm">Reset</button>
            <button type="button" class="business-interview-scheduling__submit" :disabled="isSavingSchedule" @click="createBusinessInterviewSchedule">{{ isSavingSchedule ? 'Saving live schedule...' : 'Create Schedule' }}</button>
          </div>
        </article>

        <article class="business-interview-scheduling__card">
          <div class="business-interview-scheduling__calendar-head">
            <h3>Interview Calendar</h3>
            <div class="business-interview-scheduling__calendar-nav">
              <button type="button" @click="currentDate = new Date(currentDate.getFullYear(), currentDate.getMonth() - 1, 1)">&lt;</button>
              <strong>{{ monthLabel }}</strong>
              <button type="button" @click="currentDate = new Date(currentDate.getFullYear(), currentDate.getMonth() + 1, 1)">&gt;</button>
            </div>
          </div>
          <div class="business-interview-scheduling__calendar-grid business-interview-scheduling__calendar-grid--week"><span v-for="label in weekdayLabels" :key="label">{{ label }}</span></div>
          <div class="business-interview-scheduling__calendar-grid">
            <button v-for="day in calendarDays" :key="day.key" type="button" class="business-interview-scheduling__day-cell" :class="{ 'is-muted': day.muted, 'is-today': day.isToday, 'is-selected': day.isSelected, 'is-busy': day.schedules }" :disabled="day.muted" @click="selectedDay = day.label"><span class="business-interview-scheduling__day-number">{{ day.label }}</span><small v-if="day.schedules">{{ day.schedules }} schedule{{ day.schedules === 1 ? '' : 's' }}</small></button>
          </div>
          <div class="business-interview-scheduling__day-list">
            <h4>Selected Day Schedules</h4>
            <ul v-if="selectedDaySchedules.length"><li v-for="row in selectedDaySchedules" :key="row.id"><strong>{{ row.applicantName }}</strong><span>{{ row.jobTitle }}</span><small>{{ row.scheduledAtLabel }}</small></li></ul>
            <p v-else class="business-interview-scheduling__day-empty">No live interview schedules for this day yet.</p>
          </div>
        </article>
      </section>

      <aside class="business-interview-status">
        <article v-for="row in interviewRows" :key="row.id" class="business-interview-scheduling__card business-interview-status__panel">
          <div class="business-interview-status__topline">
            <div class="business-interview-status__applicant">
              <strong>{{ row.applicantName }}</strong>
              <span>{{ row.applicantEmail }}</span>
              <small>{{ row.jobTitle }}</small>
            </div>
            <span class="business-interview-status__badge" :class="resolveStatusBadgeClass(row.interviewStatus)">{{ row.interviewStatus }}</span>
          </div>

          <div class="business-interview-status__body">
            <div class="business-interview-status__stage">
              <strong>Applicant Status</strong>
              <span>{{ row.applicantStatusLabel }}</span>
              <small>{{ row.applicantStatusMeta }}</small>
            </div>

            <div class="business-interview-status__grid">
              <div class="business-interview-status__detail">
                <span>Interviewer</span>
                <strong>{{ row.interviewer }}</strong>
              </div>
              <div class="business-interview-status__detail">
                <span>Schedule</span>
                <strong>{{ row.scheduledAtLabel }}</strong>
              </div>
              <div class="business-interview-status__detail">
                <span>Mode</span>
                <strong>{{ row.modeLabel }}</strong>
              </div>
              <div class="business-interview-status__detail">
                <span>Live Sync</span>
                <strong>{{ row.updatedAtLabel }}</strong>
              </div>
            </div>

            <div class="business-interview-status__meta">
              <div class="business-interview-status__meta-block">
                <span>Location or Link</span>
                <strong>{{ row.locationLabel }}</strong>
              </div>
              <div class="business-interview-status__meta-block">
                <span>Notes</span>
                <strong>{{ row.notesLabel }}</strong>
              </div>
            </div>
          </div>
        </article>
        <article v-if="!interviewRows.length" class="business-interview-scheduling__card business-interview-status__panel"><div class="business-interview-status__applicant"><strong>No live interview queue yet</strong><span>Approved applicants will appear here once they are ready for interview scheduling.</span></div></article>
      </aside>
    </div>

    <section v-else-if="showStatusView" class="business-interview-status-view">
      <article class="business-interview-scheduling__card">
        <div class="business-interview-scheduling__card-head">
          <div>
            <p class="business-interview-scheduling__card-kicker">Interview Status</p>
            <h3>Live applicant interview table</h3>
            <p>Review sent schedules, reschedule requests, applicant confirmations, and final interview outcomes in real time.</p>
          </div>
        </div>

        <div class="business-interview-status-table__wrap">
          <table class="business-interview-status-table">
            <thead>
              <tr>
                <th>Applicant</th>
                <th>Applied Role</th>
                <th>Schedule</th>
                <th>Status</th>
                <th>Action</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in interviewStatusRows" :key="row.id">
                <td>
                  <div class="business-interview-status-table__identity">
                    <strong>{{ row.applicantName }}</strong>
                    <span>{{ row.applicantEmail }}</span>
                  </div>
                </td>
                <td>
                  <div class="business-interview-status-table__identity">
                    <strong>{{ row.jobTitle }}</strong>
                    <span>{{ row.interviewer }}</span>
                  </div>
                </td>
                <td>
                  <div class="business-interview-status-table__identity">
                    <strong>{{ row.scheduledAtLabel }}</strong>
                    <span>{{ row.modeLabel }}</span>
                  </div>
                </td>
                <td>
                  <span class="business-interview-status__badge" :class="row.workflowClass">{{ row.workflowStatus }}</span>
                </td>
                <td>
                  <div class="business-interview-status-table__actions">
                    <button
                      type="button"
                      class="business-interview-status-table__button"
                      :disabled="activeStatusActionId === row.id"
                      @click="openDecisionModal(row, row.canReviewReschedule ? 'review' : row.canMarkOutcome ? 'outcome' : 'review')"
                    >
                      {{ row.actionLabel }}
                    </button>
                    <button
                      v-if="row.canMarkOutcome"
                      type="button"
                      class="business-interview-status-table__button business-interview-status-table__button--success"
                      :disabled="activeStatusActionId === row.id"
                      @click="markInterviewCompleted(row)"
                    >
                      {{ activeStatusActionId === row.id ? 'Saving...' : 'Complete' }}
                    </button>
                    <button
                      v-if="row.canMarkOutcome"
                      type="button"
                      class="business-interview-status-table__button business-interview-status-table__button--danger"
                      :disabled="activeStatusActionId === row.id"
                      @click="openDecisionModal(row, 'failed')"
                    >
                      Failed
                    </button>
                  </div>
                </td>
              </tr>
              <tr v-if="!interviewStatusRows.length">
                <td colspan="5">
                  <div class="business-interview-status-table__empty">
                    No live interview status rows yet. Create an interview schedule first.
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </article>
    </section>

    <div v-if="isDecisionModalOpen && selectedInterviewRow" class="business-interview-modal" @click.self="closeDecisionModal()">
      <article class="business-interview-modal__card">
        <header class="business-interview-modal__header">
          <div>
            <p class="business-interview-scheduling__card-kicker">Interview Status Action</p>
            <h3 v-if="decisionModalMode === 'review'">Review Reschedule Request</h3>
            <h3 v-else-if="decisionModalMode === 'outcome'">Interview Details</h3>
            <h3 v-else>Mark Interview Failed</h3>
          </div>
          <button type="button" class="business-interview-modal__close" :disabled="activeStatusActionId === selectedInterviewRow.id" @click="closeDecisionModal()">Close</button>
        </header>

        <div class="business-interview-modal__body">
          <div class="business-interview-status__stage">
            <strong>{{ selectedInterviewRow.applicantName }}</strong>
            <span>{{ selectedInterviewRow.applicantStatusLabel }}</span>
            <small>{{ selectedInterviewRow.applicantStatusMeta }}</small>
          </div>

          <div class="business-interview-status__grid">
            <div class="business-interview-status__detail"><span>Schedule</span><strong>{{ selectedInterviewRow.scheduledAtLabel }}</strong></div>
            <div class="business-interview-status__detail"><span>Status</span><strong>{{ selectedInterviewRow.workflowStatus }}</strong></div>
            <div class="business-interview-status__detail"><span>Mode</span><strong>{{ selectedInterviewRow.modeLabel }}</strong></div>
            <div class="business-interview-status__detail"><span>Updated</span><strong>{{ selectedInterviewRow.updatedAtLabel }}</strong></div>
          </div>

          <div v-if="decisionModalMode === 'review'" class="business-interview-modal__section">
            <label class="business-interview-scheduling__field business-interview-scheduling__field--full">
              <span>Applicant Reason</span>
              <textarea :value="selectedInterviewRow.scheduleRecord?.applicantResponseReason || selectedInterviewRow.scheduleRecord?.applicant_response_reason || 'No reason submitted.'" rows="4" disabled />
            </label>

            <div class="business-interview-modal__date-list">
              <label v-for="(dateValue, index) in decisionForm.availableDates" :key="`available-date-${index}`" class="business-interview-scheduling__field business-interview-scheduling__field--full">
                <span>Available Date {{ index + 1 }}</span>
                <div class="business-interview-modal__date-row">
                  <input :value="dateValue" type="datetime-local" @input="updateAvailableDateOption(index, $event.target.value)">
                  <button type="button" class="business-interview-status-table__button" @click="removeAvailableDateOption(index)">Remove</button>
                </div>
              </label>
              <button type="button" class="business-interview-status-table__button" @click="addAvailableDateOption()">Add Date</button>
            </div>

            <label class="business-interview-scheduling__field business-interview-scheduling__field--full">
              <span>Reject Reason</span>
              <textarea v-model.trim="decisionForm.rejectionReason" rows="3" placeholder="Required only if you reject the reschedule request." />
            </label>

            <div class="business-interview-status-table__actions">
              <button type="button" class="business-interview-status-table__button business-interview-status-table__button--success" :disabled="activeStatusActionId === selectedInterviewRow.id" @click="approveRescheduleRequest()">
                {{ activeStatusActionId === selectedInterviewRow.id ? 'Saving...' : 'Approve and Send Dates' }}
              </button>
              <button type="button" class="business-interview-status-table__button business-interview-status-table__button--danger" :disabled="activeStatusActionId === selectedInterviewRow.id" @click="rejectRescheduleRequest()">
                Reject Request
              </button>
            </div>
          </div>

          <div v-else-if="decisionModalMode === 'outcome'" class="business-interview-modal__section">
            <div class="business-interview-status__meta">
              <div class="business-interview-status__meta-block"><span>Location or Link</span><strong>{{ selectedInterviewRow.locationLabel }}</strong></div>
              <div class="business-interview-status__meta-block"><span>Notes</span><strong>{{ selectedInterviewRow.notesLabel }}</strong></div>
            </div>
          </div>

          <div v-else class="business-interview-modal__section">
            <label class="business-interview-scheduling__field business-interview-scheduling__field--full">
              <span>Failure Reason</span>
              <textarea v-model.trim="decisionForm.failureReason" rows="4" placeholder="Explain why the applicant failed the interview." />
            </label>
            <div class="business-interview-status-table__actions">
              <button type="button" class="business-interview-status-table__button business-interview-status-table__button--danger" :disabled="activeStatusActionId === selectedInterviewRow.id" @click="markInterviewFailed()">
                {{ activeStatusActionId === selectedInterviewRow.id ? 'Saving...' : 'Confirm Failed' }}
              </button>
            </div>
          </div>
        </div>
      </article>
    </div>
  </section>
</template>

<style scoped>
.business-interview-scheduling__sync-badge,.business-interview-scheduling__info{display:inline-flex;align-items:center;padding:.75rem 1rem;border:1px solid rgba(126,155,140,.24);background:rgba(245,249,246,.96);color:#325240;font-weight:600}
.business-interview-scheduling__info{display:block}
.business-interview-scheduling__info--success{background:rgba(234,247,238,.98);color:#276746}
.business-interview-scheduling__error{margin:0;color:#a13f3f;font-weight:700}
.business-interview-status__panel{display:grid;gap:1rem}
.business-interview-status__topline{display:flex;justify-content:space-between;gap:1rem;align-items:flex-start}
.business-interview-status__body{display:grid;gap:.9rem}
.business-interview-status__grid{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:.75rem}
.business-interview-status__detail,.business-interview-status__meta-block{display:grid;gap:.2rem;padding:.8rem .9rem;border:1px solid rgba(126,155,140,.16);background:rgba(248,251,249,.96)}
.business-interview-status__detail span,.business-interview-status__meta-block span{font-size:.75rem;font-weight:700;letter-spacing:.08em;text-transform:uppercase;color:#6a8a79}
.business-interview-status__detail strong,.business-interview-status__meta-block strong{color:#274234;line-height:1.45;overflow-wrap:anywhere}
.business-interview-status__stage{display:grid;gap:.2rem;padding:.95rem 1rem;border:1px solid rgba(126,155,140,.18);background:linear-gradient(180deg,rgba(245,249,246,.98),rgba(238,245,241,.96))}
.business-interview-status__stage strong{font-size:.82rem;letter-spacing:.08em;text-transform:uppercase;color:#698574}
.business-interview-status__stage span{font-size:1rem;font-weight:700;color:#233b2d}
.business-interview-status__stage small{color:#5f786a;line-height:1.5}
.business-interview-status__meta{display:grid;gap:.75rem}
.business-interview-status__badge.is-completed{background:rgba(223,242,230,.98);color:#2f7550}
.business-interview-status__badge.is-cancelled{background:rgba(248,229,229,.98);color:#9b4040}
.business-interview-status__badge.is-scheduled{background:rgba(230,239,251,.95);color:#42648d}
.business-interview-status__badge.is-pending,.business-interview-status__badge.is-ready{background:rgba(255,242,215,.95);color:#ad760f}
.business-interview-status-view{display:grid;gap:1.5rem}
.business-interview-status-table__wrap{overflow:auto;border:1px solid rgba(126,155,140,.18);background:rgba(250,252,250,.95)}
.business-interview-status-table{width:100%;border-collapse:collapse;min-width:720px}
.business-interview-status-table thead th{font-size:.7rem;letter-spacing:.12em;text-transform:uppercase;color:#6a8a79;text-align:left;padding:1rem;border-bottom:1px solid rgba(126,155,140,.2);background:rgba(245,249,246,.98)}
.business-interview-status-table tbody td{padding:1rem;border-bottom:1px solid rgba(126,155,140,.12);vertical-align:top;color:#243b2f}
.business-interview-status-table__identity{display:grid;gap:.35rem}
.business-interview-status-table__identity strong{font-size:.98rem;color:#233b2d}
.business-interview-status-table__identity span{font-size:.82rem;color:#5f786a}
.business-interview-status-table__actions{display:flex;flex-wrap:wrap;gap:.5rem}
.business-interview-status-table__button{border:1px solid rgba(126,155,140,.35);background:rgba(245,249,246,.98);color:#2f4b3c;font-weight:600;padding:.45rem .85rem;cursor:pointer}
.business-interview-status-table__button:disabled{opacity:.6;cursor:not-allowed}
.business-interview-status-table__button--success{background:rgba(223,242,230,.98);border-color:rgba(120,177,146,.6);color:#2c6a49}
.business-interview-status-table__button--danger{background:rgba(248,229,229,.98);border-color:rgba(191,110,110,.55);color:#8c3232}
.business-interview-status-table__empty{padding:1.5rem;background:rgba(245,249,246,.98);border:1px dashed rgba(126,155,140,.3);text-align:center;color:#5f786a}
.business-interview-modal{position:fixed;inset:0;background:rgba(22,32,27,.5);display:flex;align-items:center;justify-content:center;padding:1.5rem;z-index:50}
.business-interview-modal__card{width:min(760px,100%);background:#fff;border:1px solid rgba(126,155,140,.25);box-shadow:0 18px 60px rgba(21,34,27,.2)}
.business-interview-modal__header{display:flex;justify-content:space-between;gap:1rem;align-items:flex-start;padding:1.25rem 1.5rem;border-bottom:1px solid rgba(126,155,140,.2);background:rgba(245,249,246,.98)}
.business-interview-modal__close{border:1px solid rgba(126,155,140,.35);background:rgba(250,252,250,.98);padding:.4rem .75rem;font-weight:600;color:#2f4b3c;cursor:pointer}
.business-interview-modal__body{display:grid;gap:1.25rem;padding:1.5rem}
.business-interview-modal__section{display:grid;gap:1rem}
.business-interview-modal__date-list{display:grid;gap:.75rem}
.business-interview-modal__date-row{display:flex;gap:.75rem;align-items:center}
.business-interview-modal__date-row input{flex:1}
@media (max-width:900px){.business-interview-status-table{min-width:0}.business-interview-status-table thead{display:none}.business-interview-status-table tbody tr{display:grid;gap:.75rem;padding:1rem}.business-interview-status-table tbody td{border-bottom:none;padding:0}.business-interview-status-table__actions{justify-content:flex-start}}
@media (max-width:900px){.business-interview-status__topline{display:grid}.business-interview-status__grid{grid-template-columns:minmax(0,1fr)}}
</style>
