<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="card">
      <div class="budget-control">
        <label class="budget-label" for="budget-slider">{{ t('restocking.budgetLabel') }}</label>
        <input
          id="budget-slider"
          v-model.number="budget"
          type="range"
          min="0"
          max="50000"
          step="250"
          class="budget-slider"
        />
        <span class="budget-value">{{ formatCurrency(budget, currentCurrency) }}</span>
      </div>
    </div>

    <div v-if="submitSuccess" class="success-banner">{{ t('restocking.successMessage') }}</div>
    <div v-if="submitError" class="error">{{ submitError }}</div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">{{ formatCurrency(recommendation?.total_cost ?? 0, currentCurrency) }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">{{ formatCurrency(recommendation?.remaining_budget ?? 0, currentCurrency) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.itemsSelected') }}</div>
          <div class="stat-value">{{ recommendation?.recommended_items?.length ?? 0 }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
          <button
            class="btn-primary"
            :disabled="loading || submitting || !recommendation || recommendation.recommended_items.length === 0"
            @click="openConfirm"
          >
            {{ t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="!recommendation || recommendation.recommended_items.length === 0" class="empty-state">
          {{ t('restocking.emptyState') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.category') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.urgency') }}</th>
                <th>{{ t('restocking.table.quantity') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendation.recommended_items" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.category }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ t(`trends.${item.trend}`) }}</span>
                </td>
                <td>
                  <span :class="['badge', item.urgency]">{{ t(`priority.${item.urgency}`) }}</span>
                </td>
                <td>{{ item.quantity }}</td>
                <td>{{ formatCurrency(item.unit_cost, currentCurrency) }}</td>
                <td><strong>{{ formatCurrency(item.line_total, currentCurrency) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <ConfirmRestockModal
      :is-open="showConfirmModal"
      :items="recommendation?.recommended_items ?? []"
      :total-cost="recommendation?.total_cost ?? 0"
      :lead-time-days="recommendation?.lead_time_days ?? 0"
      @close="showConfirmModal = false"
      @confirm="confirmOrder"
    />
  </div>
</template>

<script>
import { ref, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'
import ConfirmRestockModal from '../components/ConfirmRestockModal.vue'

export default {
  name: 'Restocking',
  components: {
    ConfirmRestockModal
  },
  setup() {
    const { t, currentCurrency } = useI18n()

    const budget = ref(2000)
    const recommendation = ref(null)
    const loading = ref(false)
    const error = ref(null)

    const showConfirmModal = ref(false)
    const submitting = ref(false)
    const submitError = ref(null)
    const submitSuccess = ref(false)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendation.value = await api.getRestockRecommendations(budget.value)
      } catch (err) {
        error.value = 'Failed to load restock recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    let debounceTimer = null
    watch(budget, () => {
      submitSuccess.value = false
      if (debounceTimer) clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 300)
    })

    const openConfirm = () => {
      submitError.value = null
      showConfirmModal.value = true
    }

    const confirmOrder = async () => {
      if (!recommendation.value) return
      try {
        submitting.value = true
        submitError.value = null
        await api.submitRestockOrder({
          budget: budget.value,
          items: recommendation.value.recommended_items
        })
        submitSuccess.value = true
        showConfirmModal.value = false
        await loadRecommendations()
      } catch (err) {
        submitError.value = t('restocking.errorMessage') + ': ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      t,
      currentCurrency,
      budget,
      recommendation,
      loading,
      error,
      showConfirmModal,
      submitting,
      submitError,
      submitSuccess,
      openConfirm,
      confirmOrder,
      formatCurrency
    }
  }
}
</script>

<style scoped>
.budget-control {
  display: flex;
  align-items: center;
  gap: 1.25rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  flex-shrink: 0;
}

.budget-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 120px;
  text-align: right;
  flex-shrink: 0;
}

.budget-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 3px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
  transition: background 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #1d4ed8;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 3px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
  transition: background 0.15s ease;
}

.budget-slider::-moz-range-thumb:hover {
  background: #1d4ed8;
}

.budget-slider::-moz-range-track {
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  border: 1px solid #2563eb;
  border-radius: 8px;
  font-weight: 500;
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
  background: #cbd5e1;
  border-color: #cbd5e1;
  cursor: not-allowed;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

.success-banner {
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
