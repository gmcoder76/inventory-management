<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Optimize your inventory spend based on demand forecasts</p>
    </div>

    <!-- Success banner -->
    <div v-if="successMessage" class="banner banner-success">
      <span>{{ successMessage }}</span>
      <button class="banner-close" @click="successMessage = null">&times;</button>
    </div>

    <!-- Error banner -->
    <div v-if="submitError" class="banner banner-error">
      <span>{{ submitError }}</span>
      <button class="banner-close" @click="submitError = null">&times;</button>
    </div>

    <div v-if="loading" class="loading">Loading demand and inventory data...</div>
    <div v-else-if="loadError" class="error">{{ loadError }}</div>
    <div v-else>

      <!-- Budget section -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
          <span class="budget-value">${{ budget.toLocaleString() }}</span>
        </div>
        <div class="budget-controls">
          <div class="slider-row">
            <span class="slider-label">$1,000</span>
            <input
              type="range"
              class="budget-slider"
              min="1000"
              max="100000"
              step="1000"
              v-model.number="budget"
            />
            <span class="slider-label">$100,000</span>
          </div>
        </div>

        <div class="usage-row">
          <span class="usage-label">
            Using
            <strong>${{ totalCost.toLocaleString() }}</strong>
            of
            <strong>${{ budget.toLocaleString() }}</strong>
            ({{ usagePct }}%)
          </span>
        </div>
        <div class="usage-bar-track">
          <div
            class="usage-bar-fill"
            :class="usageColorClass"
            :style="{ width: Math.min(usagePct, 100) + '%' }"
          ></div>
        </div>
      </div>

      <!-- Recommended items section -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restocking ({{ recommendedItems.length }} items)</h3>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          No items match the current budget. Try increasing the budget.
        </div>
        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>Priority</th>
                <th>Item Name</th>
                <th>Category</th>
                <th>Trend</th>
                <th>Current Stock / Reorder Point</th>
                <th>Qty to Restock</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td>
                  <span class="priority-dot" :class="priorityDotClass(item.score)" :title="priorityLabel(item.score)"></span>
                </td>
                <td>{{ item.name }}</td>
                <td>{{ item.category }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ capitalize(item.trend) }}</span>
                </td>
                <td class="stock-cell">
                  <span :class="{ 'low-stock': item.isLowStock }">{{ item.quantity_on_hand }}</span>
                  <span class="muted"> / {{ item.reorder_point }}</span>
                </td>
                <td><strong>{{ item.qty }}</strong></td>
                <td>${{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>${{ item.total_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place order button -->
      <div class="action-row">
        <button
          class="btn-primary"
          :disabled="recommendedItems.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing Order...' : 'Place Restocking Order' }}
        </button>
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
    const loading = ref(true)
    const loadError = ref(null)
    const submitting = ref(false)
    const successMessage = ref(null)
    const submitError = ref(null)

    const demandForecasts = ref([])
    const inventoryItems = ref([])

    // Budget slider — default $25,000
    const budget = ref(25000)

    // Load both data sources in parallel
    const loadData = async () => {
      loading.value = true
      loadError.value = null
      try {
        const [demand, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = demand
        inventoryItems.value = inventory
      } catch (err) {
        loadError.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Recommendation algorithm — recalculates whenever budget changes
    const recommendedItems = computed(() => {
      // Build lookup map: sku → inventory item
      const inventoryBySku = {}
      for (const inv of inventoryItems.value) {
        inventoryBySku[inv.sku] = inv
      }

      const scoreable = []

      for (const forecast of demandForecasts.value) {
        const inv = inventoryBySku[forecast.item_sku]

        // Skip items with no matching inventory (no unit cost available)
        if (!inv) continue

        // Trend score: increasing → 3, stable → 2, decreasing → 1
        const trendScore = { increasing: 3, stable: 2, decreasing: 1 }[forecast.trend] ?? 1
        const isLowStock = inv.quantity_on_hand <= inv.reorder_point

        // Multiply by 2 if low stock
        const score = isLowStock ? trendScore * 2 : trendScore

        // Quantity to restock
        const qty = Math.max(
          inv.reorder_point - inv.quantity_on_hand,
          Math.ceil(forecast.forecasted_demand * 0.5)
        )

        const unit_cost = inv.unit_cost
        const total_cost = qty * unit_cost

        scoreable.push({
          sku: forecast.item_sku,
          name: forecast.item_name,
          category: inv.category,
          trend: forecast.trend,
          score,
          isLowStock,
          qty,
          unit_cost,
          total_cost,
          quantity_on_hand: inv.quantity_on_hand,
          reorder_point: inv.reorder_point
        })
      }

      // Sort by score descending
      scoreable.sort((a, b) => b.score - a.score)

      // Greedy fill: include item if qty * unit_cost fits remaining budget
      let remaining = budget.value
      const included = []
      for (const item of scoreable) {
        if (item.total_cost <= remaining) {
          included.push(item)
          remaining -= item.total_cost
        }
      }

      return included
    })

    const totalCost = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const usagePct = computed(() => {
      if (budget.value === 0) return 0
      return parseFloat(((totalCost.value / budget.value) * 100).toFixed(1))
    })

    const usageColorClass = computed(() => {
      if (usagePct.value > 90) return 'bar-red'
      if (usagePct.value > 70) return 'bar-yellow'
      return 'bar-green'
    })

    const priorityDotClass = (score) => {
      if (score >= 4) return 'dot-high'
      if (score >= 2) return 'dot-medium'
      return 'dot-low'
    }

    const priorityLabel = (score) => {
      if (score >= 4) return 'High priority'
      if (score >= 2) return 'Medium priority'
      return 'Low priority'
    }

    const capitalize = (str) => {
      if (!str) return ''
      return str.charAt(0).toUpperCase() + str.slice(1)
    }

    // Auto-dismiss success banner after 6 seconds
    let dismissTimer = null
    const scheduleDismiss = () => {
      if (dismissTimer) clearTimeout(dismissTimer)
      dismissTimer = setTimeout(() => {
        successMessage.value = null
      }, 6000)
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      submitError.value = null
      successMessage.value = null

      try {
        const items = recommendedItems.value.map(item => ({
          sku: item.sku,
          name: item.name,
          category: item.category,
          quantity: item.qty,
          unit_cost: item.unit_cost
        }))

        const result = await api.createRestockingOrder({
          items,
          budget: budget.value
        })

        successMessage.value =
          `Restocking order submitted! Order #${result.order_number} placed for ${result.items.length} items totaling $${result.total_cost.toLocaleString()}. Check the Orders tab to track it.`

        scheduleDismiss()

        // Reload data so the recommended list reflects a fresh state
        await loadData()
      } catch (err) {
        submitError.value = 'Failed to submit restocking order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      loadError,
      submitting,
      successMessage,
      submitError,
      budget,
      recommendedItems,
      totalCost,
      usagePct,
      usageColorClass,
      priorityDotClass,
      priorityLabel,
      capitalize,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

/* Banners */
.banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.banner-success {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
}

.banner-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
}

.banner-close {
  background: none;
  border: none;
  font-size: 1.25rem;
  cursor: pointer;
  color: inherit;
  opacity: 0.7;
  padding: 0 0.25rem;
  line-height: 1;
}

.banner-close:hover {
  opacity: 1;
}

/* Budget card */
.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-controls {
  margin-bottom: 1rem;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  height: 6px;
  cursor: pointer;
  accent-color: #2563eb;
}

.usage-row {
  margin-bottom: 0.5rem;
  font-size: 0.875rem;
  color: #64748b;
}

.usage-bar-track {
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}

.usage-bar-fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.3s ease, background-color 0.3s ease;
}

.bar-green { background: #059669; }
.bar-yellow { background: #d97706; }
.bar-red { background: #dc2626; }

/* Table */
.restock-table {
  table-layout: auto;
  width: 100%;
}

.stock-cell {
  white-space: nowrap;
}

.low-stock {
  color: #dc2626;
  font-weight: 600;
}

.muted {
  color: #94a3b8;
}

/* Priority dots */
.priority-dot {
  display: inline-block;
  width: 10px;
  height: 10px;
  border-radius: 50%;
}

.dot-high { background: #dc2626; }
.dot-medium { background: #d97706; }
.dot-low { background: #2563eb; }

/* Empty state */
.empty-state {
  text-align: center;
  padding: 2.5rem 1rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* Action row */
.action-row {
  display: flex;
  justify-content: flex-end;
  margin-top: 0.5rem;
  margin-bottom: 1.5rem;
}

.btn-primary {
  padding: 0.75rem 2rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
