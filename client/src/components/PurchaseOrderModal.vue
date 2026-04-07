<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Item summary header -->
            <div class="item-header">
              <div class="item-title-section">
                <h4 class="item-name">{{ translateProductName(backlogItem.item_name) }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <span class="priority-badge" :class="backlogItem.priority">
                {{ backlogItem.priority }} Priority
              </span>
            </div>

            <!-- Create mode: form -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="submitForm">
              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="supplier-name">Supplier Name</label>
                  <input
                    id="supplier-name"
                    v-model="form.supplier_name"
                    type="text"
                    class="form-input"
                    placeholder="Enter supplier name"
                    required
                  />
                </div>
              </div>

              <div class="form-row two-cols">
                <div class="form-group">
                  <label class="form-label" for="quantity">
                    Quantity
                    <span class="label-hint">(shortage: {{ shortage }} units)</span>
                  </label>
                  <input
                    id="quantity"
                    v-model.number="form.quantity"
                    type="number"
                    class="form-input"
                    min="1"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="unit-cost">Unit Cost</label>
                  <input
                    id="unit-cost"
                    v-model.number="form.unit_cost"
                    type="number"
                    class="form-input"
                    min="0"
                    step="0.01"
                    placeholder="0.00"
                    required
                  />
                </div>
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="delivery-date">Expected Delivery Date</label>
                  <input
                    id="delivery-date"
                    v-model="form.expected_delivery_date"
                    type="date"
                    class="form-input"
                    required
                  />
                </div>
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="notes">Notes <span class="label-optional">(optional)</span></label>
                  <textarea
                    id="notes"
                    v-model="form.notes"
                    class="form-textarea"
                    rows="3"
                    placeholder="Add any additional notes..."
                  ></textarea>
                </div>
              </div>

              <div v-if="formError" class="form-error">{{ formError }}</div>
            </form>

            <!-- View mode: PO details -->
            <div v-else>
              <div v-if="poLoading" class="state-message">Loading purchase order...</div>
              <div v-else-if="poError" class="state-message error">{{ poError }}</div>
              <div v-else-if="existingPO" class="po-details">
                <div class="detail-grid">
                  <div class="detail-item">
                    <div class="detail-label">Supplier</div>
                    <div class="detail-value">{{ existingPO.supplier_name }}</div>
                  </div>
                  <div class="detail-item">
                    <div class="detail-label">Quantity</div>
                    <div class="detail-value">{{ existingPO.quantity }} units</div>
                  </div>
                  <div class="detail-item">
                    <div class="detail-label">Unit Cost</div>
                    <div class="detail-value">{{ formatCost(existingPO.unit_cost) }}</div>
                  </div>
                  <div class="detail-item">
                    <div class="detail-label">Total Cost</div>
                    <div class="detail-value total">{{ formatCost(existingPO.unit_cost * existingPO.quantity) }}</div>
                  </div>
                  <div class="detail-item">
                    <div class="detail-label">Status</div>
                    <div class="detail-value">
                      <span class="status-badge" :class="existingPO.status">{{ existingPO.status }}</span>
                    </div>
                  </div>
                  <div class="detail-item">
                    <div class="detail-label">Expected Delivery</div>
                    <div class="detail-value">{{ formatDate(existingPO.expected_delivery_date) }}</div>
                  </div>
                  <div class="detail-item">
                    <div class="detail-label">Created Date</div>
                    <div class="detail-value">{{ formatDate(existingPO.created_date) }}</div>
                  </div>
                  <div v-if="existingPO.notes" class="detail-item full-width">
                    <div class="detail-label">Notes</div>
                    <div class="detail-value notes">{{ existingPO.notes }}</div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">Close</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              :disabled="submitting"
              @click="submitForm"
            >
              {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'PurchaseOrderModal',

  props: {
    isOpen: {
      type: Boolean,
      default: false
    },
    backlogItem: {
      type: Object,
      default: null
    },
    // 'create' or 'view'
    mode: {
      type: String,
      default: 'create'
    }
  },

  emits: ['close', 'po-created'],

  setup(props, { emit }) {
    const { translateProductName } = useI18n()

    // Shortage = how many units still need to be procured
    const shortage = computed(() => {
      if (!props.backlogItem) return 0
      return props.backlogItem.quantity_needed - props.backlogItem.quantity_available
    })

    // --- Create mode state ---
    const form = ref({
      supplier_name: '',
      quantity: 0,
      unit_cost: null,
      expected_delivery_date: '',
      notes: ''
    })
    const submitting = ref(false)
    const formError = ref(null)

    // Pre-fill quantity with the shortage amount whenever the modal opens in create mode
    watch(
      () => [props.isOpen, props.backlogItem],
      ([isOpen]) => {
        if (isOpen && props.mode === 'create') {
          form.value = {
            supplier_name: '',
            quantity: shortage.value,
            unit_cost: null,
            expected_delivery_date: '',
            notes: ''
          }
          formError.value = null
        }
      }
    )

    const submitForm = async () => {
      if (!props.backlogItem) return
      submitting.value = true
      formError.value = null
      try {
        const payload = {
          backlog_item_id: props.backlogItem.id,
          supplier_name: form.value.supplier_name,
          quantity: form.value.quantity,
          unit_cost: form.value.unit_cost,
          expected_delivery_date: form.value.expected_delivery_date,
          notes: form.value.notes || null
        }
        const response = await api.createPurchaseOrder(payload)
        emit('po-created', response.data)
      } catch (err) {
        formError.value = 'Failed to create purchase order. Please try again.'
        console.error('PurchaseOrderModal: create failed', err)
      } finally {
        submitting.value = false
      }
    }

    // --- View mode state ---
    const existingPO = ref(null)
    const poLoading = ref(false)
    const poError = ref(null)

    // Fetch the existing PO when the modal opens in view mode
    watch(
      () => [props.isOpen, props.mode],
      async ([isOpen, mode]) => {
        if (isOpen && mode === 'view' && props.backlogItem) {
          poLoading.value = true
          poError.value = null
          existingPO.value = null
          try {
            const response = await api.getPurchaseOrderByBacklogItem(props.backlogItem.id)
            existingPO.value = response.data
          } catch (err) {
            poError.value = 'Failed to load purchase order details.'
            console.error('PurchaseOrderModal: fetch failed', err)
          } finally {
            poLoading.value = false
          }
        }
      }
    )

    const close = () => {
      emit('close')
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return 'N/A'
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    const formatCost = (value) => {
      if (value == null) return 'N/A'
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
    }

    return {
      shortage,
      form,
      submitting,
      formError,
      submitForm,
      existingPO,
      poLoading,
      poError,
      close,
      formatDate,
      formatCost,
      translateProductName
    }
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 600px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 2rem;
}

/* Item header */
.item-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1rem;
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.75rem;
}

.item-title-section {
  flex: 1;
  min-width: 0;
}

.item-name {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.375rem 0;
}

.item-sku {
  font-size: 0.875rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.priority-badge {
  padding: 0.375rem 0.875rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.priority-badge.high {
  background: #fecaca;
  color: #991b1b;
}

.priority-badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.priority-badge.low {
  background: #dbeafe;
  color: #1e40af;
}

/* Form */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: flex;
  flex-direction: column;
}

.form-row.two-cols {
  flex-direction: row;
  gap: 1rem;
}

.form-row.two-cols .form-group {
  flex: 1;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.label-hint {
  text-transform: none;
  letter-spacing: 0;
  font-weight: 400;
  color: #94a3b8;
}

.label-optional {
  text-transform: none;
  letter-spacing: 0;
  font-weight: 400;
  color: #94a3b8;
}

.form-input {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.9375rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  width: 100%;
  box-sizing: border-box;
}

.form-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-textarea {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.9375rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  resize: vertical;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  width: 100%;
  box-sizing: border-box;
}

.form-textarea:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-error {
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #dc2626;
}

/* View mode details */
.state-message {
  font-size: 0.9375rem;
  color: #64748b;
  text-align: center;
  padding: 2rem 0;
}

.state-message.error {
  color: #dc2626;
}

.detail-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.5rem;
}

.detail-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.detail-item.full-width {
  grid-column: 1 / -1;
}

.detail-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.detail-value {
  font-size: 0.9375rem;
  color: #0f172a;
  font-weight: 500;
}

.detail-value.total {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.detail-value.notes {
  color: #475569;
  font-weight: 400;
  line-height: 1.5;
  white-space: pre-wrap;
}

.status-badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: capitalize;
}

/* Map common PO status values to colours */
.status-badge.pending {
  background: #fef9c3;
  color: #854d0e;
}

.status-badge.confirmed,
.status-badge.approved {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.shipped,
.status-badge.in_transit {
  background: #ede9fe;
  color: #5b21b6;
}

.status-badge.delivered,
.status-badge.complete,
.status-badge.completed {
  background: #dcfce7;
  color: #15803d;
}

.status-badge.cancelled,
.status-badge.canceled {
  background: #f1f5f9;
  color: #64748b;
}

/* Footer */
.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.375rem;
  background: #2563eb;
  border: 1px solid #2563eb;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
  border-color: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modal transition animations — matches BacklogDetailModal */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
