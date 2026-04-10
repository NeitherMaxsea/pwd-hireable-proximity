<script setup>
import { computed, onBeforeUnmount, ref, watch } from 'vue'
import BusinessContractSigningContent from '@/modules/Employer/Business/business_ContractSigning.vue'
import { sendBusinessContractFile, subscribeToBusinessContracts } from '@/lib/contract_signing'

const props = defineProps({
  businessJobApplications: {
    type: Array,
    default: () => [],
  },
  businessName: {
    type: String,
    default: 'Business Workspace',
  },
  workspaceOwnerId: {
    type: String,
    default: '',
  },
  workspaceOwnerEmail: {
    type: String,
    default: '',
  },
  isEmployeeWorkspaceMode: {
    type: Boolean,
    default: false,
  },
})

const text = (value = '') => String(value ?? '').trim()
const normalizeValue = (value = '') => text(value).toLowerCase()
const nowIso = () => new Date().toISOString()
const buildInitials = (value = '') =>
  text(value)
    .split(/\s+/)
    .filter(Boolean)
    .slice(0, 2)
    .map((part) => part.charAt(0).toUpperCase())
    .join('') || 'AP'

const isAcceptedOfferStatus = (value = '') =>
  ['accepted', 'confirmed', 'signed'].includes(normalizeValue(value))

const normalizeBusinessContractStatus = (value = '') =>
  text(value).toLowerCase() || 'draft'

const getBusinessContractStatusLabel = (value = '') => {
  const normalizedStatus = normalizeBusinessContractStatus(value)
  if (normalizedStatus === 'completed') return 'Completed'
  if (normalizedStatus === 'applicant_signed') return 'Returned by Applicant'
  if (normalizedStatus === 'sent') return 'Sent to Applicant'
  return 'Ready to Send'
}

const getBusinessContractStatusTone = (value = '') => {
  const normalizedStatus = normalizeBusinessContractStatus(value)
  if (normalizedStatus === 'completed') return 'success'
  if (normalizedStatus === 'applicant_signed') return 'info'
  if (normalizedStatus === 'sent') return 'warning'
  return 'muted'
}

const resolveOfferStatusLabel = (offerStatus = '', applicationStatus = '') => {
  if (normalizeValue(applicationStatus) === 'hired') return 'Hiring Confirmed'
  if (normalizeValue(offerStatus) === 'sent') return 'Offer Sent'
  return 'Offer Accepted'
}

const createDefaultBusinessContractDraft = () => ({
  applicationId: '',
  applicantId: '',
  applicantName: '',
  applicantEmail: '',
  jobId: '',
  jobTitle: '',
  companyName: '',
  businessName: '',
  contractTitle: '',
  employmentType: '',
  salaryOffer: '',
  startDate: '',
  notes: '',
  contractBody: '',
})

const contractSigningFilter = ref('all')
const contractSigningSelectedRowId = ref('')
const isBusinessContractSaving = ref(false)
const activeBusinessContractSigningId = ref('')
const businessContractSignatureName = ref('')
const businessContractDraft = ref(createDefaultBusinessContractDraft())
const liveBusinessContractRecords = ref([])
const feedback = ref({
  title: 'Realtime Contract Sync',
  text: 'Contract files sync in real time between the business Contract Signing page and the applicant Contracts page.',
  tone: 'info',
})
let stopBusinessContractSubscription = null

const canEditBusinessModule = (moduleId = '') =>
  !props.isEmployeeWorkspaceMode && text(moduleId) === 'contract-signing'

const buildBusinessContractBody = (row = {}) => {
  const applicantName = text(row?.name || row?.applicantName) || 'Applicant'
  const businessLabel = text(row?.businessName || row?.companyName || props.businessName) || 'Business Workspace'
  const jobTitle = text(row?.jobTitle || row?.role) || 'Applied Job'
  const compensation = text(row?.salaryOffer || row?.compensation) || 'To be finalized'
  const startDate = text(row?.startDate)
  const employmentType = text(row?.employmentType || row?.jobType) || 'Full-time'

  return [
    `This Employment Contract is entered into by and between ${businessLabel} and ${applicantName}.`,
    '',
    `${applicantName} is being offered the position of ${jobTitle} under a ${employmentType.toLowerCase()} arrangement.`,
    '',
    `Compensation: ${compensation}`,
    startDate ? `Target start date: ${startDate}` : 'Target start date: To be finalized with the applicant.',
    '',
    'This contract file will be shared through the portal so the applicant can download it, sign it offline, and return the signed copy here.',
  ].join('\n')
}

const buildBusinessContractDraftFromRow = (row = {}) => ({
  applicationId: text(row?.applicationId || row?.id),
  applicantId: text(row?.applicantId),
  applicantName: text(row?.name || row?.applicantName) || 'Applicant',
  applicantEmail: text(row?.email || row?.applicantEmail).toLowerCase(),
  jobId: text(row?.jobId),
  jobTitle: text(row?.jobTitle || row?.role) || 'Applied Job',
  companyName: text(row?.companyName || row?.businessName || props.businessName),
  businessName: text(row?.businessName || row?.companyName || props.businessName),
  contractTitle: text(row?.contractTitle) || `${text(row?.jobTitle || 'Employment')} Contract`,
  employmentType: text(row?.employmentType || row?.jobType) || 'Full-time',
  salaryOffer: text(row?.salaryOffer || row?.compensation),
  startDate: text(row?.startDate),
  notes: text(row?.notes),
  contractBody: text(row?.contractBody) || buildBusinessContractBody(row),
})

const setFeedback = (textValue = '', tone = 'info', title = 'Contract Signing') => {
  feedback.value = {
    title: text(title) || 'Contract Signing',
    text: text(textValue),
    tone: normalizeValue(tone) || 'info',
  }
}

const resolveBusinessContractStatusFromRecord = (record = {}) => {
  const explicitStatus = normalizeBusinessContractStatus(
    record?.contractStatus
    || record?.contract_status
    || record?.contractStage
    || record?.contract_stage,
  )

  if (explicitStatus !== 'draft') return explicitStatus
  if (text(record?.businessSignedAt || record?.business_signed_at)) return 'completed'
  if (text(record?.applicantSignedAt || record?.applicant_signed_at)) return 'applicant_signed'
  if (text(record?.contractSentAt || record?.contract_sent_at || record?.sentAt || record?.sent_at)) return 'sent'
  return 'draft'
}

const getRowActivityValue = (row = {}) =>
  Math.max(
    Date.parse(text(row?.updatedAt || row?.updated_at)) || 0,
    Date.parse(text(row?.businessSignedAt || row?.business_signed_at)) || 0,
    Date.parse(text(row?.applicantSignedAt || row?.applicant_signed_at)) || 0,
    Date.parse(text(row?.contractSentAt || row?.contract_sent_at || row?.sentAt || row?.sent_at)) || 0,
    Date.parse(text(row?.jobOfferApplicantRespondedAt || row?.job_offer_applicant_responded_at)) || 0,
    Date.parse(text(row?.jobOfferUpdatedAt || row?.job_offer_updated_at)) || 0,
    Date.parse(text(row?.jobOfferSentAt || row?.job_offer_sent_at)) || 0,
    Date.parse(text(row?.approvedAt || row?.reviewedAt || row?.appliedAt || row?.createdAt)) || 0,
  )

const liveContractRecordByApplicationId = computed(() => {
  const recordMap = new Map()

  liveBusinessContractRecords.value.forEach((record) => {
    const applicationId = text(record?.applicationId || record?.application_id)
    if (!applicationId) return

    const existingRecord = recordMap.get(applicationId)
    if (!existingRecord || getRowActivityValue(record) >= getRowActivityValue(existingRecord)) {
      recordMap.set(applicationId, record)
    }
  })

  return recordMap
})

const baseContractSigningRows = computed(() =>
  (Array.isArray(props.businessJobApplications) ? props.businessJobApplications : [])
    .filter((item) => {
      const offerStatus = normalizeValue(item?.jobOfferStatus || item?.job_offer_status || item?.offerStatus)
      const applicationStatus = normalizeValue(item?.status)
      return isAcceptedOfferStatus(offerStatus) || applicationStatus === 'hired'
    })
    .map((item, index) => {
      const applicationId = text(item?.id || item?.applicationId || item?.application_id)
      const applicantName = text(item?.applicantName || item?.applicant_name || item?.name) || `Applicant ${index + 1}`
      const offerStatus = normalizeValue(item?.jobOfferStatus || item?.job_offer_status || item?.offerStatus) || 'accepted'
      const applicationStatus = normalizeValue(item?.status)
      const linkedContractRecord = liveContractRecordByApplicationId.value.get(applicationId) || null
      const contractStatus = linkedContractRecord
        ? resolveBusinessContractStatusFromRecord(linkedContractRecord)
        : resolveBusinessContractStatusFromRecord(item)
      const businessName = text(
        item?.businessName
        || item?.business_name
        || item?.companyName
        || item?.company_name
        || props.businessName,
      ) || props.businessName
      const companyName = text(item?.companyName || item?.company_name || businessName) || businessName
      const compensation = text(
        item?.salaryOffer
        || item?.salary_offer
        || item?.jobOfferCompensation
        || item?.job_offer_compensation
        || item?.salaryRange
        || item?.salary_range,
      ) || 'To be finalized'
      const employmentType = text(
        item?.employmentType
        || item?.employment_type
        || item?.jobType
        || item?.job_type,
      ) || 'Full-time'
      const startDate = text(
        linkedContractRecord?.startDate
        || linkedContractRecord?.start_date
        || item?.startDate
        || item?.start_date
        || item?.jobOfferStartDate
        || item?.job_offer_start_date,
      )
      const sentAt = text(
        linkedContractRecord?.sentAt
        || linkedContractRecord?.sent_at
        || item?.contractSentAt
        || item?.contract_sent_at
        || item?.sentAt
        || item?.sent_at,
      )
      const applicantSignedAt = text(linkedContractRecord?.applicantSignedAt || linkedContractRecord?.applicant_signed_at || item?.applicantSignedAt || item?.applicant_signed_at)
      const businessSignedAt = text(linkedContractRecord?.businessSignedAt || linkedContractRecord?.business_signed_at || item?.businessSignedAt || item?.business_signed_at)

      const nextRow = {
        id: applicationId || `contract-row-${index + 1}`,
        contractId: text(linkedContractRecord?.id || item?.contractId || item?.contract_id),
        applicationId: applicationId || `contract-row-${index + 1}`,
        applicantId: text(item?.applicantId || item?.applicant_id || item?.userId || item?.user_id),
        name: applicantName,
        email: text(item?.applicantEmail || item?.applicant_email || item?.email) || 'No email',
        jobId: text(item?.jobId || item?.job_id),
        jobTitle: text(item?.jobTitle || item?.job_title || item?.role || item?.position) || 'Open role',
        role: text(item?.jobTitle || item?.job_title || item?.role || item?.position) || 'Open role',
        jobType: employmentType,
        companyName,
        businessName,
        offerTitle: text(item?.jobOfferTitle || item?.job_offer_title || item?.offerTitle) || 'Issued Job Offer',
        offerStatus,
        offerStatusLabel: resolveOfferStatusLabel(offerStatus, applicationStatus),
        compensation,
        startDate,
        status: contractStatus,
        statusLabel: getBusinessContractStatusLabel(contractStatus),
        statusTone: getBusinessContractStatusTone(contractStatus),
        contractTitle: text(linkedContractRecord?.contractTitle || linkedContractRecord?.contract_title || item?.contractTitle || item?.contract_title) || `${text(item?.jobTitle || item?.job_title || 'Employment')} Contract`,
        employmentType: text(linkedContractRecord?.employmentType || linkedContractRecord?.employment_type || employmentType),
        salaryOffer: text(linkedContractRecord?.salaryOffer || linkedContractRecord?.salary_offer || compensation),
        notes: text(linkedContractRecord?.notes || item?.notes),
        contractBody: text(linkedContractRecord?.contractBody || linkedContractRecord?.contract_body || item?.contractBody || item?.contract_body),
        businessContractFileName: text(linkedContractRecord?.businessContractFileName || linkedContractRecord?.business_contract_file_name),
        businessContractDownloadUrl: text(linkedContractRecord?.businessContractDownloadUrl || linkedContractRecord?.business_contract_download_url),
        businessContractStoragePath: text(linkedContractRecord?.businessContractStoragePath || linkedContractRecord?.business_contract_storage_path),
        businessContractUploadedAt: text(linkedContractRecord?.businessContractUploadedAt || linkedContractRecord?.business_contract_uploaded_at),
        applicantSignedContractFileName: text(linkedContractRecord?.applicantSignedContractFileName || linkedContractRecord?.applicant_signed_contract_file_name),
        applicantSignedContractDownloadUrl: text(linkedContractRecord?.applicantSignedContractDownloadUrl || linkedContractRecord?.applicant_signed_contract_download_url),
        applicantSignedContractStoragePath: text(linkedContractRecord?.applicantSignedContractStoragePath || linkedContractRecord?.applicant_signed_contract_storage_path),
        applicantSignedContractUploadedAt: text(linkedContractRecord?.applicantSignedContractUploadedAt || linkedContractRecord?.applicant_signed_contract_uploaded_at),
        sentAt,
        applicantSignedAt,
        businessSignedAt,
        applicantSignatureDataUrl: text(linkedContractRecord?.applicantSignatureDataUrl || linkedContractRecord?.applicant_signature_data_url || item?.applicantSignatureDataUrl || item?.applicant_signature_data_url),
        businessSignatureDataUrl: text(linkedContractRecord?.businessSignatureDataUrl || linkedContractRecord?.business_signature_data_url || item?.businessSignatureDataUrl || item?.business_signature_data_url),
        businessSignatureName: text(linkedContractRecord?.businessSignatureName || linkedContractRecord?.business_signature_name || item?.businessSignatureName || item?.business_signature_name),
        offerAcceptedAt: text(
          item?.jobOfferApplicantRespondedAt
          || item?.job_offer_applicant_responded_at
          || item?.jobOfferUpdatedAt
          || item?.job_offer_updated_at
          || item?.approvedAt
          || item?.reviewedAt,
        ),
        lastActivityValue: getRowActivityValue(item),
      }

      nextRow.contractBody = nextRow.contractBody || buildBusinessContractBody(nextRow)
      nextRow.canSend = !['applicant_signed', 'completed'].includes(contractStatus)
      nextRow.canBusinessSign = false

      return nextRow
    })
    .sort((left, right) => right.lastActivityValue - left.lastActivityValue),
)

const contractSigningQueueRows = computed(() =>
  baseContractSigningRows.value
    .map((row) => {
      const status = normalizeBusinessContractStatus(row.status)

      return {
        ...row,
        status,
        contractId: text(row.contractId),
        contractTitle: text(row.contractTitle) || `${text(row.jobTitle || 'Employment')} Contract`,
        employmentType: text(row.employmentType || row.jobType) || 'Full-time',
        salaryOffer: text(row.salaryOffer || row.compensation),
        contractBody: text(row.contractBody) || buildBusinessContractBody(row),
        statusLabel: getBusinessContractStatusLabel(status),
        statusTone: getBusinessContractStatusTone(status),
        canSend: !['applicant_signed', 'completed'].includes(status),
        canBusinessSign: false,
        lastActivityValue: Math.max(
          Number(row.lastActivityValue) || 0,
          Date.parse(text(row.applicantSignedContractUploadedAt)) || 0,
          Date.parse(text(row.businessContractUploadedAt)) || 0,
          Date.parse(text(row.businessSignedAt)) || 0,
          Date.parse(text(row.applicantSignedAt)) || 0,
          Date.parse(text(row.sentAt)) || 0,
        ),
      }
    })
    .sort((left, right) => right.lastActivityValue - left.lastActivityValue),
)

const contractSigningOverviewCards = computed(() => ([
  {
    label: 'Ready For Contract',
    value: String(contractSigningQueueRows.value.length),
    copy: 'Accepted offers that are now visible in the live Contract Signing workspace.',
  },
  {
    label: 'Ready To Send',
    value: String(contractSigningQueueRows.value.filter((row) => row.status === 'draft').length),
    copy: 'Accepted offers that are still waiting for the initial contract file upload.',
  },
  {
    label: 'Awaiting Applicant',
    value: String(contractSigningQueueRows.value.filter((row) => row.status === 'sent').length),
    copy: 'Contracts already sent and waiting for the applicant to return a signed copy.',
  },
  {
    label: 'Completed',
    value: String(contractSigningQueueRows.value.filter((row) => row.status === 'completed').length),
    copy: 'Completed contract threads currently visible in the live queue.',
  },
]))

const contractSigningFilterChips = computed(() => {
  const rows = contractSigningQueueRows.value
  return [
    { id: 'all', label: 'All', count: rows.length },
    { id: 'draft', label: 'Ready To Send', count: rows.filter((row) => row.status === 'draft').length },
    { id: 'sent', label: 'Awaiting Applicant', count: rows.filter((row) => row.status === 'sent').length },
    { id: 'applicant_signed', label: 'Returned', count: rows.filter((row) => row.status === 'applicant_signed').length },
    { id: 'completed', label: 'Completed', count: rows.filter((row) => row.status === 'completed').length },
  ]
})

const contractSigningLastSyncedLabel = computed(() =>
  contractSigningQueueRows.value.length
    ? 'Live contract records are synced with the current business job application records.'
    : 'No accepted job offers are ready for contract signing yet.',
)

const contractSigningTraceSummary = computed(() =>
  `${contractSigningQueueRows.value.length} contract-ready applicant record(s) visible in this live business queue.`,
)

const filteredContractSigningQueueRows = computed(() => {
  const activeFilter = text(contractSigningFilter.value || 'all') || 'all'
  if (activeFilter === 'all') return contractSigningQueueRows.value
  return contractSigningQueueRows.value.filter((row) => row.status === activeFilter)
})

const selectedContractSigningRow = computed(() =>
  filteredContractSigningQueueRows.value.find((row) => row.id === contractSigningSelectedRowId.value)
  || filteredContractSigningQueueRows.value[0]
  || contractSigningQueueRows.value[0]
  || null,
)

const selectedContractSigningRecord = computed(() =>
  liveContractRecordByApplicationId.value.get(text(selectedContractSigningRow.value?.applicationId))
  || selectedContractSigningRow.value,
)

const restoreSelectedContractSigningDraft = () => {
  businessContractDraft.value = selectedContractSigningRow.value
    ? buildBusinessContractDraftFromRow(selectedContractSigningRow.value)
    : createDefaultBusinessContractDraft()
}

const selectContractSigningRow = (rowId = '') => {
  const normalizedRowId = text(rowId)
  if (!normalizedRowId) return

  contractSigningSelectedRowId.value = normalizedRowId
  restoreSelectedContractSigningDraft()
}

const setContractDraftField = (field, value) => {
  businessContractDraft.value = {
    ...businessContractDraft.value,
    [field]: value,
  }
}

const setContractSigningFilter = (value = '') => {
  contractSigningFilter.value = text(value || 'all') || 'all'
}

const setBusinessContractSignatureName = (value = '') => {
  businessContractSignatureName.value = text(value)
}

const refreshContractSigningQueue = () => {
  startBusinessContractSubscription()
  setFeedback(
    'The contract queue refreshed from the live Firestore records.',
    'info',
    'Queue refreshed',
  )
}

const mergeLiveBusinessContractRecord = (record = {}) => {
  const recordId = text(record?.id)
  const applicationId = text(record?.applicationId || record?.application_id)
  if (!recordId && !applicationId) return

  const nextRecords = Array.isArray(liveBusinessContractRecords.value)
    ? [...liveBusinessContractRecords.value]
    : []
  const existingIndex = nextRecords.findIndex((item) =>
    text(item?.id) === recordId
    || text(item?.applicationId || item?.application_id) === applicationId,
  )

  if (existingIndex >= 0) {
    nextRecords[existingIndex] = {
      ...nextRecords[existingIndex],
      ...record,
    }
  } else {
    nextRecords.unshift(record)
  }

  liveBusinessContractRecords.value = nextRecords
    .map((item) => ({ ...item }))
    .sort((left, right) => getRowActivityValue(right) - getRowActivityValue(left))
}

const startBusinessContractSubscription = () => {
  stopBusinessContractSubscription?.()

  const workspaceOwnerId = text(props.workspaceOwnerId)
  const workspaceOwnerEmail = normalizeValue(props.workspaceOwnerEmail)
  const applicationIds = (Array.isArray(props.businessJobApplications) ? props.businessJobApplications : [])
    .map((record) => text(record?.id || record?.applicationId || record?.application_id))
    .filter(Boolean)
  if (!workspaceOwnerId && !workspaceOwnerEmail && !applicationIds.length) {
    liveBusinessContractRecords.value = []
    return
  }

  stopBusinessContractSubscription = subscribeToBusinessContracts(
    {
      workspaceOwnerId,
      workspaceOwnerEmail,
      applicationIds,
    },
    (records) => {
      liveBusinessContractRecords.value = Array.isArray(records) ? records : []
    },
    () => {
      liveBusinessContractRecords.value = []
    },
  )
}

const sendContractToApplicant = async (rowId = '', payload = {}) => {
  if (isBusinessContractSaving.value) {
    return {
      ok: false,
      message: 'May kasalukuyang contract send request pa. Hintayin muna itong matapos.',
    }
  }

  if (!canEditBusinessModule('contract-signing')) {
    const message = 'Contract sending is only available in the main business owner workspace.'
    setFeedback(message, 'warning', 'Owner access required')
    return {
      ok: false,
      message,
    }
  }

  const targetId = text(rowId) || text(selectedContractSigningRow.value?.id)
  const targetRow = contractSigningQueueRows.value.find((row) => row.id === targetId) || selectedContractSigningRow.value
  if (!targetRow) {
    return {
      ok: false,
      message: 'Walang valid contract row na napili para sa send action.',
    }
  }

  const selectedFile = payload?.file
  if (!(typeof File !== 'undefined' && selectedFile instanceof File)) {
    const message = 'Upload the contract file before sending it to the applicant.'
    setFeedback(message, 'warning', 'File required')
    return {
      ok: false,
      message,
    }
  }

  const missingFields = []
  if (!text(props.workspaceOwnerId)) missingFields.push('workspace owner')
  if (!text(targetRow.applicationId || targetRow.id)) missingFields.push('application')
  if (!text(targetRow.applicantId)) missingFields.push('applicant')
  if (!text(targetRow.jobId)) missingFields.push('job')

  if (missingFields.length) {
    const message = `Missing required contract details: ${missingFields.join(', ')}.`
    setFeedback(message, 'error', 'Contract data incomplete')
    return {
      ok: false,
      message,
    }
  }

  isBusinessContractSaving.value = true
  try {
    const draftSource = targetId === text(selectedContractSigningRow.value?.id)
      ? businessContractDraft.value
      : buildBusinessContractDraftFromRow(targetRow)

    const savedRecord = await sendBusinessContractFile({
      id: text(targetRow.contractId),
      workspaceOwnerId: text(props.workspaceOwnerId),
      workspaceOwnerEmail: text(props.workspaceOwnerEmail),
      workspaceOwnerName: text(props.businessName),
      applicationId: text(targetRow.applicationId || targetRow.id),
      applicantId: text(targetRow.applicantId),
      applicantName: text(targetRow.name),
      applicantEmail: text(targetRow.email).toLowerCase(),
      jobId: text(targetRow.jobId),
      jobTitle: text(targetRow.jobTitle || targetRow.role),
      companyName: text(targetRow.companyName || targetRow.businessName || props.businessName),
      businessName: text(targetRow.businessName || targetRow.companyName || props.businessName),
      contractTitle: text(draftSource.contractTitle) || targetRow.contractTitle,
      employmentType: text(draftSource.employmentType) || targetRow.employmentType,
      salaryOffer: text(draftSource.salaryOffer) || targetRow.salaryOffer,
      startDate: text(draftSource.startDate) || targetRow.startDate,
      notes: text(draftSource.notes),
      contractBody: text(draftSource.contractBody) || buildBusinessContractBody({ ...targetRow, ...draftSource }),
    }, selectedFile)
    mergeLiveBusinessContractRecord(savedRecord)

    contractSigningSelectedRowId.value = targetRow.id
    restoreSelectedContractSigningDraft()
    const successMessage = `${targetRow.name} received the contract file in real time. The applicant can now download it from the Contracts page.`
    setFeedback(
      successMessage,
      'success',
      'Contract sent',
    )
    return {
      ok: true,
      message: successMessage,
    }
  } catch (error) {
    const message = error instanceof Error ? error.message : 'Unable to send the contract file right now.'
    setFeedback(
      message,
      'error',
      'Send failed',
    )
    return {
      ok: false,
      message,
    }
  } finally {
    isBusinessContractSaving.value = false
  }
}

const saveAndSendBusinessContract = () => {
  if (!selectedContractSigningRow.value) return
  sendContractToApplicant(selectedContractSigningRow.value.id)
}

const completeBusinessContractSigning = ({ signatureDataUrl } = {}) => {
  setFeedback(
    signatureDataUrl
      ? 'Business-side countersigning is not needed in the current file-return flow.'
      : 'Business-side countersigning is not needed in the current file-return flow.',
    'info',
    'File-based flow active',
  )
}

const feedbackToneClass = computed(() => {
  const normalizedTone = normalizeValue(feedback.value?.tone)
  if (normalizedTone === 'success') return 'is-success'
  if (normalizedTone === 'warning') return 'is-warning'
  if (normalizedTone === 'error') return 'is-error'
  return 'is-info'
})

watch(filteredContractSigningQueueRows, (rows) => {
  const availableIds = rows.map((row) => text(row?.id)).filter(Boolean)
  if (!availableIds.length) {
    contractSigningSelectedRowId.value = ''
    businessContractDraft.value = createDefaultBusinessContractDraft()
    return
  }

  if (!availableIds.includes(text(contractSigningSelectedRowId.value))) {
    contractSigningSelectedRowId.value = availableIds[0]
  }

  restoreSelectedContractSigningDraft()
}, { immediate: true })

watch(
  () => [
    text(props.workspaceOwnerId),
    normalizeValue(props.workspaceOwnerEmail),
    (Array.isArray(props.businessJobApplications) ? props.businessJobApplications : [])
      .map((record) => text(record?.id || record?.applicationId || record?.application_id))
      .filter(Boolean)
      .join('|'),
  ].join('|'),
  () => {
    startBusinessContractSubscription()
  },
  { immediate: true },
)

onBeforeUnmount(() => {
  stopBusinessContractSubscription?.()
})
</script>

<template>
  <section class="business-shell__stage">
    <article class="business-contract-signing-stage__notice" :class="feedbackToneClass">
      <strong>{{ feedback.title }}</strong>
      <p>{{ feedback.text }}</p>
    </article>

    <BusinessContractSigningContent
      :overview-cards="contractSigningOverviewCards"
      :filter-chips="contractSigningFilterChips"
      :active-filter="contractSigningFilter"
      :set-filter="setContractSigningFilter"
      :sync-label="contractSigningLastSyncedLabel"
      :trace-summary="contractSigningTraceSummary"
      :refresh-queue="refreshContractSigningQueue"
      :rows="filteredContractSigningQueueRows"
      :selected-row-id="contractSigningSelectedRowId"
      :select-row="selectContractSigningRow"
      :send-contract-to-applicant="sendContractToApplicant"
      :selected-row="selectedContractSigningRow"
      :selected-record="selectedContractSigningRecord"
      :contract-draft="businessContractDraft"
      :set-contract-draft-field="setContractDraftField"
      :restore-contract-draft="restoreSelectedContractSigningDraft"
      :save-and-send-business-contract="saveAndSendBusinessContract"
      :is-business-contract-saving="isBusinessContractSaving"
      :can-edit-business-module="canEditBusinessModule"
      :business-contract-signature-name="businessContractSignatureName"
      :set-business-contract-signature-name="setBusinessContractSignatureName"
      :active-business-contract-signing-id="activeBusinessContractSigningId"
      :complete-business-contract-signing="completeBusinessContractSigning"
    />
  </section>
</template>

<style scoped>
.business-contract-signing-stage__notice {
  display: grid;
  gap: 0.35rem;
  padding: 1rem 1.15rem;
  border-radius: 1rem;
  border: 1px solid rgba(206, 220, 212, 0.92);
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.98), rgba(247, 251, 248, 0.96));
}

.business-contract-signing-stage__notice strong,
.business-contract-signing-stage__notice p {
  margin: 0;
}

.business-contract-signing-stage__notice strong {
  color: #183126;
  font-size: 0.95rem;
}

.business-contract-signing-stage__notice p {
  color: #60786a;
  line-height: 1.55;
}

.business-contract-signing-stage__notice.is-success {
  border-color: rgba(169, 214, 184, 0.92);
  background: linear-gradient(180deg, rgba(244, 252, 247, 0.98), rgba(236, 248, 240, 0.96));
}

.business-contract-signing-stage__notice.is-warning {
  border-color: rgba(233, 211, 167, 0.92);
  background: linear-gradient(180deg, rgba(255, 250, 242, 0.98), rgba(255, 246, 228, 0.96));
}

.business-contract-signing-stage__notice.is-error {
  border-color: rgba(236, 193, 193, 0.92);
  background: linear-gradient(180deg, rgba(255, 247, 247, 0.98), rgba(255, 239, 239, 0.96));
}
</style>
