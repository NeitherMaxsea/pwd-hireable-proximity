<script setup>
import { onBeforeUnmount, watch } from 'vue'

const props = defineProps({
  isOpen: { type: Boolean, default: false },
  candidate: { type: Object, default: () => null },
  isSubmitting: { type: Boolean, default: false },
  form: { type: Object, default: () => ({}) },
  formError: { type: String, default: '' },
  submitLabel: { type: String, default: 'Send Offer' },
  minDate: { type: String, default: '' },
})

const emit = defineEmits([
  'close',
  'submit',
  'update:formTitle',
  'update:formCompensation',
  'update:formStartDate',
  'update:formResponseDeadline',
  'update:formOfferLetter',
])

const buildUserOverviewInitials = (name, fallback) => {
  if (!name) return fallback
  const parts = String(name).trim().split(' ')
  if (parts.length > 1) return (parts[0][0] + parts[parts.length - 1][0]).toUpperCase()
  return parts[0].substring(0, 2).toUpperCase()
}

const resolveBusinessJobOfferStatusTone = (offer) => {
  const status = String(offer?.offerStatus || offer?.offer_status || '').toLowerCase().trim()
  if (status === 'accepted') return 'success'
  if (status === 'rejected') return 'danger'
  if (status === 'sent') return 'warning'
  return 'muted'
}

const formatBusinessJobOfferStatusLabel = (offer) => {
  const status = String(offer?.offerStatus || offer?.offer_status || '').toLowerCase().trim()
  if (status === 'accepted') return 'Accepted'
  if (status === 'rejected') return 'Declined'
  if (status === 'sent') return 'Offer Sent'
  return 'Not Sent'
}

const resolveBusinessJobOfferDateValue = (value) => {
  if (!value) return null
  if (value instanceof Date) return Number.isNaN(value.getTime()) ? null : value
  if (typeof value?.toDate === 'function') {
    const resolved = value.toDate()
    return resolved instanceof Date && !Number.isNaN(resolved.getTime()) ? resolved : null
  }
  if (typeof value === 'object') {
    const seconds = Number(value.seconds ?? value._seconds)
    const nanoseconds = Number(value.nanoseconds ?? value._nanoseconds ?? 0)
    if (Number.isFinite(seconds)) {
      const resolved = new Date((seconds * 1000) + Math.floor(nanoseconds / 1000000))
      return Number.isNaN(resolved.getTime()) ? null : resolved
    }
  }

  const resolved = new Date(String(value).trim())
  return Number.isNaN(resolved.getTime()) ? null : resolved
}

const formatBusinessJobOfferDateLabel = (dateRaw, options = {}) => {
  const resolved = resolveBusinessJobOfferDateValue(dateRaw)
  if (!resolved) return 'Not set'

  try {
    return new Intl.DateTimeFormat('en-PH', {
      month: 'short',
      day: 'numeric',
      year: 'numeric',
      ...options,
    }).format(resolved)
  } catch {
    return 'Not set'
  }
}

let previousHtmlOverflow = ''
let previousBodyOverflow = ''

const syncIssueOfferPageLock = (isLocked) => {
  if (typeof document === 'undefined') return

  const html = document.documentElement
  const body = document.body
  if (!html || !body) return

  if (isLocked) {
    previousHtmlOverflow = html.style.overflow
    previousBodyOverflow = body.style.overflow
    html.style.overflow = 'hidden'
    body.style.overflow = 'hidden'
    return
  }

  html.style.overflow = previousHtmlOverflow
  body.style.overflow = previousBodyOverflow
}

watch(
  () => props.isOpen,
  (isOpen) => {
    syncIssueOfferPageLock(isOpen)
  },
  { immediate: true },
)

onBeforeUnmount(() => {
  syncIssueOfferPageLock(false)
})
</script>

<template>
  <Teleport to="body">
    <Transition name="business-trial-modal">
      <div v-if="isOpen" class="business-modal business-modal--applicant-details business-modal--issue-offer" @click.self="$emit('close')">
        <div class="business-modal__card business-applicants-modal business-applicant-details-modal business-issue-offer-modal" role="dialog" aria-modal="true">
          <div class="business-modal__copy business-issue-offer-modal__copy">
            <div class="business-issue-offer-modal__copy-block">
              <p class="business-issue-offer-modal__eyebrow">Offer Workflow</p>
              <h2>Issue Job Offer</h2>
              <p>Send the offer letter below to {{ candidate?.applicantName || 'the applicant' }} for the {{ candidate?.jobTitle || 'selected' }} role.</p>
            </div>
            <button
              type="button"
              class="business-issue-offer-modal__close"
              :disabled="isSubmitting"
              @click="$emit('close')"
            >
              <i class="bi bi-x-lg" aria-hidden="true" />
            </button>
          </div>

          <div v-if="candidate" class="business-applicants-modal__body">
            <div class="business-applicants-modal__identity">
              <div class="business-applicants-modal__avatar">
                <img
                  v-if="candidate.applicantAvatar"
                  :src="candidate.applicantAvatar"
                  :alt="`${candidate.applicantName} avatar`"
                >
                <template v-else>{{ buildUserOverviewInitials(candidate.applicantName, 'AP') }}</template>
              </div>

              <div class="business-applicants-modal__identity-copy">
                <strong>{{ candidate.applicantName }}</strong>
                <span>{{ candidate.applicantEmail }}</span>
                <div class="business-issue-offer-modal__identity-meta">
                  <span class="business-issue-offer__status" :class="`is-${resolveBusinessJobOfferStatusTone(candidate)}`">
                    {{ formatBusinessJobOfferStatusLabel(candidate) }}
                  </span>
                  <small>{{ candidate.passedStageLabel }}</small>
                </div>
              </div>
            </div>

            <div class="business-applicants-modal__grid">
              <div class="business-applicants-modal__item">
                <span>Applied Job</span>
                <strong>{{ candidate.jobTitle }}</strong>
              </div>
              <div class="business-applicants-modal__item">
                <span>Passed Stage</span>
                <strong>{{ candidate.passedStageLabel }}</strong>
              </div>
              <div class="business-applicants-modal__item">
                <span>Completed At</span>
                <strong>{{ formatBusinessJobOfferDateLabel(candidate.completedAt, { hour: 'numeric', minute: '2-digit' }) }}</strong>
              </div>
              <div class="business-applicants-modal__item">
                <span>Interviewer</span>
                <strong>{{ candidate.interviewer }}</strong>
              </div>
            </div>

            <div class="business-issue-offer-modal__form">
              <label class="business-interview-review-modal__option-field">
                <span>Offer Title</span>
                <input
                  :value="form.offerTitle"
                  @input="$emit('update:formTitle', $event.target.value)"
                  type="text"
                  placeholder="Customer Support Associate Job Offer"
                >
              </label>

              <div class="business-issue-offer-modal__grid">
                <label class="business-interview-review-modal__option-field">
                  <span>Compensation</span>
                  <input
                    :value="form.compensation"
                    @input="$emit('update:formCompensation', $event.target.value)"
                    type="text"
                    placeholder="PHP 25,000 monthly"
                  >
                </label>

                <label class="business-interview-review-modal__option-field">
                  <span>Start Date</span>
                  <input
                    :value="form.startDate"
                    @input="$emit('update:formStartDate', $event.target.value)"
                    type="date"
                    :min="minDate"
                  >
                </label>

                <label class="business-interview-review-modal__option-field">
                  <span>Response Deadline</span>
                  <input
                    :value="form.responseDeadline"
                    @input="$emit('update:formResponseDeadline', $event.target.value)"
                    type="date"
                    :min="minDate"
                  >
                </label>
              </div>

              <label class="business-applicants-modal__reason business-issue-offer-modal__letter">
                <span>Job Offer Letter</span>
                <textarea
                  :value="form.offerLetter"
                  @input="$emit('update:formOfferLetter', $event.target.value)"
                  rows="9"
                  placeholder="Write the full job offer details that the applicant should review on their Job Offers page."
                />
              </label>
            </div>

            <p v-if="formError" class="business-applicants-modal__error">
              {{ formError }}
            </p>
          </div>

          <div class="business-modal__actions business-issue-offer-modal__actions">
            <button
              type="button"
              class="business-modal__button business-modal__button--secondary"
              :disabled="isSubmitting"
              @click="$emit('close')"
            >
              Close
            </button>
            <button
              type="button"
              class="business-modal__button business-modal__button--primary"
              :disabled="isSubmitting"
              @click="$emit('submit')"
            >
              <span v-if="isSubmitting" class="business-issue-offer-modal__button-spinner" aria-hidden="true">
                <span></span>
                <span></span>
                <span></span>
              </span>
              <span>{{ isSubmitting ? 'Sending offer' : submitLabel }}</span>
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style scoped>
.business-modal--issue-offer {
  width: 100vw;
  min-height: 100vh;
  min-height: 100dvh;
  place-items: center;
  align-content: center;
  padding: clamp(0.85rem, 2.6vh, 1.5rem) clamp(0.85rem, 2vw, 1.5rem);
  overflow-y: auto;
  overscroll-behavior: contain;
  scrollbar-gutter: stable both-edges;
  background:
    radial-gradient(circle at top, rgba(106, 179, 137, 0.22), transparent 36%),
    linear-gradient(180deg, rgba(244, 248, 244, 0.62) 0%, rgba(225, 236, 230, 0.78) 100%);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
}

.business-issue-offer-modal {
  width: min(48rem, calc(100vw - 2rem));
  max-height: min(88vh, 43rem);
  grid-template-rows: auto minmax(0, 1fr) auto;
  gap: 0;
  padding: 0;
  margin: 0 auto;
  overflow: hidden;
  border-color: rgba(196, 216, 205, 0.9);
  background:
    radial-gradient(circle at top left, rgba(232, 247, 238, 0.9), transparent 36%),
    linear-gradient(180deg, rgba(255, 255, 255, 0.99) 0%, rgba(246, 250, 247, 0.98) 100%);
}

.business-issue-offer-modal__copy {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 0.85rem;
  padding: 1.05rem 1rem 0.9rem;
  border-bottom: 1px solid rgba(215, 226, 220, 0.92);
  background: linear-gradient(180deg, rgba(247, 251, 248, 0.98) 0%, rgba(255, 255, 255, 0.98) 100%);
}

.business-issue-offer-modal__copy-block {
  display: grid;
  gap: 0.28rem;
}

.business-issue-offer-modal__eyebrow {
  margin: 0;
  color: #6a8476;
  font-size: 0.66rem;
  font-weight: 800;
  letter-spacing: 0.12em;
  text-transform: uppercase;
}

.business-issue-offer-modal__copy :deep(h2) {
  margin: 0;
  color: #183126;
  font-size: 1.28rem;
  line-height: 1.1;
}

.business-issue-offer-modal__copy :deep(p:last-child) {
  margin: 0;
  color: #607167;
  font-size: 0.84rem;
  line-height: 1.48;
  max-width: 38rem;
}

.business-issue-offer-modal__close {
  width: 2.1rem;
  height: 2.1rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: 1px solid rgba(204, 216, 209, 0.94);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.96);
  color: #50665a;
  cursor: pointer;
  transition: background-color 0.18s ease, border-color 0.18s ease, color 0.18s ease, transform 0.18s ease;
}

.business-issue-offer-modal__close:hover:not(:disabled) {
  border-color: rgba(121, 178, 147, 0.52);
  background: rgba(238, 248, 242, 0.98);
  color: #1f6b4d;
  transform: translateY(-1px);
}

.business-issue-offer-modal :deep(.business-applicants-modal__body) {
  min-height: 0;
  overflow-y: auto;
  padding: 0.9rem 1rem 0.9rem;
  scrollbar-gutter: stable;
  scroll-padding-bottom: 1rem;
}

.business-issue-offer-modal :deep(.business-applicants-modal__identity) {
  gap: 0.85rem;
  padding: 0 0 0.95rem;
  border: 0;
  border-bottom: 1px solid rgba(214, 225, 218, 0.92);
  border-radius: 0;
  background: transparent;
  box-shadow: none;
}

.business-issue-offer-modal :deep(.business-applicants-modal__grid) {
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0.75rem;
}

.business-issue-offer-modal :deep(.business-applicants-modal__item) {
  min-height: auto;
  align-content: start;
  padding: 0 0 0.7rem;
  border: 0;
  border-bottom: 1px solid rgba(221, 230, 224, 0.92);
  border-radius: 0;
  background: transparent;
  box-shadow: none;
}

.business-issue-offer-modal :deep(.business-applicants-modal__item::before) {
  display: none;
}

.business-issue-offer-modal :deep(.business-applicants-modal__item span) {
  font-size: 0.66rem;
  color: #6a8174;
}

.business-issue-offer-modal :deep(.business-applicants-modal__item strong) {
  font-size: 0.9rem;
  line-height: 1.4;
}

.business-issue-offer-modal :deep(.business-applicants-modal__avatar) {
  width: 3.35rem;
  height: 3.35rem;
  border-radius: 1rem;
  font-size: 0.9rem;
}

.business-issue-offer-modal :deep(.business-applicants-modal__identity-copy) {
  gap: 0.28rem;
}

.business-issue-offer-modal :deep(.business-applicants-modal__identity-copy strong) {
  font-size: 1rem;
}

.business-issue-offer-modal :deep(.business-applicants-modal__identity-copy span) {
  font-size: 0.8rem;
}

.business-issue-offer-modal__identity-meta {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  flex-wrap: wrap;
}

.business-issue-offer-modal__identity-meta small {
  color: #6b7d72;
  font-size: 0.72rem;
  font-weight: 700;
}

.business-issue-offer-modal__actions {
  position: relative;
  margin-top: auto;
  padding: 0.8rem 1rem 0.95rem;
  border-top: 1px solid rgba(217, 226, 220, 0.92);
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.78) 0%, rgba(255, 255, 255, 0.96) 34%, #ffffff 100%);
  box-shadow: 0 -10px 24px rgba(18, 54, 33, 0.05);
}

.business-issue-offer-modal__actions .business-modal__button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.55rem;
  min-width: 8.2rem;
}

.business-issue-offer-modal__button-spinner {
  display: inline-flex;
  align-items: center;
  gap: 0.22rem;
}

.business-issue-offer-modal__button-spinner span {
  width: 0.32rem;
  height: 0.32rem;
  border-radius: 999px;
  background: currentColor;
  opacity: 0.32;
  animation: business-issue-offer-modal-dot 1s ease-in-out infinite;
}

.business-issue-offer-modal__button-spinner span:nth-child(2) {
  animation-delay: 0.12s;
}

.business-issue-offer-modal__button-spinner span:nth-child(3) {
  animation-delay: 0.24s;
}

.business-issue-offer__status {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: fit-content;
  padding: 0.34rem 0.7rem;
  border-radius: 999px;
  font-size: 0.72rem;
  font-weight: 700;
}

.business-issue-offer__status.is-success {
  background: rgba(34, 197, 94, 0.12);
  color: #166534;
}

.business-issue-offer__status.is-danger {
  background: rgba(239, 68, 68, 0.12);
  color: #b91c1c;
}

.business-issue-offer__status.is-warning {
  background: rgba(245, 158, 11, 0.14);
  color: #b45309;
}

.business-issue-offer__status.is-muted {
  background: rgba(148, 163, 184, 0.14);
  color: #475569;
}

.business-issue-offer-modal__form {
  display: grid;
  gap: 0.75rem;
  margin-top: 0.2rem;
  padding-top: 0.15rem;
}

.business-issue-offer-modal__grid {
  display: grid;
  gap: 0.8rem;
  grid-template-columns: repeat(3, minmax(0, 1fr));
}

.business-issue-offer-modal :deep(.business-interview-review-modal__option-field),
.business-issue-offer-modal :deep(.business-applicants-modal__reason) {
  border: 0;
  border-radius: 0;
  background: transparent;
  box-shadow: none;
}

.business-issue-offer-modal :deep(.business-interview-review-modal__option-field) {
  gap: 0.34rem;
  padding: 0;
}

.business-issue-offer-modal :deep(.business-interview-review-modal__option-field span),
.business-issue-offer-modal :deep(.business-applicants-modal__reason span) {
  font-size: 0.68rem;
  color: #60766a;
}

.business-issue-offer-modal :deep(.business-interview-review-modal__option-field input) {
  min-height: 2.55rem;
  padding: 0.62rem 0.78rem;
  border-color: rgba(203, 216, 209, 0.96);
  border-radius: 0.8rem;
  background: #ffffff;
  box-shadow: none;
  font-size: 0.94rem;
}

.business-issue-offer-modal__letter {
  gap: 0.45rem;
  padding: 0;
}

.business-issue-offer-modal__letter textarea {
  min-height: clamp(6.5rem, 16vh, 8.75rem);
  padding: 0.8rem 0.9rem;
  border-color: rgba(203, 216, 209, 0.96);
  border-radius: 0.8rem;
  box-shadow: none;
}

@media (max-width: 980px) {
  .business-issue-offer-modal {
    width: min(44rem, calc(100vw - 1.25rem));
  }

  .business-issue-offer-modal :deep(.business-applicants-modal__grid),
  .business-issue-offer-modal__grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 720px) {
  .business-modal--issue-offer {
    place-items: start center;
    padding: 0.55rem;
  }

  .business-issue-offer-modal {
    width: min(100%, calc(100vw - 0.5rem));
    max-height: calc(100vh - 0.75rem);
  }

  .business-issue-offer-modal :deep(.business-applicants-modal__grid),
  .business-issue-offer-modal__grid {
    grid-template-columns: 1fr;
  }

  .business-issue-offer-modal__copy,
  .business-issue-offer-modal :deep(.business-applicants-modal__body),
  .business-issue-offer-modal__actions {
    padding-left: 0.95rem;
    padding-right: 0.95rem;
  }
}

@media (max-height: 760px) and (orientation: landscape) {
  .business-modal--issue-offer {
    place-items: stretch center;
    padding: 0.6rem;
  }

  .business-issue-offer-modal {
    width: min(50rem, calc(100vw - 1rem));
    max-height: calc(100vh - 0.75rem);
  }

  .business-issue-offer-modal__letter textarea {
    min-height: 6rem;
  }
}

@keyframes business-issue-offer-modal-dot {
  0%,
  80%,
  100% {
    transform: translateY(0);
    opacity: 0.32;
  }

  40% {
    transform: translateY(-0.16rem);
    opacity: 1;
  }
}
</style>
