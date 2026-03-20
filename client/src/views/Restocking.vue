<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Budget-based restocking recommendations from demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Configuration Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget Configuration</h3>
        </div>
        <div class="budget-display">${{ budget.toLocaleString() }}</div>
        <input
          type="range"
          v-model.number="budget"
          :min="BUDGET_MIN"
          :max="BUDGET_MAX"
          :step="BUDGET_STEP"
          class="budget-slider"
        />
        <div class="budget-progress">
          <div
            class="budget-progress-bar"
            :style="{ width: budgetUtilization + '%' }"
          ></div>
        </div>
        <div class="budget-stats">
          <span>Allocated: ${{ totalAllocated.toLocaleString() }}</span>
          <span>Remaining: ${{ (budget - totalAllocated).toLocaleString() }}</span>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendations.length }})</h3>
          <button
            class="btn-primary"
            :disabled="submitting || recommendations.length === 0"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>

        <div v-if="successMessage" class="success-alert">
          {{ successMessage }}
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items fit within the current budget. Try increasing the budget.
        </div>
        <div v-else class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Forecast Qty</th>
                <th>Unit Cost</th>
                <th>Line Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="rec in recommendations" :key="rec.sku">
                <td><strong>{{ rec.sku }}</strong></td>
                <td>{{ rec.name }}</td>
                <td>
                  <span :class="['badge', getTrendClass(rec.trend)]">{{ rec.trend }}</span>
                </td>
                <td>{{ rec.forecasted_demand.toLocaleString() }}</td>
                <td>
                  ${{ rec.unit_cost.toLocaleString() }}
                  <span v-if="rec.unit_cost_is_estimated" class="badge warning">est.</span>
                </td>
                <td>${{ rec.line_cost.toLocaleString() }}</td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="totals-row">
                <td colspan="5">Total</td>
                <td>${{ totalAllocated.toLocaleString() }}</td>
              </tr>
            </tfoot>
          </table>
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
    const budget = ref(150000)
    const BUDGET_MIN = 10000
    const BUDGET_MAX = 500000
    const BUDGET_STEP = 5000
    const FALLBACK_UNIT_COST = 50.0

    const forecasts = ref([])
    const inventory = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref(null)

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        // Load forecasts and inventory in parallel; restocking uses unfiltered global data
        const [forecastData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastData
        inventory.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const recommendations = computed(() => {
      // Build lookup map from inventory by SKU
      const inventoryBySku = {}
      inventory.value.forEach(item => {
        inventoryBySku[item.sku] = item
      })

      // Sort by trend priority: increasing=1, stable=2, decreasing=3
      // Ties broken by highest forecasted_demand first
      const trendPriority = { increasing: 1, stable: 2, decreasing: 3 }
      const sorted = [...forecasts.value].sort((a, b) => {
        const priorityDiff =
          (trendPriority[a.trend] || 99) - (trendPriority[b.trend] || 99)
        if (priorityDiff !== 0) return priorityDiff
        return b.forecasted_demand - a.forecasted_demand
      })

      // Greedy fill within budget — no partial fills
      let remaining = budget.value
      const result = []

      for (const forecast of sorted) {
        const inventoryItem = inventoryBySku[forecast.item_sku]
        const unitCostIsEstimated = !inventoryItem
        const unitCost = inventoryItem ? inventoryItem.unit_cost : FALLBACK_UNIT_COST
        const lineCost = unitCost * forecast.forecasted_demand

        if (lineCost <= remaining) {
          remaining -= lineCost
          result.push({
            sku: forecast.item_sku,
            name: forecast.item_name,
            trend: forecast.trend,
            forecasted_demand: forecast.forecasted_demand,
            unit_cost: unitCost,
            line_cost: lineCost,
            unit_cost_is_estimated: unitCostIsEstimated
          })
        }
        // Skip (don't partially fill) if too expensive
      }

      return result
    })

    const totalAllocated = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.line_cost, 0)
    )

    const budgetUtilization = computed(() =>
      Math.min((totalAllocated.value / budget.value) * 100, 100)
    )

    const getTrendClass = (trend) => {
      const map = { increasing: 'success', stable: 'info', decreasing: 'danger' }
      return map[trend] || 'info'
    }

    const placeOrder = async () => {
      if (submitting.value || recommendations.value.length === 0) return
      submitting.value = true
      successMessage.value = null
      try {
        const payload = {
          customer: 'Restocking System',
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.forecasted_demand,
            unit_price: r.unit_cost
          })),
          warehouse: null,
          category: null
        }
        await api.createOrder(payload)
        successMessage.value = 'Order placed successfully!'
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadData())

    return {
      budget,
      BUDGET_MIN,
      BUDGET_MAX,
      BUDGET_STEP,
      forecasts,
      inventory,
      loading,
      error,
      submitting,
      successMessage,
      recommendations,
      totalAllocated,
      budgetUtilization,
      getTrendClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  margin-top: 0.75rem;
  margin-bottom: 0.75rem;
}

.budget-progress {
  background: #e2e8f0;
  border-radius: 9999px;
  height: 8px;
  overflow: hidden;
  margin: 0.75rem 0;
}

.budget-progress-bar {
  background: #2563eb;
  height: 100%;
  transition: width 0.3s ease;
}

.budget-stats {
  display: flex;
  gap: 2rem;
  margin-top: 0.75rem;
  font-size: 0.875rem;
  color: #64748b;
}

.btn-primary {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.totals-row td {
  font-weight: 700;
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
}

.success-alert {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.75rem 1rem;
  border-radius: 6px;
  margin-bottom: 1rem;
}

.recommendations-table {
  width: 100%;
}
</style>
