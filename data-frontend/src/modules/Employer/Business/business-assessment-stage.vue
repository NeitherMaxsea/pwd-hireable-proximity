<script setup>
import { computed, reactive, ref, watch } from 'vue'
import '@/components/businesss.css'

const props = defineProps({
  activeSection: { type: String, default: 'assessment-management' },
  businessJobApplications: { type: Array, default: () => [] },
})

const text = (value = '') => String(value ?? '').trim()
const normalizeValue = (value = '') => text(value).toLowerCase()
const buildInitials = (value = '') =>
  text(value).split(/\s+/).filter(Boolean).slice(0, 2).map((part) => part.charAt(0).toUpperCase()).join('') || 'AP'
const formatStatusLabel = (value = '', fallback = 'Pending') => {
  const normalized = normalizeValue(value)
  if (!normalized) return fallback
  return normalized
    .split(/[_\s-]+/)
    .filter(Boolean)
    .map((part) => part.charAt(0).toUpperCase() + part.slice(1))
    .join(' ')
}
const resolveAssessmentTab = (value = '') => {
  const normalized = normalizeValue(value)
  if (normalized === 'assessment-assignment') return 'assign'
  if (normalized === 'applicant-score') return 'score'
  return 'builder'
}

const activeTab = ref(resolveAssessmentTab(props.activeSection))
watch(
  () => props.activeSection,
  (value) => {
    activeTab.value = resolveAssessmentTab(value)
  },
  { immediate: true },
)

const assessmentTemplates = ref([
  {
    id: 'assessment-template-1',
    title: 'Customer Support Screening',
    description: 'Measure communication, empathy, and problem-solving for support-facing roles.',
    category: 'Customer Service',
    passingScore: 75,
    questions: [
      { id: 'question-1', question: 'How would you calm an upset customer on chat support?', type: 'Short Answer' },
      { id: 'question-2', question: 'Choose the best response for an escalation request.', type: 'Multiple Choice' },
    ],
  },
  {
    id: 'assessment-template-2',
    title: 'Retail Readiness Check',
    description: 'Covers product knowledge, customer handling, and shift responsibility.',
    category: 'Retail',
    passingScore: 70,
    questions: [{ id: 'question-3', question: 'What would you do if a customer changes their order at the register?', type: 'Short Answer' }],
  },
])

const fallbackAssessmentApplicants = [
  {
    id: 'score-row-1',
    applicantName: 'Maria Santos',
    applicantEmail: 'maria.santos@example.com',
    jobTitle: 'Cashier',
    status: 'approved',
    technicalAssessmentResult: 'Passed',
    technicalAssessmentStatus: 'submitted',
    technicalAssessmentScoreValue: 91,
    assessmentTemplateTitle: 'Customer Support Screening',
  },
  {
    id: 'score-row-2',
    applicantName: 'John Dela Cruz',
    applicantEmail: 'john.delacruz@example.com',
    jobTitle: 'Sales Associate',
    status: 'under review',
    technicalAssessmentResult: 'Waiting for submission',
    technicalAssessmentStatus: 'assigned',
    technicalAssessmentScoreValue: 0,
    assessmentTemplateTitle: 'Retail Readiness Check',
  },
]

const selectedTemplateId = ref(assessmentTemplates.value[0]?.id || '')
const assessmentDraft = reactive({
  title: assessmentTemplates.value[0]?.title || '',
  description: assessmentTemplates.value[0]?.description || '',
  category: assessmentTemplates.value[0]?.category || '',
  passingScore: assessmentTemplates.value[0]?.passingScore || 70,
  questions: assessmentTemplates.value[0]?.questions?.map((item) => ({ ...item })) || [],
})

const assessmentAssignmentOverrides = reactive({})
const assessmentAssignmentFilters = reactive({ search: '', roleFilter: 'all' })
const applicantScoreFilters = reactive({ search: '', roleFilter: 'all' })

const syncAssessmentDraft = (templateId = '') => {
  const template = assessmentTemplates.value.find((item) => item.id === templateId) || assessmentTemplates.value[0]
  if (!template) return
  selectedTemplateId.value = template.id
  assessmentDraft.title = template.title
  assessmentDraft.description = template.description
  assessmentDraft.category = template.category
  assessmentDraft.passingScore = template.passingScore
  assessmentDraft.questions = template.questions.map((item) => ({ ...item }))
}

const startNewAssessmentTemplate = () => {
  selectedTemplateId.value = ''
  assessmentDraft.title = ''
  assessmentDraft.description = ''
  assessmentDraft.category = ''
  assessmentDraft.passingScore = 70
  assessmentDraft.questions = [{ id: `question-${Date.now()}`, question: '', type: 'Short Answer' }]
}

const saveAssessmentTemplate = () => {
  const payload = {
    id: selectedTemplateId.value || `assessment-template-${Date.now()}`,
    title: text(assessmentDraft.title) || 'Untitled Assessment',
    description: text(assessmentDraft.description) || 'Assessment description',
    category: text(assessmentDraft.category) || 'General',
    passingScore: Number(assessmentDraft.passingScore) || 70,
    questions: assessmentDraft.questions.map((item, index) => ({
      id: text(item.id) || `question-${index + 1}`,
      question: text(item.question) || `Question ${index + 1}`,
      type: text(item.type) || 'Short Answer',
    })),
  }
  const existingIndex = assessmentTemplates.value.findIndex((item) => item.id === payload.id)
  if (existingIndex >= 0) assessmentTemplates.value.splice(existingIndex, 1, payload)
  else assessmentTemplates.value.unshift(payload)
  syncAssessmentDraft(payload.id)
}

const deleteAssessmentTemplate = () => {
  if (!selectedTemplateId.value) return
  assessmentTemplates.value = assessmentTemplates.value.filter((item) => item.id !== selectedTemplateId.value)
  syncAssessmentDraft(assessmentTemplates.value[0]?.id || '')
}

const addAssessmentTemplateQuestion = () => {
  assessmentDraft.questions.push({ id: `question-${Date.now()}`, question: '', type: 'Short Answer' })
}

const removeAssessmentTemplateQuestion = (questionId = '') => {
  assessmentDraft.questions = assessmentDraft.questions.filter((item) => item.id !== questionId)
}

const assessmentApplicationRows = computed(() =>
  Array.isArray(props.businessJobApplications) && props.businessJobApplications.length
    ? props.businessJobApplications
    : fallbackAssessmentApplicants,
)

const resolveAssessmentTemplateByTitle = (value = '') =>
  assessmentTemplates.value.find((item) => normalizeValue(item.title) === normalizeValue(value)) || null

const getAssessmentAssignmentState = (item = {}, index = 0) => {
  const rowId = text(item?.id || item?.applicationId || item?.application_id || `${item?.applicantEmail || item?.email || 'applicant'}-${index}`)
  const override = assessmentAssignmentOverrides[rowId] || {}
  const persistedTemplateName = text(item?.assessmentTemplateTitle || item?.assessment_template_title)
  const matchedPersistedTemplate = resolveAssessmentTemplateByTitle(persistedTemplateName)
  const selectedTemplateId = text(override.templateId || matchedPersistedTemplate?.id)
  const selectedTemplate = assessmentTemplates.value.find((template) => template.id === selectedTemplateId) || matchedPersistedTemplate
  const templateName = text(override.templateName || selectedTemplate?.title || persistedTemplateName)
  const rawStatus = text(override.status || item?.technicalAssessmentStatus || item?.technical_assessment_status || (templateName ? 'assigned' : 'not assigned'))

  return {
    rowId,
    selectedTemplateId,
    templateName,
    rawStatus,
  }
}

const assessmentAssignmentRows = computed(() =>
  assessmentApplicationRows.value.map((item, index) => {
    const name = text(item?.applicantName || item?.applicant_name || item?.name) || `Applicant ${index + 1}`
    const email = text(item?.applicantEmail || item?.applicant_email || item?.email) || 'No email'
    const jobTitle = text(item?.jobTitle || item?.job_title || item?.role || item?.position) || 'Applicant'
    const assignmentState = getAssessmentAssignmentState(item, index)

    return {
      id: assignmentState.rowId,
      applicantName: name,
      applicantEmail: email,
      applicantInitials: buildInitials(name),
      applicantAvatar: text(item?.applicantAvatar || item?.avatar || item?.avatarUrl),
      jobTitle,
      companyLabel: text(item?.businessName || item?.companyName || 'Business application'),
      selectedTemplateId: assignmentState.selectedTemplateId,
      templateName: assignmentState.templateName || 'No template selected',
      assignmentStatusLabel: formatStatusLabel(assignmentState.rawStatus, 'Not Assigned'),
      assignmentStatusClass: normalizeValue(assignmentState.rawStatus).replace(/[_\s]+/g, '-') || 'not-assigned',
      assignmentNote: assignmentState.templateName
        ? 'Template ready for applicant delivery'
        : 'Select a template before assigning',
    }
  }),
)

const assessmentAssignmentRoleOptions = computed(() => {
  const seen = new Set()
  const options = [{ value: 'all', label: 'All Roles' }]
  assessmentAssignmentRows.value.forEach((row) => {
    const key = normalizeValue(row.jobTitle)
    if (!key || seen.has(key)) return
    seen.add(key)
    options.push({ value: row.jobTitle, label: row.jobTitle })
  })
  return options
})

const filteredAssessmentAssignmentRows = computed(() =>
  assessmentAssignmentRows.value.filter((row) => {
    const query = normalizeValue(assessmentAssignmentFilters.search)
    const roleFilter = normalizeValue(assessmentAssignmentFilters.roleFilter)
    const matchesQuery = !query || [row.applicantName, row.applicantEmail, row.jobTitle, row.templateName].some((value) => normalizeValue(value).includes(query))
    const matchesRole = roleFilter === 'all' || normalizeValue(row.jobTitle) === roleFilter
    return matchesQuery && matchesRole
  }),
)

const assessmentAssignmentSummary = computed(() => {
  const assignedCount = assessmentAssignmentRows.value.filter((row) => row.selectedTemplateId).length
  return `${assignedCount} assignment${assignedCount === 1 ? '' : 's'} ready`
})

const handleAssessmentAssignmentTemplateSelection = (rowId = '', templateId = '') => {
  const normalizedRowId = text(rowId)
  if (!normalizedRowId) return
  const selectedTemplate = assessmentTemplates.value.find((template) => template.id === text(templateId)) || null
  assessmentAssignmentOverrides[normalizedRowId] = {
    ...assessmentAssignmentOverrides[normalizedRowId],
    templateId: text(templateId),
    templateName: text(selectedTemplate?.title),
    status: text(templateId) ? 'draft' : 'not assigned',
  }
}

const requestAssignAssessmentTemplateToApplicant = (rowId = '') => {
  const normalizedRowId = text(rowId)
  if (!normalizedRowId) return
  const currentOverride = assessmentAssignmentOverrides[normalizedRowId] || {}
  if (!text(currentOverride.templateId || currentOverride.templateName)) return
  assessmentAssignmentOverrides[normalizedRowId] = {
    ...currentOverride,
    status: 'assigned',
  }
}

const removeAssignedAssessmentFromApplicant = (rowId = '') => {
  const normalizedRowId = text(rowId)
  if (!normalizedRowId) return
  assessmentAssignmentOverrides[normalizedRowId] = {
    templateId: '',
    templateName: '',
    status: 'not assigned',
  }
}

const canRemoveAssignedAssessment = (row = {}) =>
  Boolean(text(row?.selectedTemplateId || row?.templateName))

const applicantScoreRows = computed(() =>
  assessmentApplicationRows.value.map((item, index) => {
    const assignmentState = getAssessmentAssignmentState(item, index)
    const scoreValue = Number(item?.technicalAssessmentScoreValue ?? item?.technical_assessment_score_value ?? 0) || 0
    const scoreTone = scoreValue >= 85 ? 'top-score' : scoreValue >= 70 ? 'qualified' : scoreValue > 0 ? 'needs-review' : 'pending'
    const status = normalizeValue(item?.technicalAssessmentStatus || item?.technical_assessment_status || assignmentState.rawStatus || 'pending')

    return {
      id: assignmentState.rowId,
      applicantName: text(item?.applicantName || item?.applicant_name || item?.name) || `Applicant ${index + 1}`,
      applicantEmail: text(item?.applicantEmail || item?.applicant_email || item?.email) || 'No email',
      applicantInitials: buildInitials(item?.applicantName || item?.applicant_name || item?.name),
      applicantAvatar: text(item?.applicantAvatar || item?.avatar || item?.avatarUrl),
      jobTitle: text(item?.jobTitle || item?.job_title || item?.role) || 'Applicant',
      templateName: assignmentState.templateName || 'No template assigned',
      scoreValue: scoreValue ? `${scoreValue}%` : text(item?.technicalAssessmentScoreLabel || item?.technical_assessment_score_label) || 'Pending',
      scoreTone,
      statusLabel: formatStatusLabel(status, 'Pending'),
      statusClass: status.replace(/[_\s]+/g, '-') || 'pending',
      resultLabel: text(item?.technicalAssessmentResult || item?.technical_assessment_result)
        || (status === 'assigned' ? 'Assessment assigned' : 'Waiting for submission'),
    }
  }),
)

const applicantScoreRoleOptions = computed(() => {
  const seen = new Set()
  return [{ value: 'all', label: 'All Roles' }].concat(
    applicantScoreRows.value.filter((row) => {
      const key = normalizeValue(row.jobTitle)
      if (!key || seen.has(key)) return false
      seen.add(key)
      return true
    }).map((row) => ({ value: row.jobTitle, label: row.jobTitle })),
  )
})

const filteredApplicantScoreRows = computed(() =>
  applicantScoreRows.value.filter((row) => {
    const query = normalizeValue(applicantScoreFilters.search)
    const roleFilter = normalizeValue(applicantScoreFilters.roleFilter)
    const matchesQuery = !query || [row.applicantName, row.applicantEmail, row.jobTitle, row.templateName].some((value) => normalizeValue(value).includes(query))
    const matchesRole = roleFilter === 'all' || normalizeValue(row.jobTitle) === roleFilter
    return matchesQuery && matchesRole
  }),
)

const applicantScoreSummary = computed(() => `${filteredApplicantScoreRows.value.length} applicant${filteredApplicantScoreRows.value.length === 1 ? '' : 's'} visible`)
</script>

<template>
  <section class="business-assessment-management">
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
            <p>Build role-based assessments for applicants, then keep the final preview visible beside the editor so your form stays clean.</p>
          </div>
          <div class="business-template-builder__hero-actions">
            <button type="button" class="business-template-builder__publish business-template-builder__publish--secondary" @click="startNewAssessmentTemplate">New Template</button>
            <button type="button" class="business-template-builder__publish business-template-builder__publish--danger" :disabled="!selectedTemplateId" @click="deleteAssessmentTemplate">Delete</button>
            <button type="button" class="business-template-builder__publish" @click="saveAssessmentTemplate">Save Template</button>
          </div>
        </section>

        <section class="business-template-builder__card business-template-builder__card--header">
          <div class="business-assessment-builder__bar">
            <div class="business-template-builder__select-wrap business-assessment-builder__picker">
              <span>Template Library</span>
              <select :value="selectedTemplateId" @change="syncAssessmentDraft($event.target.value)">
                <option v-for="template in assessmentTemplates" :key="template.id" :value="template.id">{{ template.title }}</option>
              </select>
            </div>
            <div class="business-assessment-builder__bar-status">
              <div class="business-assessment-builder__status-card"><strong>{{ assessmentTemplates.length }}</strong><span>Templates</span></div>
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
          <div class="business-template-builder__toolbar-copy"><strong>Questions</strong><span>Use concise questions so applicants can answer clearly and reviewers can score faster.</span></div>
          <div class="business-template-builder__toolbar-actions">
            <button type="button" class="business-template-builder__add" @click="addAssessmentTemplateQuestion"><i class="bi bi-plus-lg" aria-hidden="true" />Add Question</button>
          </div>
        </section>

        <section class="business-template-builder__questions">
          <article v-for="(question, index) in assessmentDraft.questions" :key="question.id" class="business-template-builder__card business-template-builder__question">
            <div class="business-template-builder__question-top">
              <div class="business-template-builder__question-meta">
                <span class="business-template-builder__question-order">Question {{ index + 1 }}</span>
                <span class="business-template-builder__question-type">{{ question.type }}</span>
              </div>
              <button type="button" class="business-template-builder__remove" @click="removeAssessmentTemplateQuestion(question.id)">Remove</button>
            </div>
            <label class="business-template-builder__field"><span>Question</span><textarea v-model="question.question" placeholder="Write your assessment question here."></textarea></label>
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
            <article class="business-template-preview__detail"><span>Passing Score</span><strong>{{ assessmentDraft.passingScore }}%</strong></article>
            <article class="business-template-preview__detail"><span>Questions</span><strong>{{ assessmentDraft.questions.length }}</strong></article>
          </div>
          <article v-for="(question, index) in assessmentDraft.questions" :key="question.id" class="business-template-preview__question">
            <div class="business-template-preview__question-head"><span>Question {{ index + 1 }}</span><small>{{ question.type }}</small></div>
            <p>{{ question.question || 'Question preview will appear here.' }}</p>
            <input type="text" placeholder="Applicant answer preview" disabled>
          </article>
          <div class="business-template-preview__footer">
            <button type="button" disabled>Preview Submit Button</button>
            <p class="business-template-preview__footer-copy">Frontend preview only. Publishing logic can be reattached later.</p>
          </div>
        </section>
      </aside>
    </section>

    <section v-else-if="activeTab === 'assign'" class="business-assessment-assignments">
      <div class="business-assign-templates__header">
        <div>
          <p class="business-assign-templates__eyebrow">Assessment Assignment</p>
          <h2>Assign templates to applicants</h2>
          <p>Restore the assignment workspace so recruiters can choose a template, assign it, and track who is ready for submission without leaving the assessment module.</p>
        </div>
        <div class="business-assign-templates__summary"><span>{{ assessmentAssignmentSummary }}</span></div>
      </div>

      <div class="business-assessment-assignments__toolbar">
        <label class="business-applicant-score__field"><span>Search</span><input v-model.trim="assessmentAssignmentFilters.search" type="text" placeholder="Search applicant or role"></label>
        <label class="business-applicant-score__field">
          <span>Role</span>
          <select v-model="assessmentAssignmentFilters.roleFilter">
            <option v-for="option in assessmentAssignmentRoleOptions" :key="option.value" :value="option.value">{{ option.label }}</option>
          </select>
        </label>
        <div class="business-applicant-score__summary">{{ assessmentAssignmentSummary }}</div>
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

          <div v-if="filteredAssessmentAssignmentRows.length" class="business-assessment-assignments__body">
            <article v-for="row in filteredAssessmentAssignmentRows" :key="row.id" class="business-assessment-assignments__row">
              <div class="business-applicant-score__account">
                <div class="business-applicant-score__avatar">
                  <img v-if="row.applicantAvatar" :src="row.applicantAvatar" :alt="`${row.applicantName} avatar`">
                  <span v-else>{{ row.applicantInitials }}</span>
                </div>
                <div class="business-applicant-score__meta">
                  <strong>{{ row.applicantName }}</strong>
                  <span>{{ row.applicantEmail }}</span>
                </div>
              </div>

              <div class="business-applicant-score__role">
                <strong>{{ row.jobTitle }}</strong>
                <span>{{ row.companyLabel }}</span>
              </div>

              <label class="business-assessment-assignments__select">
                <span class="business-assessment-assignments__select-label">Assessment Template</span>
                <select :value="row.selectedTemplateId" @change="handleAssessmentAssignmentTemplateSelection(row.id, $event.target.value)">
                  <option value="">Select template</option>
                  <option v-for="template in assessmentTemplates" :key="template.id" :value="template.id">{{ template.title }}</option>
                </select>
              </label>

              <div class="business-assessment-assignments__status-wrap">
                <span class="business-assessment-assignments__status" :class="`is-${row.assignmentStatusClass}`">{{ row.assignmentStatusLabel }}</span>
                <small>{{ row.assignmentNote }}</small>
              </div>

              <div class="business-assessment-assignments__actions">
                <button type="button" class="business-template-builder__publish" :disabled="!row.selectedTemplateId" @click="requestAssignAssessmentTemplateToApplicant(row.id)">Assign</button>
                <button v-if="canRemoveAssignedAssessment(row)" type="button" class="business-template-builder__publish business-template-builder__publish--secondary" @click="removeAssignedAssessmentFromApplicant(row.id)">Remove</button>
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
          <p>Review completed and assigned assessments in one clean score table so recruiters can spot strong matches quickly.</p>
        </div>
        <div class="business-assign-templates__summary"><span>{{ applicantScoreSummary }}</span></div>
      </div>

      <div class="business-applicant-score__toolbar">
        <label class="business-applicant-score__field"><span>Search</span><input v-model.trim="applicantScoreFilters.search" type="text" placeholder="Search applicant or template"></label>
        <label class="business-applicant-score__field">
          <span>Role</span>
          <select v-model="applicantScoreFilters.roleFilter">
            <option v-for="option in applicantScoreRoleOptions" :key="option.value" :value="option.value">{{ option.label }}</option>
          </select>
        </label>
        <div class="business-applicant-score__summary">{{ applicantScoreSummary }}</div>
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
          <div v-if="filteredApplicantScoreRows.length" class="business-applicant-score__body">
            <article v-for="row in filteredApplicantScoreRows" :key="row.id" class="business-applicant-score__row">
              <div class="business-applicant-score__account">
                <div class="business-applicant-score__avatar">
                  <img v-if="row.applicantAvatar" :src="row.applicantAvatar" :alt="`${row.applicantName} avatar`">
                  <span v-else>{{ row.applicantInitials }}</span>
                </div>
                <div class="business-applicant-score__meta">
                  <strong>{{ row.applicantName }}</strong>
                  <span>{{ row.applicantEmail }}</span>
                </div>
              </div>
              <div class="business-applicant-score__role">
                <strong>{{ row.jobTitle }}</strong>
                <span>{{ row.resultLabel }}</span>
              </div>
              <div class="business-applicant-score__template">
                <strong>{{ row.templateName }}</strong>
                <span>Assessment result overview</span>
              </div>
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
.business-assessment-management,
.business-template-builder,
.business-applicant-score,
.business-assessment-assignments,
.business-assessment-builder__workspace,
.business-template-preview__shell,
.business-applicant-score__table-shell,
.business-assessment-assignments__table-shell {
  min-width: 0;
}

.business-assessment-management {
  display: grid;
  gap: 1rem;
}

.business-assessment-management__tabs,
.business-template-builder__hero,
.business-template-builder__card,
.business-template-preview__shell,
.business-applicant-score__toolbar,
.business-applicant-score__table-shell,
.business-assign-templates__header,
.business-assessment-assignments__toolbar,
.business-assessment-assignments__table-shell {
  box-shadow: 0 16px 34px rgba(24, 49, 38, 0.06);
}

.business-template-builder {
  align-items: start;
}

.business-assessment-builder__workspace {
  position: sticky;
  top: 1rem;
  align-self: start;
}

.business-template-builder__questions,
.business-applicant-score__body,
.business-assessment-assignments__body {
  min-width: 0;
}

.business-template-preview__question p,
.business-applicant-score__meta span,
.business-applicant-score__role span,
.business-applicant-score__template span,
.business-assessment-assignments__status-wrap small {
  overflow-wrap: anywhere;
}

.business-applicant-score__table-shell,
.business-assessment-assignments__table-shell {
  overflow-x: auto;
}

.business-assessment-assignments__toolbar {
  display: grid;
  grid-template-columns: minmax(0, 1.45fr) minmax(10rem, 0.8fr) auto;
  gap: 0.8rem;
  padding: 1rem;
  border: 1px solid rgba(213, 226, 219, 0.92);
  border-radius: 1.2rem;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.98) 0%, rgba(247, 251, 249, 0.96) 100%);
}

.business-assessment-assignments__table {
  display: grid;
  min-width: 1120px;
}

.business-assessment-assignments__head,
.business-assessment-assignments__row {
  display: grid;
  grid-template-columns: minmax(16rem, 1.45fr) minmax(11rem, 1fr) minmax(16rem, 1.2fr) minmax(11rem, 0.8fr) minmax(13rem, 0.95fr);
  align-items: center;
  gap: 1rem;
  padding: 0.95rem 1rem;
}

.business-assessment-assignments__head {
  color: #709180;
  font-size: 0.78rem;
  font-weight: 800;
  border-bottom: 1px solid rgba(221, 231, 225, 0.92);
  background: rgba(248, 252, 249, 0.92);
}

.business-assessment-assignments__row {
  border-bottom: 1px solid rgba(221, 231, 225, 0.72);
}

.business-assessment-assignments__row:last-child {
  border-bottom: 0;
}

.business-assessment-assignments__select {
  display: grid;
  gap: 0.35rem;
  min-width: 0;
}

.business-assessment-assignments__select-label {
  color: #557161;
  font-size: 0.72rem;
  font-weight: 700;
}

.business-assessment-assignments__select select {
  width: 100%;
  min-height: 2.8rem;
  padding: 0.7rem 0.85rem;
  border: 1px solid #d8e3dc;
  border-radius: 0.85rem;
  background: #ffffff;
  color: #183126;
  font: inherit;
  outline: none;
  transition: border-color 0.2s ease, box-shadow 0.2s ease, background-color 0.2s ease;
}

.business-assessment-assignments__select select:focus {
  border-color: #79b293;
  box-shadow: 0 0 0 3px rgba(46, 154, 98, 0.12);
}

.business-assessment-assignments__status-wrap {
  display: grid;
  gap: 0.3rem;
}

.business-assessment-assignments__status-wrap small {
  color: #718096;
  font-size: 0.78rem;
  line-height: 1.45;
}

.business-assessment-assignments__status {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 2rem;
  padding: 0 0.78rem;
  border-radius: 999px;
  font-size: 0.76rem;
  font-weight: 800;
  white-space: nowrap;
  background: rgba(148, 163, 184, 0.16);
  color: #475569;
  justify-self: start;
}

.business-assessment-assignments__status.is-assigned {
  background: rgba(59, 130, 246, 0.14);
  color: #2758a5;
}

.business-assessment-assignments__status.is-submitted {
  background: rgba(34, 197, 94, 0.14);
  color: #1f7a45;
}

.business-assessment-assignments__status.is-draft {
  background: rgba(245, 158, 11, 0.14);
  color: #9a6700;
}

.business-assessment-assignments__status.is-not-assigned {
  background: rgba(148, 163, 184, 0.16);
  color: #475569;
}

.business-assessment-assignments__actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.65rem;
  justify-content: flex-end;
}

.business-applicant-score__head,
.business-applicant-score__row {
  grid-template-columns: minmax(16rem, 1.45fr) minmax(11rem, 1fr) minmax(15rem, 1.15fr) minmax(8rem, 0.7fr) minmax(9rem, 0.8fr);
}

.business-applicant-score__table {
  min-width: 960px;
}

.business-applicant-score__account {
  min-width: 0;
}

.business-applicant-score__avatar img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: inherit;
}

.business-applicant-score__score-pill,
.business-applicant-score__status {
  justify-self: start;
}

@media (max-width: 1200px) {
  .business-assessment-builder__workspace {
    position: static;
  }
}

@media (max-width: 880px) {
  .business-assessment-assignments__toolbar,
  .business-applicant-score__toolbar {
    grid-template-columns: 1fr;
  }

  .business-assessment-assignments__actions {
    justify-content: flex-start;
  }
}
</style>
