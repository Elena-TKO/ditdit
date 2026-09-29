<script setup>
import { ref } from 'vue'
import { api } from '../api.js'
import Icon from './Icon.vue'
import StagesOverview from './StagesOverview.vue'

// План работ: Гант «план ↔ факт по снимкам» и таблица «Все работы» по этапам.
const props = defineProps({ building: { type: Object, required: true } })
const emit = defineEmits(['changed'])
const error = ref('')
const busy = ref(false)
const overview = ref(null)

async function update(reset = false) {
  if (reset && !window.confirm('Сбросить изменения календарного плана к исходному справочнику?')) return
  busy.value = true
  error.value = ''
  try {
    if (reset) await api.regenerate(props.building.id)
    else await api.reschedule(props.building.id)
    await overview.value?.load()
  } catch (e) {
    error.value = e.message
  } finally {
    busy.value = false
  }
}

function changed() {
  emit('changed')
}
</script>

<template>
  <section class="work-plan">
    <header class="review-heading">
      <div><h1>План работ</h1><p>{{ building.name }} · план этапов и факт по снимкам</p></div>
      <div class="review-actions">
        <button class="btn" type="button" :disabled="busy" @click="update(true)">Сбросить</button>
        <button class="btn primary" type="button" :disabled="busy" @click="update(false)"><Icon name="refresh" />Пересчитать</button>
      </div>
    </header>
    <p v-if="error" class="error" role="alert">{{ error }}</p>
    <StagesOverview ref="overview" :building="building" @changed="changed" />
  </section>
</template>

<style>
.work-plan > .so { margin-bottom: 24px; }
</style>
