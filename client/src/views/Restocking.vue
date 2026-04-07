<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.budget') }}</h3>
      </div>
      <div class="budget-section">
        <div class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
        <input
          type="range"
          class="budget-slider"
          :min="1000"
          :max="50000"
          :step="500"
          v-model.number="budget"
        />
        <div class="budget-range">
          <span>{{ currencySymbol }}1,000</span>
          <span>{{ currencySymbol }}50,000</span>
        </div>
      </div>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.itemsToRestock') }}</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.budgetRemaining') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ budgetRemaining.toLocaleString() }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ recommendations.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="orderPlaced || recommendations.length === 0 || placingOrder"
            @click="placeOrder"
          >
            {{ placingOrder ? t('common.loading') : t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="orderPlaced" class="success-message">
          <strong>{{ t('restocking.orderPlaced') }}</strong>
          <span>&nbsp;{{ t('restocking.orderNumber') }}: {{ orderResult.order_number }}</span>
          <span>&nbsp;&mdash;&nbsp;{{ t('restocking.expectedDelivery') }}: {{ orderResult.expected_delivery }}</span>
        </div>

        <div v-if="recommendations.length === 0" style="padding: 3rem; text-align: center; color: #64748b;">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.name') }}</th>
                <th>{{ t('restocking.table.category') }}</th>
                <th>{{ t('restocking.table.warehouse') }}</th>
                <th>{{ t('restocking.table.onHand') }}</th>
                <th>{{ t('restocking.table.shortage') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.qtyToOrder') }}</th>
                <th>{{ t('restocking.table.lineCost') }}</th>
                <th>{{ t('restocking.table.source') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku + '-' + item.warehouse">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.category }}</td>
                <td>{{ item.warehouse }}</td>
                <td>{{ item.quantity_on_hand }}</td>
                <td>{{ item.shortage }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td>{{ item.recommended_quantity }}</td>
                <td>{{ currencySymbol }}{{ item.line_cost.toLocaleString() }}</td>
                <td>
                  <span :class="['badge', item.source === 'forecast' ? 'info' : 'warning']">
                    {{ item.source === 'forecast' ? t('restocking.source.forecast') : t('restocking.source.reorder_point') }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const budget = ref(10000)
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const orderPlaced = ref(false)
    const placingOrder = ref(false)
    const orderResult = ref(null)

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.line_cost, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalCost.value)

    let debounceTimer = null

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations(budget.value)
        recommendations.value = data
      } catch (err) {
        error.value = 'Failed to load restocking recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (orderPlaced.value || recommendations.value.length === 0) return
      placingOrder.value = true
      error.value = null
      try {
        const items = recommendations.value.map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.recommended_quantity,
          unit_cost: item.unit_cost
        }))
        const result = await api.createRestockingOrder({
          items,
          total_budget: budget.value
        })
        orderResult.value = result
        orderPlaced.value = true
      } catch (err) {
        error.value = 'Failed to place restocking order'
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    watch(budget, () => {
      orderPlaced.value = false
      orderResult.value = null
      if (debounceTimer) clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 300)
    })

    onMounted(() => loadRecommendations())

    return {
      t,
      currencySymbol,
      budget,
      recommendations,
      loading,
      error,
      orderPlaced,
      placingOrder,
      orderResult,
      totalCost,
      budgetRemaining,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
  padding: 1.5rem 2rem 2rem;
}

.budget-display {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

.budget-range {
  display: flex;
  justify-content: space-between;
  width: 100%;
  color: #64748b;
  font-size: 0.813rem;
}

.place-order-btn {
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.5rem;
  font-weight: 600;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background 0.15s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-message {
  background: #d1fae5;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin: 0 1.5rem 1rem;
  font-size: 0.9rem;
}
</style>
