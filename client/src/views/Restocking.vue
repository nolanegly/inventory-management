<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your available budget and review demand-based restock recommendations.</p>
    </div>

    <!-- Budget Slider Card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
        <span class="budget-display">{{ formatCurrency(budget) }}</span>
      </div>
      <div class="slider-row">
        <span class="slider-label">{{ formatCurrency(BUDGET_MIN) }}</span>
        <input
          type="range"
          class="budget-slider"
          :min="BUDGET_MIN"
          :max="BUDGET_MAX"
          :step="BUDGET_STEP"
          v-model.number="budget"
          @change="loadRecommendations"
        />
        <span class="slider-label">{{ formatCurrency(BUDGET_MAX) }}</span>
      </div>
    </div>

    <!-- Success Banner -->
    <div v-if="orderSuccess" class="success-banner">
      <div class="success-banner-content">
        <div class="success-title">Order Placed Successfully</div>
        <div class="success-detail">Order <strong>{{ orderResult.order_number }}</strong> confirmed. Expected delivery: <strong>{{ orderResult.expected_delivery }}</strong>.</div>
      </div>
      <button class="btn-secondary" @click="resetOrder">Start New Order</button>
    </div>

    <!-- Recommendations -->
    <div v-if="!orderSuccess">
      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else>
        <div class="card">
          <div class="card-header">
            <h3 class="card-title">Recommended Items</h3>
            <span v-if="recommendations.length > 0" class="item-count">{{ recommendations.length }} item{{ recommendations.length !== 1 ? 's' : '' }}</span>
          </div>

          <div v-if="recommendations.length === 0" class="empty-state">
            No items fit within the selected budget. Try increasing the budget.
          </div>

          <div v-else class="table-container">
            <table>
              <thead>
                <tr>
                  <th>Item</th>
                  <th>Demand Gap</th>
                  <th>Unit Cost</th>
                  <th>Qty to Order</th>
                  <th>Est. Cost</th>
                  <th>Lead Time</th>
                  <th>Trend</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendations" :key="item.item_sku">
                  <td>
                    <div class="item-name">{{ item.item_name }}</div>
                    <div class="item-sku">{{ item.item_sku }}</div>
                  </td>
                  <td>
                    <div class="demand-gap">
                      <span class="gap-value">{{ item.demand_gap }}</span>
                      <span class="gap-detail">{{ item.current_demand }} current / {{ item.forecasted_demand }} forecasted</span>
                    </div>
                  </td>
                  <td>{{ formatCurrency(item.unit_cost) }}</td>
                  <td><strong>{{ item.quantity_recommended }}</strong></td>
                  <td>{{ formatCurrency(item.estimated_cost) }}</td>
                  <td>
                    <span :class="['badge', leadTimeBadgeClass(item.lead_time_days)]">
                      {{ item.lead_time_days }}d
                    </span>
                  </td>
                  <td>
                    <span :class="['badge', item.trend]">
                      {{ item.trend }}
                    </span>
                  </td>
                </tr>
              </tbody>
              <tfoot>
                <tr class="summary-row">
                  <td colspan="3"><strong>Total</strong></td>
                  <td><strong>{{ totalQuantity }}</strong></td>
                  <td><strong>{{ formatCurrency(totalEstimatedCost) }}</strong></td>
                  <td colspan="2">
                    <span :class="['badge', budgetRemaining >= 0 ? 'success' : 'danger']">
                      {{ formatCurrency(Math.abs(budgetRemaining)) }} {{ budgetRemaining >= 0 ? 'remaining' : 'over budget' }}
                    </span>
                  </td>
                </tr>
              </tfoot>
            </table>
          </div>
        </div>

        <!-- Order Error -->
        <div v-if="orderError" class="error">{{ orderError }}</div>

        <!-- Place Order Button -->
        <div class="action-row">
          <button
            class="btn-primary"
            :disabled="recommendations.length === 0 || placing"
            @click="placeOrder"
          >
            {{ placing ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const BUDGET_MIN = 10000
    const BUDGET_MAX = 500000
    const BUDGET_STEP = 5000

    const budget = ref(50000)
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const placing = ref(false)
    const orderError = ref(null)
    const orderSuccess = ref(false)
    const orderResult = ref(null)

    const totalEstimatedCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.estimated_cost, 0)
    )

    const totalQuantity = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.quantity_recommended, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalEstimatedCost.value)

    const formatCurrency = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })

    const leadTimeBadgeClass = (days) => {
      if (days <= 7) return 'success'
      if (days <= 14) return 'warning'
      return 'info'
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations(budget.value)
        recommendations.value = data
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      placing.value = true
      orderError.value = null
      try {
        const payload = {
          budget: budget.value,
          items: recommendations.value.map((item) => ({
            item_sku: item.item_sku,
            item_name: item.item_name,
            quantity: item.quantity_recommended,
            unit_cost: item.unit_cost,
            lead_time_days: item.lead_time_days
          }))
        }
        const result = await api.placeRestockingOrder(payload)
        orderResult.value = result
        orderSuccess.value = true
      } catch (err) {
        orderError.value = 'Failed to place order. Please try again.'
        console.error(err)
      } finally {
        placing.value = false
      }
    }

    const resetOrder = () => {
      orderSuccess.value = false
      orderResult.value = null
      orderError.value = null
      recommendations.value = []
      budget.value = 50000
      loadRecommendations()
    }

    onMounted(loadRecommendations)

    return {
      BUDGET_MIN,
      BUDGET_MAX,
      BUDGET_STEP,
      budget,
      recommendations,
      loading,
      error,
      placing,
      orderError,
      orderSuccess,
      orderResult,
      totalEstimatedCost,
      totalQuantity,
      budgetRemaining,
      formatCurrency,
      leadTimeBadgeClass,
      loadRecommendations,
      placeOrder,
      resetOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Budget slider */
.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.25rem 0 0.5rem;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  height: 6px;
  appearance: none;
  -webkit-appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 20px;
  height: 20px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.15);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

/* Item cell */
.item-name {
  font-weight: 600;
  color: #0f172a;
  font-size: 0.875rem;
}

.item-sku {
  font-size: 0.75rem;
  color: #64748b;
  margin-top: 2px;
}

/* Demand gap cell */
.demand-gap {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.gap-value {
  font-weight: 700;
  color: #0f172a;
}

.gap-detail {
  font-size: 0.75rem;
  color: #64748b;
}

/* Summary row */
.summary-row {
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
}

.summary-row td {
  padding: 0.75rem;
  color: #0f172a;
  font-size: 0.875rem;
}

/* Item count badge in header */
.item-count {
  font-size: 0.813rem;
  font-weight: 600;
  color: #64748b;
  background: #f1f5f9;
  padding: 0.25rem 0.625rem;
  border-radius: 6px;
}

/* Empty state */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Action row */
.action-row {
  display: flex;
  justify-content: flex-end;
  margin-top: 1rem;
}

/* Buttons */
.btn-primary {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.btn-secondary {
  padding: 0.625rem 1.5rem;
  background: #fff;
  color: #0f172a;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, border-color 0.15s ease;
}

.btn-secondary:hover {
  background: #f8fafc;
  border-color: #cbd5e1;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 10px;
  padding: 1.25rem 1.5rem;
  margin-bottom: 1.25rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.success-title {
  font-size: 1rem;
  font-weight: 700;
  color: #065f46;
  margin-bottom: 0.25rem;
}

.success-detail {
  font-size: 0.875rem;
  color: #047857;
}
</style>
