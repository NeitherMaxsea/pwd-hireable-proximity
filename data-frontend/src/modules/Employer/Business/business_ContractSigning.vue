<script setup>
import { computed, ref } from 'vue'
import AdminSimpleModal from '@/modules/Admin/admin-simple-modal.vue'

const props = defineProps({
  overviewCards: {
    type: Array,
    default: () => [],
  },
  filterChips: {
    type: Array,
    default: () => [],
  },
  activeFilter: {
    type: String,
    default: 'all',
  },
  setFilter: {
    type: Function,
    default: () => {},
  },
  syncLabel: {
    type: String,
    default: '',
  },
  traceSummary: {
    type: String,
    default: '',
  },
  refreshQueue: {
    type: Function,
    default: () => {},
  },
  rows: {
    type: Array,
    default: () => [],
  },
  selectedRowId: {
    type: String,
    default: '',
  },
  selectRow: {
    type: Function,
    default: () => {},
  },
  sendContractToApplicant: {
    type: Function,
    default: () => {},
  },
  selectedRow: {
    type: Object,
    default: null,
  },
  selectedRecord: {
    type: Object,
    default: null,
  },
  contractDraft: {
    type: Object,
    default: () => ({}),
  },
  setContractDraftField: {
    type: Function,
    default: () => {},
  },
  restoreContractDraft: {
    type: Function,
    default: () => {},
  },
  saveAndSendBusinessContract: {
    type: Function,
    default: () => {},
  },
  isBusinessContractSaving: {
    type: Boolean,
    default: false,
  },
  canEditBusinessModule: {
    type: Function,
    default: () => false,
  },
  businessContractSignatureName: {
    type: String,
    default: '',
  },
  setBusinessContractSignatureName: {
    type: Function,
    default: () => {},
  },
  activeBusinessContractSigningId: {
    type: String,
    default: '',
  },
  completeBusinessContractSigning: {
    type: Function,
    default: () => {},
  },
})

const contractRows = computed(() => (Array.isArray(props.rows) ? props.rows : []))
const summaryCards = computed(() => (Array.isArray(props.overviewCards) ? props.overviewCards : []))
const filters = computed(() => (Array.isArray(props.filterChips) ? props.filterChips : []))
const currentRow = computed(() => props.selectedRow || null)
const currentRecord = computed(() => props.selectedRecord || null)
const canEditContracts = computed(() => props.canEditBusinessModule('contract-signing') === true)
const isSendContractModalOpen = ref(false)
const sendContractModalRowId = ref('')
const pendingContractUpload = ref(null)
const contractUploadError = ref('')
const contractUploadInputKey = ref(0)

const activeSendContractRow = computed(() =>
  contractRows.value.find((row) => String(row?.id || '').trim() === String(sendContractModalRowId.value || '').trim())
  || currentRow.value
  || null,
)

const formatDateLabel = (value, options = {}) => {
  const parsedValue = new Date(String(value || '').trim())
  if (Number.isNaN(parsedValue.getTime())) return 'Not set'

  return parsedValue.toLocaleString('en-US', {
    month: 'short',
    day: '2-digit',
    year: 'numeric',
    ...options,
  })
}

const resolveRowStatusTone = (value) => {
  const normalizedValue = String(value || '').trim().toLowerCase()
  if (normalizedValue === 'completed') return 'success'
  if (normalizedValue === 'applicant_signed') return 'info'
  if (normalizedValue === 'sent') return 'warning'
  return 'muted'
}

const resolveRowStatusLabel = (row = {}) =>
  String(row?.statusLabel || row?.status || 'Ready').trim() || 'Ready'

const isSelectedRow = (rowId) => String(props.selectedRowId || '').trim() === String(rowId || '').trim()
const isActiveSigning = (recordId) =>
  String(props.activeBusinessContractSigningId || '').trim() === String(recordId || '').trim()

const formatFileSize = (value) => {
  const size = Number(value)
  if (!Number.isFinite(size) || size <= 0) return ''
  if (size < 1024) return `${size} B`
  if (size < 1024 * 1024) return `${(size / 1024).toFixed(1)} KB`
  return `${(size / (1024 * 1024)).toFixed(1)} MB`
}

const resetSendContractModal = () => {
  isSendContractModalOpen.value = false
  sendContractModalRowId.value = ''
  pendingContractUpload.value = null
  contractUploadError.value = ''
  contractUploadInputKey.value += 1
}

const openSendContractModal = (rowId = '') => {
  const normalizedRowId = String(rowId || '').trim()
  if (!normalizedRowId) return

  props.selectRow(normalizedRowId)
  sendContractModalRowId.value = normalizedRowId
  pendingContractUpload.value = null
  contractUploadError.value = ''
  contractUploadInputKey.value += 1
  isSendContractModalOpen.value = true
}

const handleContractUploadChange = (event) => {
  const nextFile = event?.target?.files?.[0] || null
  pendingContractUpload.value = nextFile
  contractUploadError.value = ''
}

const confirmSendContract = async () => {
  const targetRow = activeSendContractRow.value
  if (!targetRow) return

  if (!pendingContractUpload.value) {
    contractUploadError.value = 'Upload muna ng contract file bago mag-confirm.'
    return
  }

  contractUploadError.value = ''
  const sendResult = await props.sendContractToApplicant(targetRow.id, {
    file: pendingContractUpload.value,
  })
  if (sendResult?.ok === true) {
    resetSendContractModal()
    return
  }

  contractUploadError.value = String(
    sendResult?.message
    || 'Hindi na-send ang contract. Pakicheck ang required fields, permissions, at Firebase deployment.',
  ).trim()
}

const openFileLink = (url = '') => {
  const normalizedUrl = String(url || '').trim()
  if (!normalizedUrl || typeof window === 'undefined') return
  window.open(normalizedUrl, '_blank', 'noopener,noreferrer')
}
</script>

<template>
  <section class="business-contract-signing">
    <div class="business-contract-signing__hero business-job-post__lead">
      <div class="business-contract-signing__hero-copy business-job-post__copy">
        <span class="business-contract-signing__eyebrow business-job-post__eyebrow">Offer &amp; Onboarding</span>
        <h2>Contract Signing</h2>
        <p>
          Dito lang lalabas ang applicants na na-issue-han ng offer at nag-confirm na sa kanilang Job Offers page.
          Mula rito puwede nang mag-upload at mag-send ng contract file, tapos i-review ang signed copy na ibabalik ng applicant.
        </p>
        <div class="business-job-post__lead-meta business-contract-signing__lead-meta">
          <span class="business-job-post__lead-chip">
            <i class="bi bi-clock-history" aria-hidden="true" />
            {{ syncLabel }}
          </span>
          <span class="business-job-post__lead-chip">
            <i class="bi bi-diagram-3" aria-hidden="true" />
            {{ traceSummary }}
          </span>
        </div>
      </div>
    </div>

    <div class="business-job-post__highlights business-contract-signing__summary-grid">
      <article
        v-for="card in summaryCards"
        :key="card.label"
        class="business-job-post__highlight business-contract-signing__summary-card"
      >
        <span>{{ card.label }}</span>
        <strong>{{ card.value }}</strong>
        <p>{{ card.copy }}</p>
      </article>
    </div>

    <div class="business-contract-signing__toolbar">
      <div class="business-contract-signing__filters">
        <button
          v-for="chip in filters"
          :key="chip.id"
          type="button"
          class="business-contract-signing__filter"
          :class="{ 'is-active': activeFilter === chip.id }"
          @click="setFilter(chip.id)"
        >
          <span>{{ chip.label }}</span>
          <strong>{{ chip.count }}</strong>
        </button>
      </div>

      <button
        type="button"
        class="business-contract-signing__button business-contract-signing__button--secondary business-job-post__button business-job-post__button--ghost"
        @click="refreshQueue"
      >
        <i class="bi bi-arrow-clockwise" aria-hidden="true" />
        <span>Refresh Fetch</span>
      </button>
    </div>

    <div class="business-contract-signing__layout">
      <article class="business-contract-signing__panel business-job-post__panel">
        <div class="business-contract-signing__panel-head business-job-post__panel-head">
          <div>
            <p class="business-contract-signing__panel-label business-job-post__tips-label">Contract Queue</p>
            <h3>Confirmed applicant table</h3>
            <p>Kapag accepted na ang job offer, dito na sila lalabas para sa contract assignment.</p>
          </div>
          <span class="business-contract-signing__count business-job-post__panel-chip">
            {{ contractRows.length }} {{ contractRows.length === 1 ? 'record' : 'records' }}
          </span>
        </div>

        <div class="business-contract-signing__table-shell">
          <table class="business-contract-signing__table">
            <thead>
              <tr>
                <th>Applicant</th>
                <th>Role</th>
                <th>Status</th>
                <th>Actions</th>
              </tr>
            </thead>

            <tbody v-if="contractRows.length">
              <tr
                v-for="row in contractRows"
                :key="row.id"
                class="business-contract-signing__row"
                :class="{ 'is-selected': isSelectedRow(row.id) }"
                @click="selectRow(row.id)"
              >
                <td>
                  <div class="business-contract-signing__identity">
                    <span class="business-contract-signing__avatar">
                      {{ String(row.name || 'A').charAt(0).toUpperCase() }}
                    </span>
                    <div>
                      <strong>{{ row.name }}</strong>
                      <small>{{ row.email || 'No email' }}</small>
                      <small>ID: {{ row.applicationId || row.id }}</small>
                      <small>{{ row.jobId ? `Job: ${row.jobId}` : 'No job id' }}</small>
                    </div>
                  </div>
                </td>
                <td>
                  <strong>{{ row.role || row.jobTitle }}</strong>
                  <small>{{ row.offerTitle || 'Issued Job Offer' }}</small>
                  <small v-if="row.businessContractFileName">
                    Contract: {{ row.businessContractFileName }}
                  </small>
                  <small v-if="row.applicantSignedContractFileName">
                    Signed copy: {{ row.applicantSignedContractFileName }}
                  </small>
                </td>
                <td>
                  <span
                    class="business-contract-signing__status"
                    :class="`is-${resolveRowStatusTone(row.status)}`"
                  >
                    {{ resolveRowStatusLabel(row) }}
                  </span>
                  <small>
                    {{ row.status === 'completed'
                      ? 'Fully signed'
                      : row.status === 'applicant_signed'
                        ? 'Applicant returned a signed file'
                        : row.status === 'sent'
                          ? 'Waiting for applicant signed upload'
                          : row.offerAcceptedAt
                            ? `Offer confirmed ${formatDateLabel(row.offerAcceptedAt, { hour: 'numeric', minute: '2-digit' })}`
                            : 'Ready to send contract' }}
                  </small>
                </td>
                <td class="business-contract-signing__actions-cell">
                  <div class="business-contract-signing__actions business-contract-signing__actions--table">
                    <button
                      v-if="row.canSend && canEditContracts"
                      type="button"
                      class="business-contract-signing__button business-contract-signing__button--primary business-job-post__save"
                      :disabled="isBusinessContractSaving"
                      @click.stop="openSendContractModal(row.id)"
                    >
                      {{
                        isBusinessContractSaving && isSelectedRow(row.id)
                          ? 'Sending...'
                          : row.contractId
                            ? 'Resend Contract'
                            : 'Send Contract'
                      }}
                    </button>
                    <button
                      v-if="row.businessContractDownloadUrl"
                      type="button"
                      class="business-contract-signing__button business-contract-signing__button--secondary business-job-post__secondary"
                      @click.stop="openFileLink(row.businessContractDownloadUrl)"
                    >
                      View Contract
                    </button>
                    <button
                      v-if="row.applicantSignedContractDownloadUrl"
                      type="button"
                      class="business-contract-signing__button business-contract-signing__button--secondary business-job-post__secondary"
                      @click.stop="openFileLink(row.applicantSignedContractDownloadUrl)"
                    >
                      View Signed File
                    </button>
                    <span
                      v-if="row.canSend && !canEditContracts"
                      class="business-contract-signing__state-note"
                    >
                      Owner access required
                    </span>
                    <span
                      v-else-if="!row.canSend && !row.businessContractDownloadUrl && !row.applicantSignedContractDownloadUrl"
                      class="business-contract-signing__state-note"
                    >
                      {{ row.status === 'completed' ? 'Completed' : 'Waiting' }}
                    </span>
                  </div>
                </td>
              </tr>
            </tbody>

            <tbody v-else>
              <tr>
                <td colspan="4" class="business-contract-signing__empty-cell">
                  <div class="business-contract-signing__empty business-contract-signing__empty--table">
                    <i class="bi bi-file-earmark-lock" aria-hidden="true" />
                    <h3>No confirmed offers yet</h3>
                    <p>
                      Kailangan munang ma-send ang offer mula sa Issue Offer at i-confirm ng applicant sa Job Offers page
                      bago sila pumasok dito sa Contract Signing.
                    </p>
                    <small>{{ traceSummary }}</small>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </article>
    </div>

    <AdminSimpleModal
      :open="isSendContractModalOpen"
      title="Send Contract"
      :subtitle="activeSendContractRow ? `${activeSendContractRow.name} - ${activeSendContractRow.jobTitle || activeSendContractRow.role}` : 'Upload the contract file and confirm the send action.'"
      max-width="34rem"
      @close="resetSendContractModal"
    >
      <div class="business-contract-signing__modal-body">
        <div v-if="activeSendContractRow" class="business-contract-signing__modal-summary">
          <div>
            <span>Applicant</span>
            <strong>{{ activeSendContractRow.name }}</strong>
          </div>
          <div>
            <span>Email</span>
            <strong>{{ activeSendContractRow.email || 'No email' }}</strong>
          </div>
          <div>
            <span>Role</span>
            <strong>{{ activeSendContractRow.role || activeSendContractRow.jobTitle }}</strong>
          </div>
          <div>
            <span>Status</span>
            <strong>{{ resolveRowStatusLabel(activeSendContractRow) }}</strong>
          </div>
        </div>

        <label class="business-contract-signing__field business-contract-signing__field--modal business-job-post__field">
          <span>Upload Contract</span>
          <input
            :key="contractUploadInputKey"
            type="file"
            accept=".pdf,.doc,.docx"
            @change="handleContractUploadChange"
          >
        </label>

        <div v-if="pendingContractUpload" class="business-contract-signing__upload-preview">
          <strong>{{ pendingContractUpload.name }}</strong>
          <span>{{ formatFileSize(pendingContractUpload.size) || 'Selected file' }}</span>
        </div>

        <p class="business-contract-signing__modal-note">
          Realtime na ito. Once ma-confirm mo, mase-save ang contract file sa Firebase at lalabas agad ito sa Contracts page ng applicant.
        </p>

        <p v-if="!canEditContracts" class="business-contract-signing__modal-error">
          Contract sending is only available in the main business owner workspace.
        </p>

        <p v-if="contractUploadError" class="business-contract-signing__modal-error">
          {{ contractUploadError }}
        </p>
      </div>

      <template #actions>
        <button
          type="button"
          class="business-contract-signing__button business-contract-signing__button--secondary business-job-post__secondary"
          @click="resetSendContractModal"
        >
          Cancel
        </button>
        <button
          type="button"
          class="business-contract-signing__button business-contract-signing__button--primary business-job-post__save"
          :disabled="isBusinessContractSaving || !canEditContracts"
          @click="confirmSendContract"
        >
          {{ isBusinessContractSaving ? 'Sending...' : 'Confirm Send' }}
        </button>
      </template>
    </AdminSimpleModal>
  </section>
</template>

<style scoped>
.business-contract-signing {
  display: grid;
  gap: 1.5rem;
}

.business-contract-signing__hero,
.business-contract-signing__panel,
.business-contract-signing__summary-card {
  border: 1px solid rgba(214, 227, 219, 0.92);
  border-radius: 1.2rem;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.99) 0%, rgba(249, 252, 250, 0.97) 100%);
  box-shadow: 0 14px 26px rgba(71, 112, 90, 0.08);
}

.business-contract-signing__hero {
  display: grid;
  gap: 1.2rem;
  padding: 1.35rem;
}

.business-contract-signing__toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  flex-wrap: wrap;
}

.business-contract-signing__eyebrow,
.business-contract-signing__panel-label,
.business-contract-signing__field span,
.business-contract-signing__meta-grid span,
.business-contract-signing__timeline-item span,
.business-contract-signing__signature-preview span {
  color: #5d7467;
  font-size: 0.73rem;
  font-weight: 700;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.business-contract-signing__hero-copy h2,
.business-contract-signing__hero-copy p,
.business-contract-signing__panel-head h3,
.business-contract-signing__panel-head p,
.business-contract-signing__summary-card p,
.business-contract-signing__empty h3,
.business-contract-signing__empty p {
  margin: 0;
}

.business-contract-signing__hero-copy {
  display: grid;
  gap: 0.55rem;
}

.business-contract-signing__hero-copy h2 {
  color: #183126;
  font-size: 1.8rem;
  font-weight: 800;
}

.business-contract-signing__hero-copy p,
.business-contract-signing__panel-head p,
.business-contract-signing__summary-card p,
.business-contract-signing__empty p,
.business-contract-signing__table small,
.business-contract-signing__meta-grid strong,
.business-contract-signing__timeline-item strong {
  color: #60786a;
  line-height: 1.55;
}

.business-contract-signing__summary-grid,
.business-contract-signing__meta-grid,
.business-contract-signing__form-grid {
  display: grid;
  gap: 0.9rem;
  grid-template-columns: repeat(auto-fit, minmax(11rem, 1fr));
}

.business-contract-signing__summary-card {
  padding: 1rem 1.1rem;
}

.business-contract-signing__summary-card span {
  display: block;
  color: #6b8576;
  font-size: 0.76rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.business-contract-signing__summary-card strong {
  display: block;
  margin-top: 0.45rem;
  color: #183126;
  font-size: 1.6rem;
}

.business-contract-signing__filters {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.business-contract-signing__filter,
.business-contract-signing__button,
.business-contract-signing__row-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.45rem;
  border-radius: 0.82rem;
  font: inherit;
  cursor: pointer;
  transition: transform 0.18s ease, box-shadow 0.18s ease, background-color 0.18s ease;
}

.business-contract-signing__filter {
  display: inline-flex;
  align-items: center;
  gap: 0.55rem;
  padding: 0.72rem 1rem;
  background: #edf4ef;
  color: #244534;
  font-weight: 700;
}

.business-contract-signing__filter strong {
  display: inline-flex;
  min-width: 1.8rem;
  justify-content: center;
}

.business-contract-signing__filter.is-active {
  background: linear-gradient(135deg, #198754, #2f9f6c);
  color: #fff;
  box-shadow: 0 14px 24px rgba(25, 135, 84, 0.2);
}

.business-contract-signing__layout {
  display: grid;
  gap: 1.25rem;
  grid-template-columns: minmax(0, 1fr);
}

.business-contract-signing__panel {
  padding: 1.1rem;
}

.business-contract-signing__panel--editor {
  display: grid;
  align-content: start;
  gap: 1rem;
}

.business-contract-signing__panel-head {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  align-items: flex-start;
  margin-bottom: 1rem;
}

.business-contract-signing__panel-head h3 {
  color: #183126;
  font-size: 1.15rem;
}

.business-contract-signing__count {
  padding: 0;
  background: transparent;
  color: inherit;
  font-size: 0.78rem;
  font-weight: 700;
}

.business-contract-signing__table-shell {
  overflow: hidden;
  border: 1px solid rgba(223, 232, 226, 0.95);
  border-radius: 1rem;
  background: #ffffff;
}

.business-contract-signing__table {
  width: 100%;
  border-collapse: collapse;
  table-layout: fixed;
}

.business-contract-signing__table th,
.business-contract-signing__table td {
  padding: 0.9rem 0.75rem;
  text-align: left;
  border-bottom: 1px solid rgba(224, 231, 226, 0.95);
  vertical-align: top;
  overflow-wrap: anywhere;
}

.business-contract-signing__table th {
  color: #658072;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.business-contract-signing__table th:last-child,
.business-contract-signing__actions-cell {
  width: 12.5rem;
}

.business-contract-signing__row {
  cursor: pointer;
  transition: background-color 0.18s ease;
}

.business-contract-signing__row:hover,
.business-contract-signing__row.is-selected {
  background: rgba(236, 247, 240, 0.9);
}

.business-contract-signing__identity {
  display: flex;
  gap: 0.7rem;
  align-items: flex-start;
  min-width: 0;
}

.business-contract-signing__identity > div {
  min-width: 0;
}

.business-contract-signing__identity strong,
.business-contract-signing__table strong,
.business-contract-signing__meta-grid strong,
.business-contract-signing__timeline-item strong {
  display: block;
  color: #183126;
}

.business-contract-signing__avatar {
  width: 2.45rem;
  height: 2.45rem;
  border-radius: 999px;
  display: grid;
  place-items: center;
  background: linear-gradient(135deg, #dff3e6, #ecf8f1);
  color: #1c6d43;
  font-weight: 800;
}

.business-contract-signing__status {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.38rem 0.72rem;
  border-radius: 999px;
  font-size: 0.74rem;
  font-weight: 700;
}

.business-contract-signing__status.is-success {
  background: rgba(34, 197, 94, 0.12);
  color: #166534;
}

.business-contract-signing__status.is-info {
  background: rgba(14, 165, 233, 0.12);
  color: #075985;
}

.business-contract-signing__status.is-warning {
  background: rgba(245, 158, 11, 0.14);
  color: #b45309;
}

.business-contract-signing__status.is-muted {
  background: rgba(148, 163, 184, 0.14);
  color: #475569;
}

.business-contract-signing__row-button,
.business-contract-signing__button {
  padding: 0.72rem 1rem;
  font-weight: 700;
  border: 1px solid transparent;
}

.business-contract-signing__row-button,
.business-contract-signing__button--secondary {
  border-color: #d7dfd9;
  background: linear-gradient(180deg, #ffffff 0%, #f6f9f7 100%);
  color: #305141;
}

.business-contract-signing__button--primary {
  border-color: #cfe6d7;
  background: linear-gradient(180deg, #ffffff 0%, #eef8f2 100%);
  color: #1f6f46;
  box-shadow: 0 8px 18px rgba(15, 23, 42, 0.08);
}

.business-contract-signing__button:disabled,
.business-contract-signing__row-button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
  box-shadow: none;
}

.business-contract-signing__field {
  display: grid;
  gap: 0.45rem;
}

.business-contract-signing__field--wide {
  grid-column: 1 / -1;
}

.business-contract-signing__field input,
.business-contract-signing__field textarea {
  width: 100%;
  border: 1px solid rgba(203, 213, 208, 0.96);
  border-radius: 0;
  padding: 0.82rem 0.95rem;
  background: #ffffff;
  color: #183126;
  font: inherit;
}

.business-contract-signing__field textarea {
  resize: vertical;
}

.business-contract-signing__actions {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.business-contract-signing__actions--table {
  align-items: center;
  flex-wrap: wrap;
}

.business-contract-signing__state-note {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 2.45rem;
  padding: 0.7rem 1rem;
  border-radius: 0.82rem;
  background: rgba(148, 163, 184, 0.14);
  color: #475569;
  font-size: 0.8rem;
  font-weight: 700;
}

.business-contract-signing__timeline {
  display: grid;
  gap: 0.75rem;
  grid-template-columns: repeat(auto-fit, minmax(10rem, 1fr));
}

.business-contract-signing__timeline-item {
  padding: 0.9rem 1rem;
  border-radius: 0.95rem;
  background: #ffffff;
  border: 1px solid rgba(223, 232, 226, 0.95);
}

.business-contract-signing__signature-preview {
  display: grid;
  gap: 0.5rem;
  padding: 1rem;
  border-radius: 0.95rem;
  border: 1px solid rgba(223, 232, 226, 0.95);
  background: #ffffff;
}

.business-contract-signing__signature-preview img {
  max-width: 16rem;
  width: 100%;
  border-radius: 0.9rem;
  border: 1px solid rgba(212, 221, 215, 0.95);
  background: #fff;
}

.business-contract-signing__signature-grid {
  display: grid;
  gap: 0.85rem;
  grid-template-columns: repeat(auto-fit, minmax(12rem, 1fr));
}

.business-contract-signing__empty {
  display: grid;
  place-items: center;
  gap: 0.55rem;
  min-height: 18rem;
  text-align: center;
  padding: 1.5rem;
  border-radius: 1rem;
  border: 1px dashed rgba(192, 205, 197, 0.95);
  background: #fbfdfb;
}

.business-contract-signing__empty i {
  color: #198754;
  font-size: 1.8rem;
}

.business-contract-signing__empty-cell {
  padding: 0 !important;
  border-bottom: 0 !important;
}

.business-contract-signing__empty--table {
  min-height: auto;
  border: 0;
  border-radius: 0;
}

.business-contract-signing__modal-body,
.business-contract-signing__modal-summary {
  display: grid;
  gap: 0.85rem;
}

.business-contract-signing__modal-summary {
  grid-template-columns: repeat(2, minmax(0, 1fr));
  padding: 0.95rem 1rem;
  border: 1px solid rgba(223, 232, 226, 0.95);
  border-radius: 0.95rem;
  background: #f9fcfa;
}

.business-contract-signing__modal-summary div {
  display: grid;
  gap: 0.22rem;
}

.business-contract-signing__modal-summary strong,
.business-contract-signing__upload-preview strong {
  color: #183126;
}

.business-contract-signing__field--modal input[type="file"] {
  padding: 0.75rem;
  border-radius: 0.85rem;
  border: 1px dashed rgba(150, 173, 160, 0.95);
  background: #fdfefd;
}

.business-contract-signing__upload-preview {
  display: grid;
  gap: 0.18rem;
  padding: 0.85rem 0.95rem;
  border-radius: 0.9rem;
  background: rgba(236, 247, 240, 0.9);
  border: 1px solid rgba(206, 227, 214, 0.95);
}

.business-contract-signing__upload-preview span,
.business-contract-signing__modal-note {
  color: #60786a;
  line-height: 1.5;
}

.business-contract-signing__modal-error {
  margin: 0;
  color: #b42318;
  font-size: 0.82rem;
  font-weight: 700;
}

@media (max-width: 1180px) {
  .business-contract-signing__layout {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 720px) {
  .business-contract-signing__hero,
  .business-contract-signing__panel {
    padding: 1rem;
  }

  .business-contract-signing__toolbar,
  .business-contract-signing__panel-head {
    flex-direction: column;
  }

  .business-contract-signing__filters,
  .business-contract-signing__actions {
    flex-direction: column;
  }

  .business-contract-signing__modal-summary {
    grid-template-columns: 1fr;
  }

  .business-contract-signing__filter,
  .business-contract-signing__button,
  .business-contract-signing__row-button {
    width: 100%;
    justify-content: center;
  }
}
</style>
