<script setup>
import { computed, reactive, ref } from 'vue'
import BusinessIssueOfferContent from '@/modules/Employer/Business/business_IssueOffer.vue'
import BusinessIssueOfferModal from '@/modules/Employer/Business/business_issue_offer.vue'
import { saveBusinessJobOfferRecord } from '@/lib/job_offers'
import '@/components/businesss.css'

const props = defineProps({
  businessJobApplications: {
    type: Array,
    default: () => [],
  },
  isEmployeeWorkspaceMode: {
    type: Boolean,
    default: false,
  },
})

const text = (value = '') => String(value ?? '').trim()
const normalizeValue = (value = '') => text(value).toLowerCase()
const buildInitials = (value = '') =>
  text(value)
    .split(/\s+/)
    .filter(Boolean)
    .slice(0, 2)
    .map((part) => part.charAt(0).toUpperCase())
    .join('') || 'AP'

const canEditBusinessModule = () => !props.isEmployeeWorkspaceMode

const fallbackRows = [
  {
    id: 'offer-1',
    applicantName: 'Maria Santos',
    applicantEmail: 'maria.santos@example.com',
    jobTitle: 'Cashier',
    status: 'approved',
    approvedAt: new Date().toISOString(),
    interviewer: 'Recruitment Team',
    offerStatus: 'not_sent',
  },
  {
    id: 'offer-2',
    applicantName: 'John Dela Cruz',
    applicantEmail: 'john.delacruz@example.com',
    jobTitle: 'Sales Associate',
    status: 'interview',
    interviewDecidedAt: new Date().toISOString(),
    interviewer: 'Hiring Manager',
    offerStatus: 'sent',
    jobOfferTitle: 'Sales Associate Job Offer',
    jobOfferCompensation: 'PHP 18,000 monthly',
    jobOfferResponseDeadline: new Date(Date.now() + (5 * 24 * 60 * 60 * 1000)).toISOString().slice(0, 10),
    jobOfferLetter: 'We are pleased to offer you the Sales Associate role. Please review the compensation package and confirm your availability.',
  },
]

const issueOfferRows = computed(() => {
  const source = Array.isArray(props.businessJobApplications) && props.businessJobApplications.length
    ? props.businessJobApplications
    : fallbackRows

  return source
    .filter((item) => {
      const status = normalizeValue(item?.status)
      const offerStatus = normalizeValue(item?.jobOfferStatus || item?.offerStatus)
      return (
        status.includes('approve')
        || status.includes('interview')
        || ['sent', 'accepted', 'rejected', 'declined'].includes(offerStatus)
      )
    })
    .map((item, index) => {
      const applicantName = text(item?.applicantName || item?.applicant_name || item?.name) || `Applicant ${index + 1}`
      const rawOfferStatus = normalizeValue(item?.jobOfferStatus || item?.offerStatus)
      let offerStatus = 'not_sent'
      if (rawOfferStatus === 'accepted') offerStatus = 'accepted'
      else if (rawOfferStatus === 'rejected' || rawOfferStatus === 'declined') offerStatus = 'rejected'
      else if (rawOfferStatus === 'sent') offerStatus = 'sent'

      const passedStageLabel = normalizeValue(item?.status).includes('interview') ? 'Interview Completed' : 'Approved for Offer'
      const completedAt = (
        item?.interviewDecidedAt
        || item?.interviewRespondedAt
        || item?.approvedAt
        || item?.reviewedAt
        || item?.statusUpdatedAt
        || item?.updatedAt
        || item?.appliedAt
        || null
      )

      const offerStatusLabelMap = {
        not_sent: 'Ready to send',
        sent: 'Offer sent',
        accepted: 'Accepted',
        rejected: 'Declined',
      }
      const offerToneMap = {
        not_sent: 'muted',
        sent: 'warning',
        accepted: 'success',
        rejected: 'danger',
      }

      return {
        id: text(item?.id || item?.applicationId || `${applicantName}-${index}`),
        applicationId: text(item?.id || item?.applicationId || item?.application_id),
        workspaceOwnerId: text(item?.workspaceOwnerId || item?.workspace_owner_id),
        workspaceOwnerName: text(item?.workspaceOwnerName || item?.workspace_owner_name || item?.businessName || item?.business_name),
        workspaceOwnerEmail: text(item?.workspaceOwnerEmail || item?.workspace_owner_email),
        applicantId: text(item?.applicantId || item?.applicant_id || item?.userId || item?.user_id),
        applicantName,
        applicantEmail: text(item?.applicantEmail || item?.applicant_email || item?.email) || 'No email',
        applicantAvatar: text(item?.applicantAvatar || item?.avatarUrl || item?.avatar),
        applicantInitials: buildInitials(applicantName),
        jobId: text(item?.jobId || item?.job_id),
        jobTitle: text(item?.jobTitle || item?.job_title || item?.role || item?.position) || 'Open role',
        passedStageLabel,
        completedAt,
        interviewer: text(item?.interviewer || item?.reviewedByName || item?.reviewedBy) || 'Business team',
        jobOfferId: text(item?.jobOfferId || item?.job_offer_id || item?.offerId || item?.offer_id),
        interviewType: text(item?.jobOfferInterviewType || item?.job_offer_interview_type || item?.interviewType || item?.interview_type || 'initial') || 'initial',
        sentAt: text(item?.jobOfferSentAt || item?.job_offer_sent_at),
        createdAt: text(item?.jobOfferCreatedAt || item?.job_offer_created_at),
        offerTitle: text(item?.jobOfferTitle || item?.offerTitle),
        offerStatus,
        offerStatusLabel: offerStatusLabelMap[offerStatus] || 'Ready to send',
        offerTone: offerToneMap[offerStatus] || 'muted',
        offerMeta: offerStatus === 'not_sent'
          ? 'Draft the offer details for this applicant.'
          : offerStatus === 'sent'
            ? 'Waiting for applicant response.'
            : offerStatus === 'accepted'
              ? 'Ready to move to Contract Signing.'
              : 'Offer was declined by the applicant.',
        actionLabel: offerStatus === 'not_sent' ? 'Draft Offer' : 'Open Offer',
        existingOffer: {
          offerTitle: text(item?.jobOfferTitle || item?.offerTitle),
          compensation: text(item?.jobOfferCompensation || item?.offerCompensation),
          startDate: text(item?.jobOfferStartDate || item?.offerStartDate),
          responseDeadline: text(item?.jobOfferResponseDeadline || item?.offerResponseDeadline),
          offerLetter: text(item?.jobOfferLetter || item?.offerLetter),
        },
      }
    })
})

const selectedIssueOfferId = ref('')
const isIssueOfferSubmitting = ref(false)
const issueOfferFormError = ref('')
const issueOfferForm = reactive({
  offerTitle: '',
  compensation: '',
  startDate: '',
  responseDeadline: '',
  offerLetter: '',
})

const selectedIssueOfferCandidate = computed(() =>
  issueOfferRows.value.find((row) => row.id === selectedIssueOfferId.value) || null,
)
const isIssueOfferOpen = computed(() => Boolean(selectedIssueOfferCandidate.value))
const issueOfferSubmitLabel = computed(() =>
  selectedIssueOfferCandidate.value?.offerStatus === 'not_sent' ? 'Send Offer' : 'Update Offer',
)
const issueOfferMinDate = computed(() => new Date().toISOString().slice(0, 10))

const syncIssueOfferForm = (candidate = null) => {
  issueOfferForm.offerTitle = text(candidate?.existingOffer?.offerTitle || candidate?.offerTitle || `${candidate?.jobTitle || 'Job'} Offer`)
  issueOfferForm.compensation = text(candidate?.existingOffer?.compensation || 'PHP ')
  issueOfferForm.startDate = text(candidate?.existingOffer?.startDate)
  issueOfferForm.responseDeadline = text(candidate?.existingOffer?.responseDeadline)
  issueOfferForm.offerLetter = text(
    candidate?.existingOffer?.offerLetter
      || `Hello ${candidate?.applicantName || 'Applicant'},\n\nWe are pleased to offer you the ${candidate?.jobTitle || 'selected'} position. Please review the details and respond on or before the stated deadline.`,
  )
}

const openIssueOfferModal = (rowId = '') => {
  const normalizedId = text(rowId)
  if (!normalizedId) return

  selectedIssueOfferId.value = normalizedId
  issueOfferFormError.value = ''
  syncIssueOfferForm(issueOfferRows.value.find((row) => row.id === normalizedId) || null)
}

const closeIssueOfferModal = () => {
  selectedIssueOfferId.value = ''
  issueOfferFormError.value = ''
  isIssueOfferSubmitting.value = false
}

const submitIssueOffer = async () => {
  const candidate = selectedIssueOfferCandidate.value
  if (!candidate) return
  if (!text(issueOfferForm.offerTitle) || !text(issueOfferForm.compensation) || !text(issueOfferForm.offerLetter)) {
    issueOfferFormError.value = 'Complete the offer title, compensation, and offer letter before sending.'
    return
  }
  if (!candidate.applicationId || !candidate.workspaceOwnerId || !candidate.applicantId || !candidate.jobId) {
    issueOfferFormError.value = 'This applicant record is still syncing. Please reopen the offer and try again.'
    return
  }

  issueOfferFormError.value = ''
  isIssueOfferSubmitting.value = true

  try {
    await saveBusinessJobOfferRecord({
      id: candidate.jobOfferId || candidate.applicationId,
      applicationId: candidate.applicationId,
      workspaceOwnerId: candidate.workspaceOwnerId,
      workspaceOwnerName: candidate.workspaceOwnerName,
      workspaceOwnerEmail: candidate.workspaceOwnerEmail,
      applicantId: candidate.applicantId,
      applicantName: candidate.applicantName,
      applicantEmail: candidate.applicantEmail,
      applicantAvatar: candidate.applicantAvatar,
      jobId: candidate.jobId,
      jobTitle: candidate.jobTitle,
      interviewType: candidate.interviewType || 'initial',
      offerTitle: text(issueOfferForm.offerTitle),
      compensation: text(issueOfferForm.compensation),
      startDate: text(issueOfferForm.startDate),
      responseDeadline: text(issueOfferForm.responseDeadline),
      offerLetter: text(issueOfferForm.offerLetter),
      offerStatus: 'sent',
      sentAt: candidate.sentAt,
      createdAt: candidate.createdAt,
    })

    closeIssueOfferModal()
  } catch (error) {
    issueOfferFormError.value = error instanceof Error ? error.message : 'Unable to send this job offer right now.'
  } finally {
    isIssueOfferSubmitting.value = false
  }
}
</script>

<template>
  <section class="business-shell__stage">
    <BusinessIssueOfferContent
      :rows="issueOfferRows"
      :can-edit-business-module="canEditBusinessModule"
      :open-issue-offer-modal="openIssueOfferModal"
    />

    <BusinessIssueOfferModal
      :is-open="isIssueOfferOpen"
      :candidate="selectedIssueOfferCandidate"
      :is-submitting="isIssueOfferSubmitting"
      :form="issueOfferForm"
      :form-error="issueOfferFormError"
      :submit-label="issueOfferSubmitLabel"
      :min-date="issueOfferMinDate"
      @close="closeIssueOfferModal"
      @submit="submitIssueOffer"
      @update:form-title="issueOfferForm.offerTitle = $event"
      @update:form-compensation="issueOfferForm.compensation = $event"
      @update:form-start-date="issueOfferForm.startDate = $event"
      @update:form-response-deadline="issueOfferForm.responseDeadline = $event"
      @update:form-offer-letter="issueOfferForm.offerLetter = $event"
    />
  </section>
</template>
