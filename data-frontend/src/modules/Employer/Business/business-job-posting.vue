<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import BusinessJobPostingContent from '@/modules/Employer/Business/business_Recruitment.vue'
import { createRecruitmentState } from '@/modules/Employer/Business/business_recruitment_bindings'
import {
  createBusinessJobPost,
  deleteBusinessJobPost,
  subscribeToWorkspaceJobs,
  updateBusinessJobPost,
} from '@/lib/jobs'

defineOptions({
  inheritAttrs: false,
})

const props = defineProps({
  workspaceOwnerId: {
    type: String,
    default: '',
  },
  workspaceOwnerEmail: {
    type: String,
    default: '',
  },
  businessName: {
    type: String,
    default: '',
  },
  businessCategory: {
    type: String,
    default: '',
  },
  businessAvatar: {
    type: String,
    default: '',
  },
  actorId: {
    type: String,
    default: '',
  },
  actorName: {
    type: String,
    default: '',
  },
  isEmployeeWorkspaceMode: {
    type: Boolean,
    default: false,
  },
})

const normalizeText = (value = '') => String(value ?? '').trim()
const buildInitials = (value = '') =>
  normalizeText(value)
    .split(/\s+/)
    .filter(Boolean)
    .slice(0, 2)
    .map((part) => part.charAt(0).toUpperCase())
    .join('') || 'BW'

const JOB_POSTING_BARANGAYS = [
  'Burol I',
  'Burol II',
  'Burol III',
  'Burol Main',
  'Datu Esmael',
  'Emmanuel Bergado I',
  'Emmanuel Bergado II',
  'Fatima I',
  'Fatima II',
  'Fatima III',
  'H-2',
  'Langkaan I',
  'Langkaan II',
  'Luzviminda I',
  'Luzviminda II',
  'Paliparan I',
  'Paliparan II',
  'Paliparan III',
  'Salawag',
  'Salitran I',
  'Salitran II',
  'Salitran III',
  'Salitran IV',
    'Sampaloc I',
  'Sampaloc II',
  'Sampaloc III',
  'Sampaloc IV',
  'Sampaloc V',
  'San Agustin I',
  'San Agustin II',
  'San Agustin III',
  'San Andres I',
  'San Andres II',
  'San Antonio De Padua I',
  'San Antonio De Padua II',
  'San Dionisio',
  'San Esteban',
  'San Francisco I',
  'San Francisco II',
  'San Isidro Labrador I',
  'San Isidro Labrador II',
  'San Jose',
  'San Lorenzo Ruiz I',
  'San Lorenzo Ruiz II',
  'San Luis I',
  'San Luis II',
  'San Manuel I',
  'San Manuel II',
  'San Mateo',
  'San Miguel',
  'San Nicolas I',
  'San Nicolas II',
  'San Roque',
  'San Simon',
  'Santa Cristina I',
  'Santa Cristina II',
  'Santa Cruz I',
  'Santa Cruz II',
  'Santa Fe',
  'Santa Lucia',
  'Santa Maria',
  'Santo Cristo',
  'Santo Nino I',
  'Santo Nino II',
  'Victoria Reyes',
  'Zone I',
  'Zone I-A',
  'Zone II',
  'Zone III',
  'Zone IV',
].map((barangay) => ({
  value: barangay,
  label: barangay,
}))
const JOB_POSTING_EMPLOYMENT_TYPES = ['Full-time', 'Part-time', 'Contract', 'Internship']
const JOB_POSTING_DISABILITY_TYPES = [
  'Physical Impairment',
  'Visual Impairment',
  'Deaf or Hard of Hearing',
  'Hearing Impairment',
  'Speech and Language Impairment',
  'Intellectual Disability',
  'Psychosocial Disability',
  'Learning Disability',
  'Autism Spectrum Disorder',
  'Chronic Illness / Medical Condition',
  'Multiple Disabilities',
].map((type) => ({
  value: type,
  label: type === 'Chronic Illness / Medical Condition'
    ? 'Chronic Illness or Medical Condition'
    : type,
}))
const JOB_POSTING_DISABILITY_TYPES_REQUIRING_SPECIFICATION = new Set([
  'Physical Impairment',
  'Visual Impairment',
  'Deaf or Hard of Hearing',
  'Hearing Impairment',
  'Speech and Language Impairment',
  'Chronic Illness / Medical Condition',
  'Multiple Disabilities',
])
const JOB_POSTING_LANGUAGE_OPTIONS = [
  { value: 'English', label: 'English' },
  { value: 'Filipino', label: 'Filipino' },
  { value: 'English, Filipino', label: 'English and Filipino' },
  { value: 'Filipino, English, Taglish', label: 'Filipino, English, and Taglish' },
]
const JOB_POSTING_MAX_VACANCIES = 500

const authUser = ref({
  id: normalizeText(props.actorId),
  uid: normalizeText(props.actorId),
  name: normalizeText(props.actorName) || 'Business User',
  email: normalizeText(props.workspaceOwnerEmail),
  workspace_owner_id: normalizeText(props.workspaceOwnerId),
  workspace_owner_email: normalizeText(props.workspaceOwnerEmail),
  business_contact_email: normalizeText(props.workspaceOwnerEmail),
})
const profileForm = ref({
  companyName: normalizeText(props.businessName),
  category: normalizeText(props.businessCategory),
  avatar: normalizeText(props.businessAvatar),
})
const businessCategory = computed(() => normalizeText(profileForm.value.category))
const businessName = computed(() => normalizeText(props.businessName) || profileForm.value.companyName || 'Business Workspace')
const businessAvatar = computed(() => normalizeText(profileForm.value.avatar))
const businessInitials = computed(() => buildInitials(businessName.value))
const activeSubscriptionPlan = ref('starter')
const businessJobApplications = ref([])
const businessInterviewSchedules = ref([])
const assessmentAssignmentRecords = ref([])
const jobPostUnlimitedHighlightRef = ref(null)
const paymentToastTimeoutId = ref(null)
const paymentToast = reactive({
  visible: false,
  title: '',
  message: '',
  tone: 'error',
  actions: [],
})

const clearPaymentToastTimeout = () => {
  if (paymentToastTimeoutId.value !== null) {
    clearTimeout(paymentToastTimeoutId.value)
    paymentToastTimeoutId.value = null
  }
}

const closePaymentToast = () => {
  clearPaymentToastTimeout()
  paymentToast.visible = false
  paymentToast.title = ''
  paymentToast.message = ''
  paymentToast.tone = 'error'
  paymentToast.actions = []
}

const getPaymentToastTitle = (message, tone = 'error') => {
  if (tone === 'success') return 'Success'
  if (tone === 'warning') return 'Please Confirm'

  const normalizedMessage = normalizeText(message).toLowerCase()
  if (normalizedMessage.includes('please complete')) return 'Missing information'
  if (normalizedMessage.includes('salary')) return 'Salary format required'
  if (normalizedMessage.includes('vacancies')) return 'Vacancy limit'
  return 'Error'
}

const showPaymentToast = (message, tone = 'error', options = {}) => {
  const normalizedMessage = normalizeText(message)
  if (!normalizedMessage) return

  clearPaymentToastTimeout()
  paymentToast.visible = true
  paymentToast.title = normalizeText(options.title || getPaymentToastTitle(normalizedMessage, tone))
  paymentToast.message = normalizedMessage
  paymentToast.tone = normalizeText(tone || 'error').toLowerCase() || 'error'
  paymentToast.actions = Array.isArray(options.actions)
    ? options.actions
      .filter((action) => normalizeText(action?.label) && typeof action?.onClick === 'function')
      .map((action) => ({
        label: normalizeText(action.label),
        variant: normalizeText(action.variant || 'primary').toLowerCase() || 'primary',
        onClick: action.onClick,
      }))
    : []

  const duration = Number.isFinite(options.duration) ? options.duration : (paymentToast.actions.length ? 0 : 2600)
  if (duration > 0) {
    paymentToastTimeoutId.value = setTimeout(() => {
      closePaymentToast()
    }, duration)
  }
}

const showPaymentConfirmationToast = ({
  title = 'Please Confirm',
  message = '',
  confirmLabel = 'Confirm',
  confirmVariant = 'primary',
  onConfirm = null,
  cancelLabel = 'Cancel',
} = {}) => {
  showPaymentToast(message, 'warning', {
    title,
    duration: 0,
    actions: [
      {
        label: cancelLabel,
        variant: 'secondary',
        onClick: () => {
          closePaymentToast()
        },
      },
      {
        label: confirmLabel,
        variant: confirmVariant,
        onClick: async () => {
          closePaymentToast()
          if (typeof onConfirm === 'function') {
            await onConfirm()
          }
        },
      },
    ],
  })
}

const canEditBusinessModule = () => !props.isEmployeeWorkspaceMode
const getWorkspaceOwnerDirectoryId = () => normalizeText(props.workspaceOwnerId || props.actorId)
const logBusinessStartupSyncIssue = (_message, error, fallback = '') =>
  error instanceof Error ? error.message : fallback
const canUpdateApplicantManagementStatus = () => true
const isPremiumGuideTarget = () => false
const setJobPostUnlimitedHighlightRef = (element) => {
  jobPostUnlimitedHighlightRef.value = element
}

const recruitmentState = createRecruitmentState({
  JOB_POSTING_DISABILITY_TYPES_REQUIRING_SPECIFICATION,
  JOB_POSTING_BARANGAYS,
  JOB_POSTING_DISABILITY_TYPES,
  JOB_POSTING_LANGUAGE_OPTIONS,
  JOB_POSTING_MAX_VACANCIES,
  profileForm,
  businessCategory,
  businessName,
  authUser,
  businessAvatar,
  canEditBusinessModule,
  showPaymentToast,
  showPaymentConfirmationToast,
  createBusinessJobPost,
  updateBusinessJobPost,
  deleteBusinessJobPost,
  updateApplicantJobApplicationStatus: async () => {},
  saveBusinessInterviewSchedule: async () => {},
  saveBusinessAssessmentAssignmentRecord: async () => {},
  subscribeToWorkspaceJobs,
  getWorkspaceOwnerDirectoryId,
  logBusinessStartupSyncIssue,
  businessJobApplications,
  businessInterviewSchedules,
  assessmentAssignmentRecords,
  canUpdateApplicantManagementStatus,
})

const setJobPostingTypeDropdownElement = (element) => {
  recruitmentState.jobPostingTypeDropdownRef.value = element
}
const setJobPostingBarangayDropdownElement = (element) => {
  recruitmentState.jobPostingBarangayDropdownRef.value = element
}
const setJobPostingDisabilityDropdownElement = (element) => {
  recruitmentState.jobPostingDisabilityDropdownRef.value = element
}
const setJobPostingLanguageDropdownElement = (element) => {
  recruitmentState.jobPostingLanguageDropdownRef.value = element
}

const recruitmentBindings = computed(() => ({
  jobPostingTab: recruitmentState.jobPostingTab.value,
  isEditingJobPost: recruitmentState.isEditingJobPost.value,
  postedJobs: recruitmentState.postedJobs.value,
  activeSubscriptionPlan: activeSubscriptionPlan.value,
  canEditBusinessModule,
  toggleJobPostingTab: recruitmentState.toggleJobPostingTab,
  saveJobPost: recruitmentState.saveJobPost,
  jobPostingDraft: recruitmentState.jobPostingDraft.value,
  jobPostingCompanyNameDisplay: recruitmentState.jobPostingCompanyNameDisplay.value,
  jobPostingCategoryLabel: recruitmentState.jobPostingCategoryLabel.value,
  jobPostingTypeLabel: recruitmentState.jobPostingTypeLabel.value,
  isJobPostingDropdownOpen: recruitmentState.isJobPostingDropdownOpen,
  toggleJobPostingDropdown: recruitmentState.toggleJobPostingDropdown,
  selectJobPostingDropdownValue: recruitmentState.selectJobPostingDropdownValue,
  JOB_POSTING_EMPLOYMENT_TYPES,
  JOB_POSTING_BARANGAYS,
  jobPostingBarangayLabel: recruitmentState.jobPostingBarangayLabel.value,
  JOB_POSTING_MAX_VACANCIES,
  JOB_POSTING_DISABILITY_TYPES,
  jobPostingDisabilityLabel: recruitmentState.jobPostingDisabilityLabel.value,
  JOB_POSTING_LANGUAGE_OPTIONS,
  jobPostingLanguageLabel: recruitmentState.jobPostingLanguageLabel.value,
  handleJobPostingFieldChange: recruitmentState.handleJobPostingFieldChange,
  isSavingJobPost: recruitmentState.isSavingJobPost.value,
  cancelJobPostEditing: recruitmentState.cancelJobPostEditing,
  jobPostingPreviewStatusLabel: recruitmentState.jobPostingPreviewStatusLabel.value,
  jobPostingCreatedPreview: recruitmentState.jobPostingCreatedPreview,
  profileForm: profileForm.value,
  businessAvatar: businessAvatar.value,
  businessInitials: businessInitials.value,
  jobPostingQualificationsPreview: recruitmentState.jobPostingQualificationsPreview.value,
  jobPostingResponsibilitiesPreview: recruitmentState.jobPostingResponsibilitiesPreview.value,
  jobPostingSalaryPreview: recruitmentState.jobPostingSalaryPreview.value,
  buildJobPostingDisabilityFitLabel: recruitmentState.buildJobPostingDisabilityFitLabel,
  jobPostHighlights: recruitmentState.jobPostHighlights.value,
  isPremiumGuideTarget,
  setJobPostUnlimitedHighlightRef,
  isPostedJobsLoading: recruitmentState.isPostedJobsLoading.value,
  postedJobsSyncMessage: recruitmentState.postedJobsSyncMessage.value,
  resolvePostedJobStatusClass: recruitmentState.resolvePostedJobStatusClass,
  jobPostPendingAction: recruitmentState.jobPostPendingAction.value,
  openJobPostActionMenuId: recruitmentState.openJobPostActionMenuId.value,
  toggleJobPostActionMenu: recruitmentState.toggleJobPostActionMenu,
  startEditingJobPost: recruitmentState.startEditingJobPost,
  updateJobPostStatus: recruitmentState.updateJobPostStatus,
  isJobPostActionPending: recruitmentState.isJobPostActionPending,
  permanentlyDeleteJobPost: recruitmentState.permanentlyDeleteJobPost,
  setJobPostingTypeDropdownElement,
  setJobPostingBarangayDropdownElement,
  setJobPostingDisabilityDropdownElement,
  setJobPostingLanguageDropdownElement,
}))

watch(
  () => props.businessName,
  (value) => {
    profileForm.value.companyName = normalizeText(value)
  },
  { immediate: true },
)

watch(
  () => props.businessCategory,
  (value) => {
    profileForm.value.category = normalizeText(value)
  },
  { immediate: true },
)

watch(
  () => props.businessAvatar,
  (value) => {
    profileForm.value.avatar = normalizeText(value)
  },
  { immediate: true },
)

watch(
  () => [props.actorId, props.actorName, props.workspaceOwnerId, props.workspaceOwnerEmail],
  ([actorId, actorName, workspaceOwnerId, workspaceOwnerEmail]) => {
    authUser.value = {
      ...authUser.value,
      id: normalizeText(actorId),
      uid: normalizeText(actorId),
      name: normalizeText(actorName) || 'Business User',
      email: normalizeText(workspaceOwnerEmail),
      workspace_owner_id: normalizeText(workspaceOwnerId),
      workspace_owner_email: normalizeText(workspaceOwnerEmail),
      business_contact_email: normalizeText(workspaceOwnerEmail),
    }
  },
  { immediate: true },
)

const handleDocumentClick = (event) => {
  recruitmentState.closeJobPostingDropdownOnOutsideClick(event)
}

onMounted(() => {
  recruitmentState.setJobPostingDefaultTab()
  recruitmentState.startWorkspaceJobPostsSync()

  if (typeof document !== 'undefined') {
    document.addEventListener('click', handleDocumentClick)
  }
})

onBeforeUnmount(() => {
  recruitmentState.stopWorkspaceJobsSync()
  clearPaymentToastTimeout()
  if (typeof document !== 'undefined') {
    document.removeEventListener('click', handleDocumentClick)
  }
})
</script>

<template>
  <BusinessJobPostingContent v-bind="recruitmentBindings" />

  <Teleport to="body">
    <transition name="business-toast">
      <div
        v-if="paymentToast.visible"
        class="business-toast"
        :class="paymentToast.tone"
        role="status"
        aria-live="polite"
      >
        <span class="business-toast__icon" :class="paymentToast.tone" aria-hidden="true">
          <i :class="paymentToast.tone === 'success' ? 'bi bi-check-circle-fill' : paymentToast.tone === 'warning' ? 'bi bi-exclamation-triangle-fill' : 'bi bi-x-circle-fill'" />
        </span>

        <div class="business-toast__copy">
          <strong>{{ paymentToast.title }}</strong>
          <span>{{ paymentToast.message }}</span>
        </div>

        <div v-if="paymentToast.actions.length" class="business-toast__actions">
          <button
            v-for="action in paymentToast.actions"
            :key="`${action.label}-${action.variant}`"
            type="button"
            class="business-toast__action"
            :class="`is-${action.variant}`"
            @click="action.onClick()"
          >
            {{ action.label }}
          </button>
        </div>

        <button
          type="button"
          class="business-toast__close"
          aria-label="Close notification"
          @click="closePaymentToast"
        >
          <i class="bi bi-x-lg" />
        </button>
      </div>
    </transition>
  </Teleport>
</template>

<style scoped>
:deep(.business-job-post) {
  display: grid;
  gap: 1rem;
  min-width: 0;
}

:deep(.business-job-post__create),
:deep(.business-job-post__posted-layout),
:deep(.business-job-post__shell),
:deep(.business-job-post__form-shell),
:deep(.business-job-post__live-preview),
:deep(.business-job-post__posted-list),
:deep(.business-job-post__panel),
:deep(.business-job-post__group),
:deep(.business-job-post__preview-main),
:deep(.business-job-post__preview-meta),
:deep(.business-job-post__preview-section),
:deep(.business-job-post__preview-panel),
:deep(.business-job-post__preview-summary-card) {
  min-width: 0;
}

:deep(.business-job-post__lead) {
  box-shadow: 0 18px 40px rgba(24, 49, 38, 0.07);
}

:deep(.business-job-post__shell) {
  align-items: start;
}

:deep(.business-job-post__live-preview) {
  position: sticky;
  top: 1rem;
}

:deep(.business-job-post__preview-grid),
:deep(.business-job-post__preview-summary),
:deep(.business-job-post__posted-details),
:deep(.business-job-post__highlights) {
  min-width: 0;
}

:deep(.business-job-post__preview-section ul) {
  overflow-wrap: anywhere;
}

:deep(.business-job-post__posted-list) {
  overflow: hidden;
}

:deep(.business-job-post__posted-row) {
  align-items: start;
}

@media (max-width: 1200px) {
  :deep(.business-job-post__live-preview) {
    position: static;
  }
}
</style>
