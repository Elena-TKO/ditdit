<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import { api } from '../api.js'

// Исходные данные расчёта техники (свёрнуты по умолчанию). Расчёт запускается сразу с типовыми значениями,
// наружу отдаётся только итог — сколько машин нужно; формулы — в METHODOLOGY.md.
const props = defineProps({ kind: { type: String, required: true } })
const emit = defineEmits(['result'])

const FIELDS = {
  excavation: [
    { key: 'volume_m3', label: 'Объём грунта, м³', value: 20000 },
    { key: 'days', label: 'Срок, сут', value: 20 },
    { key: 'shifts', label: 'Смен в сутки', value: 1 },
    { key: 'bucket_m3', label: 'Ковш, м³', value: 1 },
    { key: 'truck_t', label: 'Самосвал, т', value: 20 },
    { key: 'distance_km', label: 'Плечо вывоза, км', value: 10 },
  ],
  concreting: [
    { key: 'volume_m3', label: 'Объём бетона, м³', value: 480 },
    { key: 'hours', label: 'Время укладки, ч', value: 16 },
    { key: 'pump_m3h', label: 'Насос, м³/ч', value: 30 },
    { key: 'mixer_m3', label: 'Миксер, м³', value: 7 },
    { key: 'distance_km', label: 'До бетонного узла, км', value: 10 },
  ],
  tower_crane: [
    { key: 'building_height_m', label: 'Высота здания, м', value: 75 },
    { key: 'building_width_m', label: 'Ширина, м', value: 18 },
    { key: 'building_length_m', label: 'Длина, м', value: 120 },
    { key: 'element_t', label: 'Самый тяжёлый элемент, т', value: 3 },
  ],
}

const params = ref({})
const error = ref('')
const busy = ref(false)
const fields = computed(() => FIELDS[props.kind] || [])

async function run() {
  busy.value = true
  error.value = ''
  try {
    emit('result', await api.methodologyCalc(props.kind, params.value))
  } catch (e) {
    error.value = e.message
    emit('result', null)
  } finally {
    busy.value = false
  }
}

function reset() {
  params.value = Object.fromEntries(fields.value.map((f) => [f.key, f.value]))
}

watch(() => props.kind, () => { reset(); run() })
onMounted(() => { reset(); run() })
</script>

<template>
  <details class="sc">
    <summary>Исходные данные расчёта</summary>
    <form class="sc-fields" @submit.prevent="run">
      <label v-for="f in fields" :key="f.key">{{ f.label }}<input v-model.number="params[f.key]" type="number" min="0" step="any" required /></label>
      <button class="btn small primary" type="submit" :disabled="busy">{{ busy ? 'Считаем…' : 'Пересчитать' }}</button>
    </form>
    <p v-if="error" class="error" role="alert">{{ error }}</p>
  </details>
</template>

<style>
.sc { font-size: 12px; }
.sc summary { cursor: pointer; color: var(--accent); font-weight: 600; list-style: none; }
.sc summary::-webkit-details-marker { display: none; }
.sc summary::before { content: "▸ "; }
.sc[open] summary::before { content: "▾ "; }
.sc-fields { display: grid; grid-template-columns: repeat(auto-fill, minmax(130px, 1fr)); gap: 10px; align-items: end; margin-top: 10px; }
.sc-fields label { display: grid; gap: 4px; font-size: 11px; color: var(--ink-3); }
.sc-fields input { font: inherit; font-size: 12px; padding: 6px 8px; }
</style>
