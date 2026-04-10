<script setup>
import { computed, ref, toRefs, watch } from 'vue'
const props = defineProps([
  'jobPostingTab',
  'isEditingJobPost',
  'postedJobs',
  'activeSubscriptionPlan',
  'canEditBusinessModule',
  'toggleJobPostingTab',
  'saveJobPost',
  'jobPostingDraft',
  'jobPostingCompanyNameDisplay',
  'jobPostingCategoryLabel',
  'jobPostingTypeLabel',
  'isJobPostingDropdownOpen',
  'toggleJobPostingDropdown',
  'selectJobPostingDropdownValue',
  'JOB_POSTING_EMPLOYMENT_TYPES',
  'JOB_POSTING_BARANGAYS',
  'jobPostingBarangayLabel',
  'JOB_POSTING_MAX_VACANCIES',
  'JOB_POSTING_DISABILITY_TYPES',
  'jobPostingDisabilityLabel',
  'JOB_POSTING_LANGUAGE_OPTIONS',
  'jobPostingLanguageLabel',
  'handleJobPostingFieldChange',
  'isSavingJobPost',
  'cancelJobPostEditing',
  'jobPostingPreviewStatusLabel',
  'jobPostingCreatedPreview',
  'profileForm',
  'businessAvatar',
  'businessInitials',
  'jobPostingQualificationsPreview',
  'jobPostingResponsibilitiesPreview',
  'jobPostingSalaryPreview',
  'buildJobPostingDisabilityFitLabel',
  'jobPostHighlights',
  'isPremiumGuideTarget',
  'setJobPostUnlimitedHighlightRef',
  'isPostedJobsLoading',
  'postedJobsSyncMessage',
  'resolvePostedJobStatusClass',
  'jobPostPendingAction',
  'openJobPostActionMenuId',
  'toggleJobPostActionMenu',
  'startEditingJobPost',
  'updateJobPostStatus',
  'isJobPostActionPending',
  'permanentlyDeleteJobPost',
  'setJobPostingTypeDropdownElement',
  'setJobPostingBarangayDropdownElement',
  'setJobPostingDisabilityDropdownElement',
  'setJobPostingLanguageDropdownElement',
])
const {
  jobPostingTab,
  isEditingJobPost,
  postedJobs,
  activeSubscriptionPlan,
  canEditBusinessModule,
  toggleJobPostingTab,
  saveJobPost,
  jobPostingDraft,
  jobPostingCompanyNameDisplay,
  jobPostingCategoryLabel,
  jobPostingTypeLabel,
  isJobPostingDropdownOpen,
  toggleJobPostingDropdown,
  selectJobPostingDropdownValue,
  JOB_POSTING_EMPLOYMENT_TYPES,
  JOB_POSTING_BARANGAYS,
  jobPostingBarangayLabel,
  JOB_POSTING_MAX_VACANCIES,
  JOB_POSTING_DISABILITY_TYPES,
  jobPostingDisabilityLabel,
  JOB_POSTING_LANGUAGE_OPTIONS,
  jobPostingLanguageLabel,
  handleJobPostingFieldChange,
  isSavingJobPost,
  cancelJobPostEditing,
  jobPostingPreviewStatusLabel,
  jobPostingCreatedPreview,
  profileForm,
  businessAvatar,
  businessInitials,
  jobPostingQualificationsPreview,
  jobPostingResponsibilitiesPreview,
  jobPostingSalaryPreview,
  buildJobPostingDisabilityFitLabel,
  jobPostHighlights,
  isPremiumGuideTarget,
  setJobPostUnlimitedHighlightRef,
  isPostedJobsLoading,
  postedJobsSyncMessage,
  resolvePostedJobStatusClass,
  jobPostPendingAction,
  openJobPostActionMenuId,
  toggleJobPostActionMenu,
  startEditingJobPost,
  updateJobPostStatus,
  isJobPostActionPending,
  permanentlyDeleteJobPost,
  setJobPostingTypeDropdownElement,
  setJobPostingBarangayDropdownElement,
  setJobPostingDisabilityDropdownElement,
  setJobPostingLanguageDropdownElement,
} = toRefs(props)

const isDisabilityModalOpen = ref(false)
const disabilityModalSelection = ref([])
const selectedDisabilityValues = computed(() =>
  String(jobPostingDraft.value.disabilityType || '')
    .split(',')
    .map((item) => item.trim())
    .filter(Boolean),
)

watch(
  selectedDisabilityValues,
  (value) => {
    disabilityModalSelection.value = [...value]
  },
  { immediate: true },
)

const openDisabilityModal = () => {
  disabilityModalSelection.value = [...selectedDisabilityValues.value]
  isDisabilityModalOpen.value = true
}

const closeDisabilityModal = () => {
  isDisabilityModalOpen.value = false
}

const toggleDisabilityModalOption = (value) => {
  const normalizedValue = String(value || '').trim()
  if (!normalizedValue) return

  disabilityModalSelection.value = disabilityModalSelection.value.includes(normalizedValue)
    ? disabilityModalSelection.value.filter((entry) => entry !== normalizedValue)
    : [...disabilityModalSelection.value, normalizedValue]
}

const applyDisabilityModalSelection = () => {
  handleJobPostingFieldChange.value('disabilityType', disabilityModalSelection.value)
  closeDisabilityModal()
}
</script>

<template>
<section class="business-job-post">
              <div class="business-job-post__lead">
                <div class="business-job-post__copy">
                  <p class="business-job-post__eyebrow">Job Posting</p>
                  <h2>{{ jobPostingTab === 'create' ? (isEditingJobPost ? 'Edit job post' : 'Create a new job post') : 'Posted Jobs' }}</h2>
                  <p>
                    {{
                      jobPostingTab === 'create'
                        ? isEditingJobPost
                          ? 'Update the job details below, then save your changes to refresh the live posting.'
                          : 'Fill in the job details below to prepare a complete posting draft for your business workspace.'
                        : 'Review, edit, close, reopen, or permanently delete the job posts saved in Firebase for this business workspace.'
                    }}
                  </p>
                  <div class="business-job-post__lead-meta">
                    <span class="business-job-post__lead-chip">
                      <i class="bi bi-briefcase" aria-hidden="true" />
                      {{ postedJobs.length }} saved posts
                    </span>
                    <span class="business-job-post__lead-chip">
                      <i class="bi bi-stars" aria-hidden="true" />
                      {{ activeSubscriptionPlan === 'premium' ? 'Premium workspace' : 'Starter workspace' }}
                    </span>
                  </div>
                </div>
                <div v-if="canEditBusinessModule('job-posting')" class="business-job-post__lead-actions">
                  <button
                    type="button"
                    class="business-job-post__button business-job-post__button--ghost"
                    @click="toggleJobPostingTab"
                  >
                    <i :class="jobPostingTab === 'create' ? 'bi bi-collection' : 'bi bi-plus-circle'" aria-hidden="true" />
                    {{ jobPostingTab === 'create' ? 'Posted Jobs' : 'Create Job Post' }}
                  </button>
                </div>
              </div>

              <div v-if="jobPostingTab === 'create'" class="business-job-post__create">
                <div class="business-job-post__shell">
                  <form class="business-job-post__form-shell" @submit.prevent="saveJobPost">
                    <fieldset class="business-job-post__fieldset" :disabled="!canEditBusinessModule('job-posting')">
                      <div class="business-job-post__section-head">
                        <div>
                          <p class="business-job-post__tips-label">Job Details</p>
                          <strong>{{ isEditingJobPost ? 'Refine this posting before republishing' : 'Build a complete post with the right details' }}</strong>
                        </div>
                        <span class="business-job-post__panel-chip">{{ isEditingJobPost ? 'Editing mode' : 'Draft mode' }}</span>
                      </div>

                      <div class="business-job-post__form-groups">
                        <section class="business-job-post__group">
                          <div class="business-job-post__group-head">
                            <div>
                              <p class="business-job-post__group-kicker">Core Details</p>
                              <strong>Set the main role information first</strong>
                            </div>
                            <span class="business-job-post__group-note">This appears at the top of the listing</span>
                          </div>

                          <div class="business-job-post__grid business-job-post__grid--two">
                            <label class="business-job-post__field">
                              <span>Job Title</span>
                              <input
                                v-model="jobPostingDraft.title"
                                type="text"
                                placeholder="Data Encoder"
                              />
                            </label>

                            <label class="business-job-post__field">
                              <span>Company Name</span>
                              <input
                                :value="jobPostingCompanyNameDisplay"
                                type="text"
                                readonly
                                aria-readonly="true"
                              />
                            </label>
                          </div>

                          <label class="business-job-post__field business-job-post__field--wide">
                            <span>Description</span>
                            <textarea
                              v-model="jobPostingDraft.description"
                              rows="4"
                              placeholder="Describe the role..."
                            ></textarea>
                          </label>

                          <div class="business-job-post__grid business-job-post__grid--two">
                            <label class="business-job-post__field">
                              <span>Category</span>
                              <input
                                :value="jobPostingCategoryLabel"
                                type="text"
                                readonly
                                aria-readonly="true"
                                placeholder="Set business category in profile"
                              />
                              <small class="business-job-post__field-help">Auto-synced from your business profile category.</small>
                            </label>

                            <label class="business-job-post__field">
                              <span>Type</span>
                              <div
                                :ref="setJobPostingTypeDropdownElement"
                                class="business-job-post__select-wrap"
                                :class="{ 'is-open': isJobPostingDropdownOpen('type') }"
                              >
                                <button
                                  type="button"
                                  class="business-job-post__select-trigger"
                                  :class="{ 'is-filled': jobPostingDraft.type }"
                                  :aria-expanded="isJobPostingDropdownOpen('type') ? 'true' : 'false'"
                                  @click="toggleJobPostingDropdown('type')"
                                >
                                  <span>{{ jobPostingTypeLabel }}</span>
                                  <i class="bi bi-chevron-down business-job-post__select-icon" aria-hidden="true" />
                                </button>

                                <transition name="business-job-post__dropdown">
                                  <div v-if="isJobPostingDropdownOpen('type')" class="business-job-post__select-menu">
                                    <button
                                      v-for="type in JOB_POSTING_EMPLOYMENT_TYPES"
                                      :key="type"
                                      type="button"
                                      class="business-job-post__select-option"
                                      :class="{ 'is-active': jobPostingDraft.type === type }"
                                      @click="selectJobPostingDropdownValue('type', type)"
                                    >
                                      <span class="business-job-post__select-option-mark" aria-hidden="true" />
                                      <span>{{ type }}</span>
                                    </button>
                                  </div>
                                </transition>
                              </div>
                            </label>
                          </div>
                        </section>

                        <section class="business-job-post__group">
                          <div class="business-job-post__group-head">
                            <div>
                              <p class="business-job-post__group-kicker">Job Details</p>
                              <strong>Organize the work setup and salary</strong>
                            </div>
                            <span class="business-job-post__group-note">Keep these details clear for applicants</span>
                          </div>

                          <div class="business-job-post__grid business-job-post__grid--three">
                            <label class="business-job-post__field">
                              <span>Location</span>
                              <input :value="jobPostingDraft.location" type="text" readonly aria-readonly="true" />
                            </label>

                            <label class="business-job-post__field">
                              <span>Barangay (Dasmarinas)</span>
                              <div
                                :ref="setJobPostingBarangayDropdownElement"
                                class="business-job-post__select-wrap"
                                :class="{ 'is-open': isJobPostingDropdownOpen('barangay') }"
                              >
                                <button
                                  type="button"
                                  class="business-job-post__select-trigger"
                                  :class="{ 'is-filled': jobPostingDraft.barangay }"
                                  :aria-expanded="isJobPostingDropdownOpen('barangay') ? 'true' : 'false'"
                                  @click="toggleJobPostingDropdown('barangay')"
                                >
                                  <span>{{ jobPostingBarangayLabel }}</span>
                                  <i class="bi bi-chevron-down business-job-post__select-icon" aria-hidden="true" />
                                </button>

                                <transition name="business-job-post__dropdown">
                                  <div v-if="isJobPostingDropdownOpen('barangay')" class="business-job-post__select-menu business-job-post__select-menu--scroll">
                                    <button
                                      v-for="barangay in JOB_POSTING_BARANGAYS"
                                      :key="barangay.value"
                                      type="button"
                                      class="business-job-post__select-option"
                                      :class="{ 'is-active': jobPostingDraft.barangay === barangay.value }"
                                      @click="selectJobPostingDropdownValue('barangay', barangay.value)"
                                    >
                                      <span class="business-job-post__select-option-mark" aria-hidden="true" />
                                      <span>{{ barangay.label }}</span>
                                    </button>
                                  </div>
                                </transition>
                              </div>
                            </label>

                            <label class="business-job-post__field">
                              <span>Vacancies</span>
                              <input
                                v-model="jobPostingDraft.vacancies"
                                type="number"
                                min="1"
                                :max="JOB_POSTING_MAX_VACANCIES"
                                placeholder="Enter number of vacancies"
                              />
                            </label>
                          </div>

                          <label class="business-job-post__field business-job-post__field--wide">
                            <span>Salary Range</span>
                            <div class="business-job-post__range">
                              <input
                                :value="jobPostingDraft.salaryRange"
                                type="text"
                                inputmode="text"
                                placeholder="Example: PHP 15,000 - PHP 20,000"
                                @input="handleJobPostingFieldChange('salaryRange', $event.target.value)"
                              />
                            </div>
                          </label>
                        </section>

                        <section class="business-job-post__group">
                          <div class="business-job-post__group-head">
                            <div>
                              <p class="business-job-post__group-kicker">Applicant Preferences</p>
                              <strong>Match the posting with the right applicants</strong>
                            </div>
                            <span class="business-job-post__group-note">Use only the filters you really need</span>
                          </div>

                          <div class="business-job-post__grid business-job-post__grid--three">
                            <label class="business-job-post__field">
                              <span>Disability Type</span>
                              <button
                                type="button"
                                class="business-job-post__select-trigger"
                                :class="{ 'is-filled': jobPostingDraft.disabilityType }"
                                @click="openDisabilityModal"
                              >
                                <span>{{ jobPostingDisabilityLabel }}</span>
                                <i class="bi bi-ui-checks-grid business-job-post__select-icon" aria-hidden="true" />
                              </button>
                              <small class="business-job-post__field-help">Choose one or more disability types from the modal.</small>
                            </label>

                            <label class="business-job-post__field">
                              <span>Preferred Age</span>
                              <input
                                v-model="jobPostingDraft.preferredAgeRange"
                                type="text"
                                placeholder="Enter preferred age"
                              />
                            </label>

                            <label class="business-job-post__field">
                              <span>Language</span>
                              <div
                                :ref="setJobPostingLanguageDropdownElement"
                                class="business-job-post__select-wrap"
                                :class="{ 'is-open': isJobPostingDropdownOpen('language') }"
                              >
                                <button
                                  type="button"
                                  class="business-job-post__select-trigger"
                                  :class="{ 'is-filled': jobPostingDraft.language }"
                                  :aria-expanded="isJobPostingDropdownOpen('language') ? 'true' : 'false'"
                                  @click="toggleJobPostingDropdown('language')"
                                >
                                  <span>{{ jobPostingLanguageLabel }}</span>
                                  <i class="bi bi-chevron-down business-job-post__select-icon" aria-hidden="true" />
                                </button>

                                <transition name="business-job-post__dropdown">
                                  <div v-if="isJobPostingDropdownOpen('language')" class="business-job-post__select-menu">
                                    <button
                                      v-for="language in JOB_POSTING_LANGUAGE_OPTIONS"
                                      :key="language.value"
                                      type="button"
                                      class="business-job-post__select-option"
                                      :class="{ 'is-active': jobPostingDraft.language === language.value }"
                                      @click="selectJobPostingDropdownValue('language', language.value)"
                                    >
                                      <span class="business-job-post__select-option-mark" aria-hidden="true" />
                                      <span>{{ language.label }}</span>
                                    </button>
                                  </div>
                                </transition>
                              </div>
                            </label>
                          </div>

                        </section>

                        <section class="business-job-post__group">
                          <div class="business-job-post__group-head">
                            <div>
                              <p class="business-job-post__group-kicker">Requirements</p>
                              <strong>List the key qualifications and responsibilities</strong>
                            </div>
                            <span class="business-job-post__group-note">One line per item works best</span>
                          </div>

                          <div class="business-job-post__grid business-job-post__grid--two">
                            <label class="business-job-post__field">
                              <span>Qualifications</span>
                              <textarea
                                v-model="jobPostingDraft.qualifications"
                                rows="5"
                                placeholder="Basic computer literacy&#10;Attention to detail"
                              ></textarea>
                            </label>

                            <label class="business-job-post__field">
                              <span>Responsibilities</span>
                              <textarea
                                v-model="jobPostingDraft.responsibilities"
                                rows="5"
                                placeholder="Encode and verify records&#10;Coordinate with admin team"
                              ></textarea>
                            </label>
                          </div>
                        </section>
                      </div>

                      <div class="business-job-post__actions">
                        <button type="submit" class="business-job-post__save" :disabled="isSavingJobPost">
                          <i :class="isSavingJobPost ? 'bi bi-arrow-repeat' : (isEditingJobPost ? 'bi bi-check2-circle' : 'bi bi-send')" aria-hidden="true" />
                          {{
                            isSavingJobPost
                              ? isEditingJobPost
                                ? 'Saving Changes...'
                                : 'Publishing...'
                              : isEditingJobPost
                                ? 'Save Changes'
                                : 'Post Job'
                          }}
                        </button>
                        <button
                          v-if="isEditingJobPost"
                          type="button"
                          class="business-job-post__secondary"
                          :disabled="isSavingJobPost"
                          @click="cancelJobPostEditing"
                        >
                          <i class="bi bi-x-circle" aria-hidden="true" />
                          Cancel Edit
                        </button>
                      </div>
                    </fieldset>
                  </form>

                  <aside class="business-job-post__live-preview" aria-label="Live job preview">
                    <div class="business-job-post__section-head business-job-post__section-head--preview">
                      <div>
                        <p class="business-job-post__tips-label">Live Preview</p>
                        <strong>See how applicants will read this post</strong>
                      </div>
                    </div>

                    <div class="business-job-post__preview-head">
                      <span class="business-job-post__preview-status">{{ jobPostingPreviewStatusLabel }}</span>
                      <span class="business-job-post__preview-date">
                        <i class="bi bi-clock" aria-hidden="true" />
                        {{ jobPostingCreatedPreview }}
                      </span>
                    </div>

                    <div class="business-job-post__preview-main">
                      <div class="business-job-post__preview-logo" aria-hidden="true">
                        <img
                          v-if="profileForm.avatar || businessAvatar"
                          :src="profileForm.avatar || businessAvatar"
                          alt=""
                          class="business-job-post__preview-logo-image"
                        />
                        <template v-else>{{ businessInitials }}</template>
                      </div>

                      <div class="business-job-post__preview-copy-wrap">
                        <h3>{{ jobPostingDraft.title.trim() || 'Job title preview' }}</h3>
                        <p class="business-job-post__preview-company">
                          <i class="bi bi-building" aria-hidden="true" />
                          {{ jobPostingCompanyNameDisplay || 'Company name' }}
                        </p>
                        <div class="business-job-post__preview-rating" aria-label="Preview rating">
                          <span class="business-job-post__preview-stars" aria-hidden="true">
                            <i v-for="index in 5" :key="`job-post-star-${index}`" class="bi bi-star" />
                          </span>
                          <strong>0.0 / 5</strong>
                          <span>(0 reviews)</span>
                        </div>
                      </div>
                    </div>

                    <div class="business-job-post__preview-meta">
                      <span>
                        <i class="bi bi-geo-alt" aria-hidden="true" />
                        {{ jobPostingDraft.barangay ? `Dasmarinas City, ${jobPostingDraft.barangay}` : 'Select barangay' }}
                      </span>
                      <span>
                        <i class="bi bi-briefcase" aria-hidden="true" />
                        {{ jobPostingCategoryLabel }}
                      </span>
                      <span>
                        <i class="bi bi-clock" aria-hidden="true" />
                        {{ jobPostingDraft.type || 'Select type' }}
                      </span>
                      <span>
                        <i class="bi bi-people" aria-hidden="true" />
                        {{ jobPostingDraft.vacancies ? `${jobPostingDraft.vacancies} Vacancies` : 'Select vacancies' }}
                      </span>
                    </div>

                    <div class="business-job-post__preview-section">
                      <h4>
                        <i class="bi bi-file-text" aria-hidden="true" />
                        Job Description
                      </h4>
                      <p>
                        {{
                          jobPostingDraft.description.trim()
                            || 'Describe the role and what the applicant will do in this position.'
                        }}
                      </p>
                    </div>

                    <div class="business-job-post__preview-panel">
                      <h4>
                        <i class="bi bi-translate" aria-hidden="true" />
                        Language and Age Preference
                      </h4>
                      <p>
                        <strong>Languages:</strong>
                        {{ ` ${jobPostingDraft.language || 'Select language'}` }}
                      </p>
                      <p>
                        <strong>Preferred Age:</strong>
                        {{ ` ${jobPostingDraft.preferredAgeRange || 'Enter preferred age'}` }}
                      </p>
                    </div>

                    <div class="business-job-post__preview-grid">
                      <div class="business-job-post__preview-section">
                        <h4>
                          <i class="bi bi-check2-circle" aria-hidden="true" />
                          Qualifications
                        </h4>
                        <ul>
                          <li
                            v-for="(item, index) in (jobPostingQualificationsPreview.length
                              ? jobPostingQualificationsPreview
                              : ['Add qualifications to preview them here.'])"
                            :key="`job-post-qualification-${index}`"
                          >
                            {{ item }}
                          </li>
                        </ul>
                      </div>

                      <div class="business-job-post__preview-section">
                        <h4>
                          <i class="bi bi-list-task" aria-hidden="true" />
                          Responsibilities
                        </h4>
                        <ul>
                          <li
                            v-for="(item, index) in (jobPostingResponsibilitiesPreview.length
                              ? jobPostingResponsibilitiesPreview
                              : ['Add responsibilities to preview them here.'])"
                            :key="`job-post-responsibility-${index}`"
                          >
                            {{ item }}
                          </li>
                        </ul>
                      </div>
                    </div>

                    <div class="business-job-post__preview-summary">
                      <div class="business-job-post__preview-summary-card">
                        <span class="business-job-post__preview-summary-icon" aria-hidden="true">
                          <i class="bi bi-cash-stack" />
                        </span>
                        <span>Salary Range</span>
                        <strong>{{ jobPostingSalaryPreview }}</strong>
                      </div>

                      <div class="business-job-post__preview-summary-card">
                        <span class="business-job-post__preview-summary-icon" aria-hidden="true">
                          <i class="bi bi-universal-access-circle" />
                        </span>
                        <span>Suitable For</span>
                        <strong>
                          {{
                            buildJobPostingDisabilityFitLabel(jobPostingDraft.disabilityType) || 'Select disability type'
                          }}
                        </strong>
                      </div>
                    </div>
                  </aside>
                </div>
              </div>

              <div v-else class="business-job-post__posted-layout">
                <div class="business-job-post__highlights">
                  <article
                    v-for="item in jobPostHighlights"
                    :key="item.label"
                    class="business-job-post__highlight"
                    :class="{ 'business-guide-target': isPremiumGuideTarget('job-post-unlimited') && item.value === 'Unlimited' }"
                    :ref="item.value === 'Unlimited' ? setJobPostUnlimitedHighlightRef : undefined"
                  >
                    <span>{{ item.label }}</span>
                    <strong>{{ item.value }}</strong>
                  </article>
                </div>

                <article class="business-job-post__panel business-job-post__panel--posted">
                  <div class="business-job-post__panel-head">
                    <div>
                      <p class="business-job-post__tips-label">Posted Jobs</p>
                      <strong>Manage active and closed job posts from your business workspace</strong>
                    </div>
                    <span class="business-job-post__panel-chip">{{ postedJobs.length }} total</span>
                  </div>

                  <div class="business-job-post__posted-list">
                    <p v-if="isPostedJobsLoading" class="business-job-post__posted-empty">
                      Loading posted jobs...
                    </p>
                    <p v-else-if="postedJobsSyncMessage" class="business-job-post__posted-empty">
                      {{ postedJobsSyncMessage }}
                    </p>
                    <p v-else-if="!postedJobs.length" class="business-job-post__posted-empty">
                      No job posts yet. Publish your first opening and it will appear here and on the landing page.
                    </p>
                    <article v-for="job in postedJobs" v-else :key="job.id" class="business-job-post__posted-row">
                      <div class="business-job-post__posted-main">
                        <div class="business-job-post__posted-top">
                          <strong>{{ job.title }}</strong>
                          <span class="business-job-post__status" :class="resolvePostedJobStatusClass(job.status)">{{ job.status }}</span>
                        </div>
                        <p>{{ job.category }} | {{ job.location }} | {{ job.workType }}</p>
                        <small>{{ job.id }} | {{ job.updated }}</small>
                      </div>

                      <div class="business-job-post__posted-metric">
                        <span>Vacancies</span>
                        <strong>{{ job.vacancies }}</strong>
                      </div>

                      <div class="business-job-post__posted-side">
                        <span>Salary</span>
                        <strong>{{ job.salary }}</strong>
                      </div>

                      <div v-if="canEditBusinessModule('job-posting')" class="business-job-post__posted-actions">
                        <div class="business-job-post__menu">
                          <button
                            type="button"
                            class="business-job-post__menu-trigger"
                            aria-label="Open job post actions"
                            :disabled="Boolean(jobPostPendingAction.jobId)"
                            @click="toggleJobPostActionMenu(job.id)"
                          >
                            <i class="bi bi-three-dots-vertical" aria-hidden="true" />
                          </button>

                          <div v-if="openJobPostActionMenuId === job.id" class="business-job-post__menu-panel">
                            <button
                              type="button"
                              class="business-job-post__menu-action"
                              :disabled="Boolean(jobPostPendingAction.jobId)"
                              @click="startEditingJobPost(job.id)"
                            >
                              <i class="bi bi-pencil-square" aria-hidden="true" />
                              Edit
                            </button>
                            <button
                              type="button"
                              class="business-job-post__menu-action"
                              :disabled="Boolean(jobPostPendingAction.jobId)"
                              @click="updateJobPostStatus(job.id, String(job.status || '').trim().toLowerCase() === 'closed' ? 'open' : 'closed')"
                            >
                              <i :class="String(job.status || '').trim().toLowerCase() === 'closed' ? 'bi bi-unlock' : 'bi bi-pause-circle'" aria-hidden="true" />
                              {{
                                isJobPostActionPending(job.id, String(job.status || '').trim().toLowerCase() === 'closed' ? 'open' : 'close')
                                  ? String(job.status || '').trim().toLowerCase() === 'closed'
                                    ? 'Opening...'
                                    : 'Closing...'
                                  : String(job.status || '').trim().toLowerCase() === 'closed'
                                    ? 'Open'
                                    : 'Close'
                              }}
                            </button>
                            <button
                              type="button"
                              class="business-job-post__menu-action is-danger"
                              :disabled="Boolean(jobPostPendingAction.jobId)"
                              @click="permanentlyDeleteJobPost(job.id)"
                            >
                              <i class="bi bi-trash3" aria-hidden="true" />
                              {{ isJobPostActionPending(job.id, 'delete') ? 'Deleting...' : 'Delete' }}
                            </button>
                          </div>
                        </div>
                      </div>
                    </article>
                  </div>
                </article>

              </div>
            </section>
  <Teleport to="body">
    <transition name="business-toast">
      <div
        v-if="isDisabilityModalOpen"
        class="business-job-post__modal-backdrop business-job-post__modal-backdrop--disability"
        role="dialog"
        aria-modal="true"
        aria-labelledby="job-post-disability-modal-title"
        @click.self="closeDisabilityModal"
      >
        <div class="business-job-post__modal business-job-post__modal--disability">
          <div class="business-job-post__modal-head">
            <div>
              <p class="business-job-post__tips-label">Applicant Preferences</p>
              <strong id="job-post-disability-modal-title">Select disability types</strong>
              <p class="business-job-post__modal-copy">
                Choose one or more disability groups that match this posting.
              </p>
            </div>
            <button type="button" class="business-job-post__modal-close" aria-label="Close disability selector" @click="closeDisabilityModal">
              <i class="bi bi-x-lg" />
            </button>
          </div>

          <div class="business-job-post__modal-body">
            <div class="business-job-post__checklist">
            <label
              v-for="type in JOB_POSTING_DISABILITY_TYPES"
              :key="type.value"
              class="business-job-post__check-option"
              :class="{ 'is-active': disabilityModalSelection.includes(type.value) }"
            >
              <input
                type="checkbox"
                :checked="disabilityModalSelection.includes(type.value)"
                @change="toggleDisabilityModalOption(type.value)"
              />
              <span>{{ type.label }}</span>
            </label>
          </div>
          </div>

          <div class="business-job-post__modal-foot">
            <span class="business-job-post__modal-selection-count">
              {{ disabilityModalSelection.length }} selected
            </span>
            <div class="business-job-post__modal-actions">
              <button type="button" class="business-job-post__secondary" @click="closeDisabilityModal">Cancel</button>
              <button type="button" class="business-job-post__save" @click="applyDisabilityModalSelection">Apply Selection</button>
            </div>
          </div>
        </div>
      </div>
    </transition>
  </Teleport>
</template>

<style scoped>
.business-job-post__modal-backdrop--disability {
  z-index: 2600;
}

.business-job-post__modal--disability {
  width: min(100%, 36rem);
  border-radius: 0;
  border: 1px solid rgba(142, 164, 153, 0.42);
  background: #fdfefd;
  box-shadow: 0 26px 70px rgba(10, 24, 17, 0.22);
}

.business-job-post__modal--disability .business-job-post__modal-head,
.business-job-post__modal--disability .business-job-post__modal-foot {
  padding-inline: 1.35rem;
}

.business-job-post__modal--disability .business-job-post__modal-head {
  align-items: flex-start;
}

.business-job-post__modal--disability .business-job-post__modal-copy {
  margin-top: 0.45rem;
}

.business-job-post__modal--disability .business-job-post__modal-body {
  padding-top: 1rem;
  padding-bottom: 1rem;
}

.business-job-post__modal--disability .business-job-post__modal-close {
  border-radius: 0;
  background: #f7faf8;
}

.business-job-post__modal--disability .business-job-post__check-option {
  border-radius: 0;
  min-height: 3.4rem;
  background: #ffffff;
}

.business-job-post__modal--disability .business-job-post__check-option.is-active {
  background: #eef6f1;
  box-shadow: inset 4px 0 0 #2c9a60;
}

.business-job-post__modal--disability .business-job-post__check-option input {
  width: 1rem;
  height: 1rem;
}

.business-job-post__modal--disability .business-job-post__modal-actions {
  align-items: center;
}

</style>
