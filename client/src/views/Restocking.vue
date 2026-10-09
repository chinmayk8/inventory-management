<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Config row -->
      <div class="config-row">
        <div class="card config-card">
          <div class="config-label">Budget</div>
          <div class="slider-row">
            <input
              type="range"
              min="0"
              max="500000"
              step="1000"
              v-model.number="budget"
              class="budget-slider"
            />
            <span class="budget-value">${{ budget.toLocaleString() }}</span>
          </div>
        </div>

        <div class="card config-card">
          <div class="config-label">Warehouse</div>
          <select v-model="selectedWarehouse" class="warehouse-select">
            <option value="San Francisco">San Francisco</option>
            <option value="London">London</option>
            <option value="Tokyo">Tokyo</option>
          </select>
        </div>
      </div>

      <!-- Budget progress bar -->
      <div class="card budget-card">
        <div class="budget-summary">
          <span>Used: ${{ usedBudget.toLocaleString() }}</span>
          <span>Budget: ${{ budget.toLocaleString() }}</span>
          <span>Remaining: ${{ remainingBudget.toLocaleString() }}</span>
        </div>
        <div class="progress-track">
          <div class="progress-fill" :style="{ width: progressPercent + '%' }"></div>
        </div>
      </div>

      <!-- Success / Error messages -->
      <div v-if="successMessage" class="success-message">{{ successMessage }}</div>
      <div v-if="submitError" class="error-message">{{ submitError }}</div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ recommendedItems.length }} items)</h3>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          Adjust the budget slider to see restocking recommendations
        </div>
        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th class="col-num">Qty</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Line Total</th>
                <th>Category</th>
                <th class="col-num">Lead Time</th>
                <th>Trend</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.id">
                <td>{{ item.item_name }}</td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-num">${{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-num"><strong>${{ getLineTotal(item).toLocaleString() }}</strong></td>
                <td>{{ item.category }}</td>
                <td class="col-num">{{ getLeadTime(item.category) }}</td>
                <td>
                  <span :class="['badge', getTrendClass(item.trend)]">{{ item.trend }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Order summary footer -->
      <div class="card order-footer">
        <div class="footer-summary">
          <div class="footer-stat">
            <span class="footer-label">Total Items</span>
            <span class="footer-value">{{ recommendedItems.length }}</span>
          </div>
          <div class="footer-stat">
            <span class="footer-label">Total Cost</span>
            <span class="footer-value">${{ usedBudget.toLocaleString() }}</span>
          </div>
        </div>
        <button
          class="place-order-btn"
          :disabled="recommendedItems.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing Order...' : 'Place Order' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

const LEAD_TIME_DAYS = {
  'Power Supplies': 7,
  'Circuit Boards': 7,
  'Sensors': 10,
  'Controllers': 10,
  'Machinery': 21
}

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()
    const { selectedLocation, getCurrentFilters } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const allForecasts = ref([])
    const budget = ref(100000)
    const selectedWarehouse = ref('San Francisco')
    const submitting = ref(false)
    const successMessage = ref('')
    const submitError = ref('')

    const recommendedItems = computed(() => {
      const sorted = [...allForecasts.value].sort(
        (a, b) => b.forecasted_demand - a.forecasted_demand
      )
      let usedBudget = 0
      const result = []
      for (const item of sorted) {
        const itemCost = item.forecasted_demand * item.unit_cost
        if (usedBudget + itemCost <= budget.value) {
          result.push(item)
          usedBudget += itemCost
        }
      }
      return result
    })

    const usedBudget = computed(() => {
      return recommendedItems.value.reduce(
        (sum, item) => sum + item.forecasted_demand * item.unit_cost,
        0
      )
    })

    const remainingBudget = computed(() => budget.value - usedBudget.value)

    const progressPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, (usedBudget.value / budget.value) * 100)
    })

    const getLineTotal = (item) => item.forecasted_demand * item.unit_cost

    const getLeadTime = (category) => {
      const days = LEAD_TIME_DAYS[category] ?? 14
      return `${days} days`
    }

    const getTrendClass = (trend) => {
      const map = { increasing: 'success', stable: 'info', decreasing: 'danger' }
      return map[trend] || 'info'
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        allForecasts.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      submitError.value = ''
      try {
        const payload = {
          items: recommendedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.forecasted_demand,
            unit_cost: item.unit_cost,
            category: item.category
          })),
          warehouse: selectedWarehouse.value
        }
        await api.createRestockOrder(payload)
        successMessage.value = 'Restock order placed successfully.'
        setTimeout(() => {
          successMessage.value = ''
        }, 3000)
      } catch (err) {
        submitError.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => {
      if (selectedLocation.value !== 'all') {
        selectedWarehouse.value = selectedLocation.value
      }
      loadData()
    })

    return {
      t,
      loading,
      error,
      budget,
      selectedWarehouse,
      recommendedItems,
      usedBudget,
      remainingBudget,
      progressPercent,
      submitting,
      successMessage,
      submitError,
      getLineTotal,
      getLeadTime,
      getTrendClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
.config-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.5rem;
  margin-bottom: 1.5rem;
}

.config-card {
  padding: 1.25rem 1.5rem;
}

.config-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.75rem;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-slider {
  flex: 1;
  accent-color: #3b82f6;
  height: 4px;
  cursor: pointer;
}

.budget-value {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
  white-space: nowrap;
  min-width: 90px;
  text-align: right;
}

.warehouse-select {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  cursor: pointer;
}

.warehouse-select:focus {
  outline: none;
  border-color: #3b82f6;
}

.budget-card {
  padding: 1rem 1.5rem;
  margin-bottom: 1.5rem;
}

.budget-summary {
  display: flex;
  gap: 2rem;
  font-size: 0.875rem;
  color: #64748b;
  margin-bottom: 0.75rem;
}

.budget-summary span:first-child {
  color: #0f172a;
  font-weight: 600;
}

.progress-track {
  width: 100%;
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: #10b981;
  border-radius: 4px;
  transition: width 0.2s ease;
}

.success-message {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #6ee7b7;
  border-radius: 6px;
  padding: 0.75rem 1rem;
  margin-bottom: 1rem;
  font-size: 0.875rem;
  font-weight: 500;
}

.error-message {
  background: #fee2e2;
  color: #991b1b;
  border: 1px solid #fca5a5;
  border-radius: 6px;
  padding: 0.75rem 1rem;
  margin-bottom: 1rem;
  font-size: 0.875rem;
  font-weight: 500;
}

.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-num {
  width: 100px;
  text-align: right;
}

.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.9rem;
}

.order-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.25rem 1.5rem;
  margin-top: 1.5rem;
}

.footer-summary {
  display: flex;
  gap: 2.5rem;
}

.footer-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.footer-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.footer-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.place-order-btn {
  padding: 0.625rem 1.5rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #2563eb;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
