<script setup>
import { computed, onBeforeUnmount, reactive, ref, watch } from 'vue'
import { updateApplicantJobApplicationStatus } from '@/lib/apply_jobs'
import {
  deleteBusinessAssessmentAssignmentRecord,
  deleteBusinessAssessmentTemplateRecord,
  saveBusinessAssessmentAssignmentRecord,
  saveBusinessAssessmentTemplateRecord,
  subscribeToBusinessAssessmentAssignments,
  subscribeToBusinessAssessmentTemplates,
} from '@/lib/business_workspace_records'
import '@/components/businesss.css'

const props = defineProps({
  activeSection: { type: String, default: 'assessment-management' },
  businessJobApplications: { type: Array, default: () => [] },
  workspaceOwnerId: { type: String, default: '' },
  businessName: { type: String, default: 'Business Workspace' },
  isEmployeeWorkspaceMode: { type: Boolean, default: false },
})

const text = (value = '') => String(value ?? '').trim()
const normalize = (value = '') => text(value).toLowerCase()
const toIsoNow = () => new Date().toISOString()
const buildInitials = (value = '') =>
  text(value).split(/\s+/).filter(Boolean).slice(0, 2).map((part) => part.charAt(0).toUpperCase()).join('') || 'AP'
const ASSESSMENT_OPTION_SLOT_COUNT = 4
const normalizePercent = (value, fallback = 70) => {
  const parsedValue = Number.parseInt(String(value ?? '').trim(), 10)
  if (!Number.isFinite(parsedValue)) return fallback
  return Math.min(100, Math.max(1, parsedValue))
}
const formatStatusLabel = (value = '', fallback = 'Pending') => {
  const normalizedValue = normalize(value)
  if (!normalizedValue) return fallback
  return normalizedValue
    .split(/[_\s-]+/)
    .filter(Boolean)
    .map((part) => part.charAt(0).toUpperCase() + part.slice(1))
    .join(' ')
}
const isSubmittedAssessmentStatus = (value = '') => ['submitted', 'completed'].includes(normalize(value))
const isFinalAssessmentResult = (value = '') => ['passed', 'failed'].includes(normalize(value))
const buildMultipleChoiceOptionDrafts = (options = []) => {
  const normalizedOptions = (Array.isArray(options) ? options : [])
    .map((option) => text(typeof option === 'string' ? option : option?.label || option?.value))

  while (normalizedOptions.length < ASSESSMENT_OPTION_SLOT_COUNT) {
    normalizedOptions.push('')
  }

  return normalizedOptions.slice(0, ASSESSMENT_OPTION_SLOT_COUNT)
}
const normalizeQuestionType = (value = '') => {
  const normalizedValue = normalize(value).replace(/\s+/g, '-')
  if (['multiple-choice'].includes(normalizedValue)) return 'multiple-choice'
  return 'multiple-choice'
}
const formatQuestionTypeLabel = (value = '') => {
  const normalizedValue = normalizeQuestionType(value)
  if (normalizedValue === 'multiple-choice') return 'Multiple Choice'
  return 'Multiple Choice'
}
const resolveAssessmentTab = (value = '') => {
  const normalizedValue = normalize(value)
  if (normalizedValue === 'assessment-assignment') return 'assign'
  if (normalizedValue === 'applicant-score') return 'score'
  return 'builder'
}

const defaultTemplates = [
  {
    id: 'assessment-template-1',
    title: 'Customer Support Screening',
    description: 'Measure communication, empathy, and problem-solving for support-facing roles.',
    category: 'Customer Service',
    passingScore: 75,
    questions: [
      {
        id: 'question-1',
        question: 'What is the best first response to an upset customer in chat support?',
        type: 'multiple-choice',
        options: [
          'Acknowledge the concern and ask how you can help.',
          'End the chat and ask them to email support.',
          'Tell the customer to calm down first.',
          'Wait for the customer to stop typing before replying.',
        ],
        correctOptionIndex: 0,
      },
      {
        id: 'question-2',
        question: 'When should a support concern be escalated to a supervisor?',
        type: 'multiple-choice',
        options: [
          'When the issue needs higher approval or is outside your access.',
          'Every time a customer sounds frustrated.',
          'Only after the customer posts on social media.',
          'Only at the end of the workday.',
        ],
        correctOptionIndex: 0,
      },
    ],
  },
]
const fallbackApplicants = [
  { id: 'fallback-1', applicantName: 'Maria Santos', applicantEmail: 'maria.santos@example.com', applicantId: 'applicant-1', jobId: 'job-1', jobTitle: 'Cashier' },
]

const activeTab = ref(resolveAssessmentTab(props.activeSection))
const selectedTemplateId = ref('')
const templateRecords = ref([])
const assignmentRecords = ref([])
const hasLoadedTemplates = ref(false)
const isTemplateSaving = ref(false)
const isTemplateDeleting = ref(false)
const pendingAssignmentRowId = ref('')
const pendingRemovalRowId = ref('')
const feedback = reactive({ message: '', tone: 'success' })
const assignmentDraftByApplicationId = reactive({})
const assessmentAssignmentFilters = reactive({ search: '', roleFilter: 'all' })
const applicantScoreFilters = reactive({ search: '', roleFilter: 'all' })
const assessmentDraft = reactive({
  title: '',
  description: '',
  category: '',
  passingScore: 70,
  questions: [],
})

let stopTemplateSubscription = () => {}
let stopAssignmentSubscription = () => {}

const setFeedback = (message = '', tone = 'success') => {
  feedback.message = text(message)
  feedback.tone = text(tone) || 'success'
}
const isReadOnly = computed(() => props.isEmployeeWorkspaceMode === true)
const mapDraftQuestion = (question = {}, index = 0) => ({
  id: text(question?.id) || `question-${index + 1}`,
  question: text(question?.question || question?.label) || `Question ${index + 1}`,
  type: 'multiple-choice',
  options: buildMultipleChoiceOptionDrafts(question?.options),
  correctOptionIndex: Number.isInteger(question?.correctOptionIndex)
    ? question.correctOptionIndex
    : Number.isInteger(question?.correct_option_index)
      ? question.correct_option_index
      : 0,
})
const toDraftQuestions = (questions = []) => (Array.isArray(questions) ? questions : []).map(mapDraftQuestion)
const getQuestionFilledOptions = (question = {}) =>
  buildMultipleChoiceOptionDrafts(question?.options).map((option) => text(option))
const getStoredQuestion = (question = {}, index = 0) => {
  const optionEntries = getQuestionFilledOptions(question)
    .map((option, optionIndex) => ({ option, optionIndex }))
    .filter((entry) => entry.option)
  const storedOptions = optionEntries.map((entry) => entry.option)
  const selectedDraftIndex = Number(question?.correctOptionIndex)
  const matchedCorrectEntry = optionEntries.find((entry) => entry.optionIndex === selectedDraftIndex)
  const correctOptionIndex = matchedCorrectEntry ? storedOptions.indexOf(matchedCorrectEntry.option) : -1

  return {
    id: text(question?.id) || `question-${index + 1}`,
    label: text(question?.question || question?.label) || `Question ${index + 1}`,
    helpText: '',
    type: 'multiple-choice',
    required: true,
    options: storedOptions,
    correctOptionIndex,
  }
}
const toStoredQuestions = (questions = []) =>
  (Array.isArray(questions) ? questions : []).map(getStoredQuestion)
const getQuestionValidationMessage = (question = {}, index = 0) => {
  if (!text(question?.question)) return `Question ${index + 1} needs a prompt.`

  const optionEntries = getQuestionFilledOptions(question)
    .map((option, optionIndex) => ({ option, optionIndex }))
    .filter((entry) => entry.option)

  if (optionEntries.length < 2) return `Question ${index + 1} needs at least 2 answer choices.`

  const selectedDraftIndex = Number(question?.correctOptionIndex)
  if (!optionEntries.some((entry) => entry.optionIndex === selectedDraftIndex)) {
    return `Question ${index + 1} needs a valid correct answer.`
  }

  return ''
}

const templates = computed(() => {
  if (templateRecords.value.length) {
    return templateRecords.value.map((record, index) => ({
      id: text(record?.id) || `assessment-template-${index + 1}`,
      title: text(record?.title) || `Assessment Template ${index + 1}`,
      description: text(record?.description),
      category: text(record?.category) || 'General',
      passingScore: normalizePercent(record?.passingScorePercent ?? record?.passing_score_percent, 70),
      questions: toDraftQuestions(record?.questions),
    }))
  }
  return hasLoadedTemplates.value ? [] : defaultTemplates
})
const templateMap = computed(() => new Map(templates.value.map((template) => [text(template.id), template])))
const assignmentMap = computed(() => {
  const lookup = new Map()
  assignmentRecords.value.forEach((record) => {
    const applicationId = text(record?.applicationId || record?.application_id || record?.id)
    if (applicationId) lookup.set(applicationId, record)
  })
  return lookup
})
const applicationRows = computed(() =>
  Array.isArray(props.businessJobApplications) && props.businessJobApplications.length ? props.businessJobApplications : fallbackApplicants,
)
const syncDraft = (templateId = '') => {
  const template = templateMap.value.get(text(templateId))
  if (!template) return
  selectedTemplateId.value = template.id
  assessmentDraft.title = template.title
  assessmentDraft.description = template.description
  assessmentDraft.category = template.category
  assessmentDraft.passingScore = template.passingScore
  assessmentDraft.questions = toDraftQuestions(template.questions)
}
const startNewTemplate = () => {
  selectedTemplateId.value = ''
  assessmentDraft.title = ''
  assessmentDraft.description = ''
  assessmentDraft.category = ''
  assessmentDraft.passingScore = 70
  assessmentDraft.questions = [{ id: `question-${Date.now()}`, question: '', type: 'multiple-choice', options: buildMultipleChoiceOptionDrafts(), correctOptionIndex: 0 }]
}
const findTemplateByTitle = (value = '') =>
  templates.value.find((template) => normalize(template.title) === normalize(value)) || null
const getAssignmentState = (record = {}, index = 0) => {
  const applicationId = text(record?.id || record?.applicationId || record?.application_id || `application-${index + 1}`)
  const persisted = assignmentMap.value.get(applicationId) || null
  const stagedTemplateId = text(assignmentDraftByApplicationId[applicationId])
  const persistedTemplateId = text(persisted?.selectedTemplateId || persisted?.selected_template_id || persisted?.templateId || persisted?.template_id)
  const persistedTemplateTitle = text(persisted?.templateTitle || persisted?.template_title || record?.assessmentTemplateTitle || record?.assessment_template_title)
  const template = templateMap.value.get(stagedTemplateId || persistedTemplateId) || findTemplateByTitle(persistedTemplateTitle)
  const templateId = stagedTemplateId || persistedTemplateId || text(template?.id)
  const templateName = persistedTemplateTitle || text(template?.title)
  const rawStatus = text(
    persisted?.assessmentStatus
    || persisted?.assessment_status
    || persisted?.assignmentStatus
    || persisted?.assignment_status
    || record?.technicalAssessmentStatus
    || record?.technical_assessment_status
    || (stagedTemplateId ? 'draft' : templateName ? 'assigned' : 'not assigned'),
  )

  return { applicationId, persisted, templateId, templateName, rawStatus }
}

const assignmentRows = computed(() =>
  applicationRows.value.map((record, index) => {
    const name = text(record?.applicantName || record?.applicant_name || record?.name) || `Applicant ${index + 1}`
    const assignmentState = getAssignmentState(record, index)
    const persistedStatus = text(
      assignmentState.persisted?.assessmentStatus
      || assignmentState.persisted?.assessment_status
      || assignmentState.persisted?.assignmentStatus
      || assignmentState.persisted?.assignment_status
      || record?.technicalAssessmentStatus
      || record?.technical_assessment_status,
    )
    const persistedResult = text(
      assignmentState.persisted?.assessmentResult
      || assignmentState.persisted?.assessment_result
      || record?.technicalAssessmentResult
      || record?.technical_assessment_result,
    )
    const submittedAt = text(
      assignmentState.persisted?.submittedAt
      || assignmentState.persisted?.submitted_at
      || record?.technicalAssessmentSubmittedAt
      || record?.technical_assessment_submitted_at,
    )
    const isSubmissionLocked = Boolean(submittedAt)
      || isSubmittedAssessmentStatus(persistedStatus)
      || isFinalAssessmentResult(persistedResult)
    const assignmentNote = isSubmissionLocked
      ? 'Applicant already submitted this assessment. Assignment is now locked.'
      : assignmentState.persisted
        ? 'Live assignment synced to applicant technical assessment'
        : assignmentState.templateId
          ? 'Template selected and ready to assign'
          : 'Select a template before assigning'

    return {
      id: assignmentState.applicationId,
      applicationId: assignmentState.applicationId,
      applicantId: text(record?.applicantId || record?.applicant_id || record?.userId || record?.user_id),
      applicantName: name,
      applicantEmail: text(record?.applicantEmail || record?.applicant_email || record?.email) || 'No email',
      applicantAvatar: text(record?.applicantAvatar || record?.avatar || record?.avatarUrl),
      applicantInitials: buildInitials(name),
      jobId: text(record?.jobId || record?.job_id),
      jobTitle: text(record?.jobTitle || record?.job_title || record?.role || record?.position) || 'Applicant',
      companyLabel: text(record?.businessName || record?.companyName || props.businessName || 'Business application'),
      selectedTemplateId: assignmentState.templateId,
      templateName: assignmentState.templateName || 'No template selected',
      assignmentStatusLabel: formatStatusLabel(assignmentState.rawStatus, 'Not Assigned'),
      assignmentStatusClass: normalize(assignmentState.rawStatus).replace(/[_\s]+/g, '-') || 'not-assigned',
      assignmentNote,
      isSubmissionLocked,
      canAssign: !isSubmissionLocked && !isReadOnly.value && Boolean(assignmentState.templateId),
      canRemove: !isSubmissionLocked && !isReadOnly.value && Boolean(assignmentState.templateId),
    }
  }),
)
const filteredAssignmentRows = computed(() =>
  assignmentRows.value.filter((row) => {
    const query = normalize(assessmentAssignmentFilters.search)
    const roleFilter = normalize(assessmentAssignmentFilters.roleFilter)
    const matchesQuery = !query || [row.applicantName, row.applicantEmail, row.jobTitle, row.templateName].some((value) => normalize(value).includes(query))
    const matchesRole = roleFilter === 'all' || normalize(row.jobTitle) === roleFilter
    return matchesQuery && matchesRole
  }),
)
const assignmentRoleOptions = computed(() => [{ value: 'all', label: 'All Roles' }].concat(
  [...new Set(assignmentRows.value.map((row) => text(row.jobTitle)).filter(Boolean))].map((role) => ({ value: role, label: role })),
))
const assignmentSummary = computed(() => {
  const count = assignmentRows.value.filter((row) => row.selectedTemplateId).length
  return `${count} assignment${count === 1 ? '' : 's'} ready`
})
const scoreRows = computed(() =>
  assignmentRows.value.map((row) => {
    const persisted = assignmentMap.value.get(row.applicationId) || null
    const rawStatus = normalize(
      persisted?.assessmentStatus
      || persisted?.assessment_status
      || persisted?.assignmentStatus
      || persisted?.assignment_status
      || 'pending',
    )
    const assessmentResult = normalize(
      persisted?.assessmentResult
      || persisted?.assessment_result
      || (['submitted', 'completed'].includes(rawStatus) ? 'pending' : rawStatus),
    )
    const scoreValue = Number(persisted?.assessmentScoreValue ?? persisted?.assessment_score_value ?? 0) || 0
    const scoreStatus = assessmentResult === 'passed'
      ? 'passed'
      : assessmentResult === 'failed'
        ? 'failed'
        : rawStatus === 'cancelled'
          ? 'cancelled'
          : 'pending'

    return {
      ...row,
      scoreValue: `${scoreValue}%`,
      scoreTone: scoreStatus === 'passed'
        ? scoreValue >= 85 ? 'top-score' : 'qualified'
        : scoreStatus === 'failed'
          ? 'failed-score'
          : scoreStatus === 'cancelled'
            ? 'pending'
            : 'pending',
      statusLabel: formatStatusLabel(scoreStatus, 'Pending'),
      statusClass: scoreStatus.replace(/[_\s]+/g, '-') || 'pending',
      resultLabel: formatStatusLabel(assessmentResult || (rawStatus === 'assigned' ? 'pending' : rawStatus), 'Pending'),
    }
  }),
)
const filteredScoreRows = computed(() =>
  scoreRows.value.filter((row) => {
    const query = normalize(applicantScoreFilters.search)
    const roleFilter = normalize(applicantScoreFilters.roleFilter)
    const matchesQuery = !query || [row.applicantName, row.applicantEmail, row.jobTitle, row.templateName].some((value) => normalize(value).includes(query))
    const matchesRole = roleFilter === 'all' || normalize(row.jobTitle) === roleFilter
    return matchesQuery && matchesRole
  }),
)
const scoreRoleOptions = computed(() => [{ value: 'all', label: 'All Roles' }].concat(
  [...new Set(scoreRows.value.map((row) => text(row.jobTitle)).filter(Boolean))].map((role) => ({ value: role, label: role })),
))

watch(() => props.activeSection, (value) => { activeTab.value = resolveAssessmentTab(value) }, { immediate: true })
watch(templates, (value) => {
  if (selectedTemplateId.value && value.some((template) => template.id === selectedTemplateId.value)) return
  if (value.length) syncDraft(value[0].id)
  else if (!assessmentDraft.questions.length) startNewTemplate()
}, { immediate: true })
watch(() => props.workspaceOwnerId, (workspaceOwnerId) => {
  stopTemplateSubscription()
  stopAssignmentSubscription()
  templateRecords.value = []
  assignmentRecords.value = []
  hasLoadedTemplates.value = false
  if (!text(workspaceOwnerId)) return

  stopTemplateSubscription = subscribeToBusinessAssessmentTemplates(
    text(workspaceOwnerId),
    (records = []) => {
      templateRecords.value = Array.isArray(records) ? records : []
      hasLoadedTemplates.value = true
    },
    (error) => {
      templateRecords.value = []
      hasLoadedTemplates.value = true
      setFeedback(error instanceof Error ? error.message : 'Unable to load assessment templates right now.', 'error')
    },
  )
  stopAssignmentSubscription = subscribeToBusinessAssessmentAssignments(
    text(workspaceOwnerId),
    (records = []) => { assignmentRecords.value = Array.isArray(records) ? records : [] },
    (error) => {
      assignmentRecords.value = []
      setFeedback(error instanceof Error ? error.message : 'Unable to load assessment assignments right now.', 'error')
    },
  )
}, { immediate: true })
onBeforeUnmount(() => {
  stopTemplateSubscription()
  stopAssignmentSubscription()
})

const saveAssessmentTemplate = async () => {
  if (isReadOnly.value) return
  if (!text(props.workspaceOwnerId)) {
    setFeedback('Workspace owner ID is missing, so this template cannot be saved yet.', 'error')
    return
  }
  if (!assessmentDraft.questions.length) {
    setFeedback('Add at least one multiple-choice question before saving this assessment.', 'error')
    return
  }

  const invalidQuestionMessage = assessmentDraft.questions
    .map((question, index) => getQuestionValidationMessage(question, index))
    .find(Boolean)

  if (invalidQuestionMessage) {
    setFeedback(invalidQuestionMessage, 'error')
    return
  }

  isTemplateSaving.value = true
  try {
    const documentId = selectedTemplateId.value || `assessment-template-${Date.now()}`
    await saveBusinessAssessmentTemplateRecord({
      id: documentId,
      workspaceOwnerId: text(props.workspaceOwnerId),
      title: text(assessmentDraft.title) || 'Untitled Assessment',
      description: text(assessmentDraft.description) || 'Assessment description',
      category: text(assessmentDraft.category) || 'General',
      instructions: '',
      passingScorePercent: normalizePercent(assessmentDraft.passingScore, 70),
      questions: toStoredQuestions(assessmentDraft.questions),
    })
    selectedTemplateId.value = documentId
    setFeedback('Assessment template saved.', 'success')
  } catch (error) {
    setFeedback(error instanceof Error ? error.message : 'Unable to save this assessment template right now.', 'error')
  } finally {
    isTemplateSaving.value = false
  }
}

const deleteAssessmentTemplate = async () => {
  if (isReadOnly.value || !selectedTemplateId.value) return

  isTemplateDeleting.value = true
  try {
    await deleteBusinessAssessmentTemplateRecord(selectedTemplateId.value)
    startNewTemplate()
    setFeedback('Assessment template removed.', 'success')
  } catch (error) {
    setFeedback(error instanceof Error ? error.message : 'Unable to delete this assessment template right now.', 'error')
  } finally {
    isTemplateDeleting.value = false
  }
}

const addQuestion = () => {
  assessmentDraft.questions.push({
    id: `question-${Date.now()}`,
    question: '',
    type: 'multiple-choice',
    options: buildMultipleChoiceOptionDrafts(),
    correctOptionIndex: 0,
  })
}
const removeQuestion = (questionId = '') => {
  assessmentDraft.questions = assessmentDraft.questions.filter((question) => question.id !== questionId)
  if (!assessmentDraft.questions.length) addQuestion()
}
const updateQuestionOption = (questionId = '', optionIndex = 0, value = '') => {
  const targetQuestion = assessmentDraft.questions.find((question) => question.id === questionId)
  if (!targetQuestion) return

  const nextOptions = buildMultipleChoiceOptionDrafts(targetQuestion.options)
  nextOptions[optionIndex] = text(value)
  targetQuestion.options = nextOptions
}
const setQuestionCorrectOption = (questionId = '', optionIndex = 0) => {
  const targetQuestion = assessmentDraft.questions.find((question) => question.id === questionId)
  if (!targetQuestion) return
  targetQuestion.correctOptionIndex = optionIndex
}
const selectAssignmentTemplate = (applicationId = '', templateId = '') => {
  const normalizedApplicationId = text(applicationId)
  if (!normalizedApplicationId) return
  const targetRow = assignmentRows.value.find((row) => row.applicationId === normalizedApplicationId)
  if (targetRow?.isSubmissionLocked) return
  assignmentDraftByApplicationId[normalizedApplicationId] = text(templateId)
}

const assignAssessment = async (row = {}) => {
  if (isReadOnly.value) return

  const normalizedApplicationId = text(row?.applicationId || row?.id)
  const selectedTemplate = templateMap.value.get(text(row?.selectedTemplateId))
  if (!text(props.workspaceOwnerId)) {
    setFeedback('Workspace owner ID is missing, so this assessment cannot be assigned yet.', 'error')
    return
  }
  if (row?.isSubmissionLocked) {
    setFeedback('This applicant already submitted the assessment, so it can no longer be reassigned.', 'error')
    return
  }
  if (!normalizedApplicationId || !selectedTemplate) {
    setFeedback('Select an assessment template first before assigning it.', 'error')
    return
  }

  pendingAssignmentRowId.value = normalizedApplicationId
  try {
    await saveBusinessAssessmentAssignmentRecord({
      id: normalizedApplicationId,
      applicationId: normalizedApplicationId,
      applicantId: text(row?.applicantId),
      applicantName: text(row?.applicantName),
      applicantEmail: text(row?.applicantEmail),
      role: text(row?.jobTitle),
      jobId: text(row?.jobId),
      jobTitle: text(row?.jobTitle),
      companyName: text(props.businessName || row?.companyLabel),
      businessName: text(props.businessName || row?.companyLabel),
      selectedTemplateId: text(selectedTemplate.id),
      templateTitle: text(selectedTemplate.title),
      templateDescription: text(selectedTemplate.description),
      templateInstructions: '',
      templateQuestions: toStoredQuestions(selectedTemplate.questions),
      passingScorePercent: normalizePercent(selectedTemplate.passingScore, 70),
      assignmentStatus: 'assigned',
      assessmentStatus: 'assigned',
      assessmentResult: 'pending',
      assessmentScoreValue: 0,
      assessmentScoreLabel: '',
      assignedAt: toIsoNow(),
      workspaceOwnerId: text(props.workspaceOwnerId),
    })
    await updateApplicantJobApplicationStatus(normalizedApplicationId, {
      technicalAssessmentStatus: 'assigned',
      technicalAssessmentResult: '',
      technicalAssessmentScoreValue: 0,
      technicalAssessmentScoreLabel: '',
      technicalAssessmentSubmittedAt: '',
    })
    delete assignmentDraftByApplicationId[normalizedApplicationId]
    setFeedback(`Assessment assigned to ${text(row?.applicantName) || 'the applicant'}.`, 'success')
  } catch (error) {
    setFeedback(error instanceof Error ? error.message : 'Unable to assign this assessment right now.', 'error')
  } finally {
    pendingAssignmentRowId.value = ''
  }
}

const removeAssessment = async (row = {}) => {
  if (isReadOnly.value) return

  const normalizedApplicationId = text(row?.applicationId || row?.id)
  if (!normalizedApplicationId) return
  if (row?.isSubmissionLocked) {
    setFeedback('This applicant already submitted the assessment, so the assignment can no longer be removed.', 'error')
    return
  }

  pendingRemovalRowId.value = normalizedApplicationId
  try {
    await deleteBusinessAssessmentAssignmentRecord(normalizedApplicationId)
    await updateApplicantJobApplicationStatus(normalizedApplicationId, {
      technicalAssessmentStatus: '',
      technicalAssessmentResult: '',
      technicalAssessmentScoreValue: 0,
      technicalAssessmentScoreLabel: '',
      technicalAssessmentSubmittedAt: '',
    })
    delete assignmentDraftByApplicationId[normalizedApplicationId]
    setFeedback(`Assessment removed from ${text(row?.applicantName) || 'the applicant'}.`, 'success')
  } catch (error) {
    setFeedback(error instanceof Error ? error.message : 'Unable to remove this assessment right now.', 'error')
  } finally {
    pendingRemovalRowId.value = ''
  }
}
</script>

<template>
  <section class="business-assessment-management business-assessment-stage-live">
    <div v-if="feedback.message" class="business-assessment-stage-live__feedback" :class="feedback.tone === 'error' ? 'is-error' : 'is-success'">
      <strong>{{ feedback.tone === 'error' ? 'Action Failed' : 'Assessment Updated' }}</strong>
      <span>{{ feedback.message }}</span>
    </div>

    <div class="business-assessment-management__tabs">
      <button type="button" class="business-assessment-management__tab" :class="{ 'is-active': activeTab === 'builder' }" @click="activeTab = 'builder'">Create Assessment</button>
      <button type="button" class="business-assessment-management__tab" :class="{ 'is-active': activeTab === 'assign' }" @click="activeTab = 'assign'">Assign Assessment</button>
      <button type="button" class="business-assessment-management__tab" :class="{ 'is-active': activeTab === 'score' }" @click="activeTab = 'score'">Applicant Score</button>
    </div>

    <section v-if="activeTab === 'builder'" class="business-template-builder">
      <div class="business-template-builder__editor">
        <section class="business-template-builder__hero">
          <div>
            <p class="business-template-builder__eyebrow">Assessment Builder</p>
            <h2>Create and edit screening templates</h2>
            <p>Saved templates now feed the assignment records used by the applicant technical assessment.</p>
          </div>
          <div class="business-template-builder__hero-actions">
            <button type="button" class="business-template-builder__publish business-template-builder__publish--secondary" :disabled="isReadOnly" @click="startNewTemplate">New Template</button>
            <button type="button" class="business-template-builder__publish business-template-builder__publish--danger" :disabled="isReadOnly || !selectedTemplateId || isTemplateDeleting" @click="deleteAssessmentTemplate">Delete</button>
            <button type="button" class="business-template-builder__publish" :disabled="isReadOnly || isTemplateSaving" @click="saveAssessmentTemplate">Save Template</button>
          </div>
        </section>

        <section class="business-template-builder__card business-template-builder__card--header">
          <div class="business-assessment-builder__bar">
            <label class="business-template-builder__select-wrap business-assessment-builder__picker">
              <span>Template Library</span>
              <select :value="selectedTemplateId" @change="syncDraft($event.target.value)">
                <option value="">Select template</option>
                <option v-for="template in templates" :key="template.id" :value="template.id">{{ template.title }}</option>
              </select>
            </label>
            <div class="business-assessment-builder__bar-status">
              <div class="business-assessment-builder__status-card"><strong>{{ templates.length }}</strong><span>Templates</span></div>
            </div>
            <div class="business-assessment-builder__bar-status">
              <div class="business-assessment-builder__status-card"><strong>{{ assessmentDraft.questions.length }}</strong><span>Questions</span></div>
            </div>
          </div>

          <div class="business-template-builder__field-grid">
            <label class="business-template-builder__field"><span>Assessment Title</span><input v-model="assessmentDraft.title" type="text" placeholder="Customer Support Screening"></label>
            <label class="business-template-builder__field"><span>Category</span><input v-model="assessmentDraft.category" type="text" placeholder="Customer Service"></label>
          </div>
          <label class="business-template-builder__field"><span>Description</span><textarea v-model="assessmentDraft.description" placeholder="Describe what this assessment measures."></textarea></label>
          <label class="business-template-builder__field"><span>Passing Score</span><input v-model="assessmentDraft.passingScore" type="number" min="1" max="100"></label>
        </section>

        <section class="business-template-builder__card business-template-builder__toolbar">
          <div class="business-template-builder__toolbar-copy"><strong>Questions</strong><span>Each item is saved as multiple-choice with a correct answer for automatic scoring.</span></div>
          <div class="business-template-builder__toolbar-actions">
            <button type="button" class="business-template-builder__add" :disabled="isReadOnly" @click="addQuestion"><i class="bi bi-plus-lg" aria-hidden="true" />Add Question</button>
          </div>
        </section>

        <section class="business-template-builder__questions">
          <article v-for="(question, index) in assessmentDraft.questions" :key="question.id" class="business-template-builder__card business-template-builder__question">
            <div class="business-template-builder__question-top">
              <div class="business-template-builder__question-meta">
                <span class="business-template-builder__question-order">Question {{ index + 1 }}</span>
                <span class="business-template-builder__question-type">{{ formatQuestionTypeLabel(question.type) }}</span>
              </div>
              <button type="button" class="business-template-builder__remove" :disabled="isReadOnly" @click="removeQuestion(question.id)">Remove</button>
            </div>
            <label class="business-template-builder__field"><span>Question</span><textarea v-model="question.question" placeholder="Write your assessment question here."></textarea></label>
            <div class="business-template-builder__options">
              <div class="business-template-builder__options-head">
                <strong>Answer Choices</strong>
                <span>Select the correct answer below for automatic scoring.</span>
              </div>
              <div class="business-template-builder__options-list">
                <label
                  v-for="(option, optionIndex) in question.options"
                  :key="`${question.id}-option-${optionIndex}`"
                  class="business-template-builder__option-row"
                >
                  <input
                    :checked="question.correctOptionIndex === optionIndex"
                    type="radio"
                    :name="`${question.id}-correct-option`"
                    @change="setQuestionCorrectOption(question.id, optionIndex)"
                  />
                  <span class="business-template-builder__option-marker">Choice {{ optionIndex + 1 }}</span>
                  <input
                    :value="option"
                    type="text"
                    class="business-template-builder__option-input"
                    :placeholder="`Enter choice ${optionIndex + 1}`"
                    @input="updateQuestionOption(question.id, optionIndex, $event.target.value)"
                  />
                </label>
              </div>
              <p class="business-template-builder__options-note">The checked choice will be used as the correct answer during applicant scoring.</p>
            </div>
          </article>
        </section>
      </div>

      <aside class="business-assessment-builder__workspace">
        <section class="business-template-preview__shell">
          <div class="business-template-preview__topbar"><strong>Live Preview</strong><span class="business-template-preview__badge">Assessment</span></div>
          <div class="business-template-preview__header">
            <h3>{{ assessmentDraft.title || 'Untitled Assessment' }}</h3>
            <p>{{ assessmentDraft.description || 'Your assessment description will appear here.' }}</p>
          </div>
          <div class="business-template-preview__details">
            <article class="business-template-preview__detail"><span>Category</span><strong>{{ assessmentDraft.category || 'General' }}</strong></article>
            <article class="business-template-preview__detail"><span>Passing Score</span><strong>{{ normalizePercent(assessmentDraft.passingScore, 70) }}%</strong></article>
            <article class="business-template-preview__detail"><span>Questions</span><strong>{{ assessmentDraft.questions.length }}</strong></article>
          </div>
          <article v-for="(question, index) in assessmentDraft.questions" :key="question.id" class="business-template-preview__question">
            <div class="business-template-preview__question-head"><span>Question {{ index + 1 }}</span><small>{{ formatQuestionTypeLabel(question.type) }}</small></div>
            <p>{{ question.question || 'Question preview will appear here.' }}</p>
            <div class="business-template-preview__choices">
              <label
                v-for="(option, optionIndex) in question.options"
                :key="`${question.id}-preview-option-${optionIndex}`"
                class="business-template-preview__choice"
                :class="{ 'is-correct': question.correctOptionIndex === optionIndex }"
              >
                <input type="radio" :checked="question.correctOptionIndex === optionIndex" disabled>
                <span>{{ option || `Choice ${optionIndex + 1}` }}</span>
                <strong v-if="question.correctOptionIndex === optionIndex">Correct answer</strong>
              </label>
            </div>
          </article>
        </section>
      </aside>
    </section>

    <section v-else-if="activeTab === 'assign'" class="business-assessment-assignments">
      <div class="business-assign-templates__header">
        <div>
          <p class="business-assign-templates__eyebrow">Assessment Assignment</p>
          <h2>Assign templates to applicants</h2>
          <p>Assignments here are now saved to shared records and appear inside the applicant technical assessment screen.</p>
        </div>
        <div class="business-assign-templates__summary"><span>{{ assignmentSummary }}</span></div>
      </div>

      <div class="business-assessment-assignments__toolbar">
        <label class="business-applicant-score__field"><span>Search</span><input v-model.trim="assessmentAssignmentFilters.search" type="text" placeholder="Search applicant or role"></label>
        <label class="business-applicant-score__field"><span>Role</span><select v-model="assessmentAssignmentFilters.roleFilter"><option v-for="option in assignmentRoleOptions" :key="option.value" :value="option.value">{{ option.label }}</option></select></label>
        <div class="business-applicant-score__summary">{{ assignmentSummary }}</div>
      </div>

      <div class="business-assessment-assignments__table-shell">
        <div class="business-assessment-assignments__table">
          <div class="business-assessment-assignments__head">
            <span>Applicant</span>
            <span>Applied Role</span>
            <span>Template</span>
            <span>Assignment</span>
            <span>Action</span>
          </div>

          <div v-if="filteredAssignmentRows.length" class="business-assessment-assignments__body">
            <article v-for="row in filteredAssignmentRows" :key="row.id" class="business-assessment-assignments__row">
              <div class="business-applicant-score__account">
                <div class="business-applicant-score__avatar"><img v-if="row.applicantAvatar" :src="row.applicantAvatar" :alt="`${row.applicantName} avatar`"><span v-else>{{ row.applicantInitials }}</span></div>
                <div class="business-applicant-score__meta"><strong>{{ row.applicantName }}</strong><span>{{ row.applicantEmail }}</span></div>
              </div>
              <div class="business-applicant-score__role"><strong>{{ row.jobTitle }}</strong><span>{{ row.companyLabel }}</span></div>
              <label class="business-assessment-assignments__select">
                <span class="business-assessment-assignments__select-label">Assessment Template</span>
                <select :value="row.selectedTemplateId" :disabled="row.isSubmissionLocked" @change="selectAssignmentTemplate(row.applicationId, $event.target.value)">
                  <option value="">Select template</option>
                  <option v-for="template in templates" :key="template.id" :value="template.id">{{ template.title }}</option>
                </select>
              </label>
              <div class="business-assessment-assignments__status-wrap"><span class="business-assessment-assignments__status" :class="`is-${row.assignmentStatusClass}`">{{ row.assignmentStatusLabel }}</span><small>{{ row.assignmentNote }}</small></div>
              <div class="business-assessment-assignments__actions">
                <button type="button" class="business-template-builder__publish" :disabled="!row.canAssign || pendingAssignmentRowId === row.applicationId" @click="assignAssessment(row)">Assign</button>
                <button v-if="row.selectedTemplateId" type="button" class="business-template-builder__publish business-template-builder__publish--secondary" :disabled="!row.canRemove || pendingRemovalRowId === row.applicationId" @click="removeAssessment(row)">Remove</button>
              </div>
            </article>
          </div>
          <div v-else class="business-applicant-score__empty">No applicants match the current assignment filter.</div>
        </div>
      </div>
    </section>

    <section v-else class="business-applicant-score">
      <div class="business-assign-templates__header">
        <div>
          <p class="business-assign-templates__eyebrow">Assessment Scores</p>
          <h2>Applicant score tracker</h2>
          <p>This view now reads from the same assignment records used by the applicant technical assessment.</p>
        </div>
        <div class="business-assign-templates__summary"><span>{{ filteredScoreRows.length }} applicant<span v-if="filteredScoreRows.length !== 1">s</span> visible</span></div>
      </div>

      <div class="business-applicant-score__toolbar">
        <label class="business-applicant-score__field"><span>Search</span><input v-model.trim="applicantScoreFilters.search" type="text" placeholder="Search applicant or template"></label>
        <label class="business-applicant-score__field"><span>Role</span><select v-model="applicantScoreFilters.roleFilter"><option v-for="option in scoreRoleOptions" :key="option.value" :value="option.value">{{ option.label }}</option></select></label>
        <div class="business-applicant-score__summary">{{ filteredScoreRows.length }} records</div>
      </div>

      <div class="business-applicant-score__table-shell">
        <div class="business-applicant-score__table">
          <div class="business-applicant-score__head">
            <span>Applicant</span>
            <span>Applied Role</span>
            <span>Template</span>
            <span>Score</span>
            <span>Status</span>
          </div>
          <div v-if="filteredScoreRows.length" class="business-applicant-score__body">
            <article v-for="row in filteredScoreRows" :key="row.id" class="business-applicant-score__row">
              <div class="business-applicant-score__account">
                <div class="business-applicant-score__avatar"><img v-if="row.applicantAvatar" :src="row.applicantAvatar" :alt="`${row.applicantName} avatar`"><span v-else>{{ row.applicantInitials }}</span></div>
                <div class="business-applicant-score__meta"><strong>{{ row.applicantName }}</strong><span>{{ row.applicantEmail }}</span></div>
              </div>
              <div class="business-applicant-score__role"><strong>{{ row.jobTitle }}</strong><span>{{ row.resultLabel }}</span></div>
              <div class="business-applicant-score__template"><strong>{{ row.templateName }}</strong><span>Assessment result overview</span></div>
              <span class="business-applicant-score__score-pill" :class="`is-${row.scoreTone}`">{{ row.scoreValue }}</span>
              <span class="business-applicant-score__status" :class="`is-${row.statusClass}`">{{ row.statusLabel }}</span>
            </article>
          </div>
          <div v-else class="business-applicant-score__empty">No applicant scores match the current filter.</div>
        </div>
      </div>
    </section>
  </section>
</template>

<style scoped>
.business-assessment-stage-live,
.business-template-builder,
.business-assessment-assignments,
.business-applicant-score,
.business-assessment-builder__workspace,
.business-template-preview__shell,
.business-assessment-assignments__table-shell,
.business-applicant-score__table-shell {
  min-width: 0;
}

.business-assessment-stage-live {
  display: grid;
  gap: 1rem;
}

.business-assessment-stage-live__feedback {
  display: grid;
  gap: 0.3rem;
  padding: 0.95rem 1rem;
  border: 1px solid rgba(213, 226, 219, 0.92);
  border-radius: 1rem;
  background: rgba(248, 252, 249, 0.94);
  color: #365847;
}

.business-assessment-stage-live__feedback.is-error {
  border-color: rgba(200, 128, 117, 0.4);
  background: rgba(255, 245, 243, 0.96);
  color: #8e4b3f;
}

.business-assessment-builder__workspace {
  position: sticky;
  top: 1rem;
  align-self: start;
}

.business-template-builder__hero,
.business-template-builder__card,
.business-template-preview__shell,
.business-assessment-assignments__toolbar,
.business-assessment-assignments__table-shell,
.business-applicant-score__toolbar,
.business-applicant-score__table-shell,
.business-assessment-stage-live__feedback {
  box-shadow: 0 16px 34px rgba(24, 49, 38, 0.06);
}

.business-template-preview__question p,
.business-applicant-score__meta span,
.business-applicant-score__role span,
.business-applicant-score__template span,
.business-assessment-assignments__status-wrap small {
  overflow-wrap: anywhere;
}

.business-template-builder__options,
.business-template-builder__options-list,
.business-template-preview__choices {
  display: grid;
  gap: 0.75rem;
}

.business-template-builder__options {
  padding-top: 0.5rem;
}

.business-template-builder__options-head {
  display: grid;
  gap: 0.2rem;
}

.business-template-builder__options-head strong {
  color: #264838;
}

.business-template-builder__options-head span,
.business-template-builder__options-note {
  color: #709180;
  font-size: 0.82rem;
}

.business-template-builder__option-row,
.business-template-preview__choice {
  display: grid;
  grid-template-columns: auto auto minmax(0, 1fr) auto;
  align-items: center;
  gap: 0.75rem;
  padding: 0.85rem 0.95rem;
  border: 1px solid rgba(213, 226, 219, 0.92);
  border-radius: 1rem;
  background: rgba(248, 252, 249, 0.82);
}

.business-template-builder__option-row input[type='radio'],
.business-template-preview__choice input[type='radio'] {
  margin: 0;
  accent-color: #2f7a54;
}

.business-template-builder__option-marker {
  min-width: 4.8rem;
  font-size: 0.78rem;
  font-weight: 700;
  color: #587463;
}

.business-template-builder__option-input {
  width: 100%;
  min-width: 0;
}

.business-template-preview__choice {
  grid-template-columns: auto minmax(0, 1fr) auto;
}

.business-template-preview__choice span {
  color: #365847;
}

.business-template-preview__choice strong {
  font-size: 0.72rem;
  color: #2f7a54;
}

.business-template-preview__choice.is-correct {
  border-color: rgba(90, 165, 118, 0.34);
  background: rgba(235, 247, 239, 0.9);
}

.business-assessment-assignments__toolbar,
.business-applicant-score__toolbar {
  display: grid;
  grid-template-columns: minmax(0, 1.45fr) minmax(10rem, 0.8fr) auto;
  gap: 0.8rem;
  padding: 1rem;
  border: 1px solid rgba(213, 226, 219, 0.92);
  border-radius: 1.2rem;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.98) 0%, rgba(247, 251, 249, 0.96) 100%);
}

.business-assessment-assignments__table-shell,
.business-applicant-score__table-shell {
  overflow-x: auto;
}

.business-assessment-assignments__table,
.business-applicant-score__table {
  display: grid;
  min-width: 1120px;
}

.business-assessment-assignments__head,
.business-assessment-assignments__row,
.business-applicant-score__head,
.business-applicant-score__row {
  display: grid;
  align-items: center;
  gap: 1rem;
  padding: 0.95rem 1rem;
}

.business-assessment-assignments__head,
.business-assessment-assignments__row {
  grid-template-columns: minmax(16rem, 1.45fr) minmax(11rem, 1fr) minmax(16rem, 1.2fr) minmax(11rem, 0.8fr) minmax(13rem, 0.95fr);
}

.business-applicant-score__head,
.business-applicant-score__row {
  grid-template-columns: minmax(16rem, 1.45fr) minmax(11rem, 1fr) minmax(14rem, 1.1fr) minmax(10rem, 0.7fr) minmax(12rem, 0.9fr);
}

.business-assessment-assignments__head,
.business-applicant-score__head {
  color: #709180;
  font-size: 0.78rem;
  font-weight: 800;
  border-bottom: 1px solid rgba(221, 231, 225, 0.92);
  background: rgba(248, 252, 249, 0.92);
}

.business-assessment-assignments__row,
.business-applicant-score__row {
  border-bottom: 1px solid rgba(221, 231, 225, 0.72);
}

.business-assessment-assignments__row:last-child,
.business-applicant-score__row:last-child {
  border-bottom: 0;
}

.business-assessment-assignments__select {
  display: grid;
  gap: 0.35rem;
}

.business-assessment-assignments__select-label {
  font-size: 0.72rem;
  font-weight: 700;
  color: #709180;
}

.business-assessment-assignments__status-wrap,
.business-assessment-assignments__actions {
  display: grid;
  gap: 0.45rem;
}

.business-assessment-assignments__actions {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.business-assessment-assignments__status {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 2.5rem;
  padding: 0.55rem 1rem;
  border-radius: 999px;
  background: rgba(244, 236, 215, 0.9);
  color: #b47d12;
  font-weight: 700;
}

.business-assessment-assignments__status.is-assigned,
.business-applicant-score__status.is-submitted,
.business-applicant-score__status.is-completed,
.business-applicant-score__status.is-passed {
  background: rgba(223, 241, 230, 0.95);
  color: #2f7a54;
}

.business-assessment-assignments__status.is-draft,
.business-applicant-score__status.is-in-progress,
.business-applicant-score__status.is-started {
  background: rgba(228, 237, 247, 0.95);
  color: #45698c;
}

.business-assessment-assignments__status.is-not-assigned,
.business-applicant-score__status.is-pending,
.business-applicant-score__status.is-assigned {
  background: rgba(244, 236, 215, 0.9);
  color: #b47d12;
}

.business-applicant-score__status.is-failed,
.business-applicant-score__status.is-cancelled {
  background: rgba(252, 230, 230, 0.95);
  color: #b42318;
}

.business-applicant-score__score-pill.is-top-score,
.business-applicant-score__score-pill.is-qualified {
  background: rgba(223, 241, 230, 0.95);
  color: #2f7a54;
}

.business-applicant-score__score-pill.is-failed-score {
  background: rgba(252, 230, 230, 0.95);
  color: #b42318;
}

.business-applicant-score__score-pill.is-needs-review {
  background: rgba(255, 240, 224, 0.95);
  color: #b36f17;
}

.business-applicant-score__score-pill.is-pending {
  background: rgba(240, 243, 246, 0.95);
  color: #607182;
}

.business-applicant-score__empty {
  padding: 1rem;
}

@media (max-width: 980px) {
  .business-assessment-assignments__toolbar,
  .business-applicant-score__toolbar {
    grid-template-columns: 1fr;
  }
}
</style>
