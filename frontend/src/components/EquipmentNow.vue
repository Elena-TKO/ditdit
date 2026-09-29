<script setup>
import { computed, onMounted, ref } from 'vue'
import { api } from '../api.js'

// «Техника на площадке»: компактная сводка по последним снимкам — сколько машин видно, работают или стоят.
const props = defineProps({ building: { type: Object, required: true } })
const data = ref(null)
const loaded = ref(false)
const items = computed(() => (data.value?.verdict?.observed ? data.value.verdict.equipment || [] : []))

async function load() {
  data.value = await api.analysis(props.building.id).catch(() => null)
  loaded.value = true
}
onMounted(load)
defineExpose({ load })
</script>

<template>
  <section class="panel eq-now">
    <div class="eq-head">
      <h2>Техника на площадке</h2>
      <span>по снимкам за {{ data?.verdict?.window_hours || 24 }} ч</span>
    </div>
    <ul v-if="items.length" class="eq-list">
      <li v-for="e in items" :key="e.cls">
        <b>{{ e.label }}</b>
        <span class="eq-n">{{ e.max_count }}</span>
        <span v-if="e.working" class="eq-state work">работает {{ e.working }}</span>
        <span v-if="e.idle" class="eq-state idle">стоит {{ e.idle }}</span>
        <span v-if="!e.working && !e.idle" class="eq-state">{{ e.photos }} сним.</span>
      </li>
    </ul>
    <p v-else class="eq-empty">{{ loaded ? 'Техника на последних снимках не найдена.' : 'Загружаем…' }}</p>
  </section>
</template>

<style>
.eq-now { padding: 18px 22px; }
.eq-head { display: flex; align-items: baseline; gap: 12px; margin-bottom: 12px; }
.eq-head h2 { font-size: 20px; margin: 0; }
.eq-head span, .eq-empty { color: var(--ink-3); font-size: 12px; margin: 0; }
.eq-list { list-style: none; margin: 0; padding: 0; display: flex; flex-wrap: wrap; gap: 8px; }
.eq-list li { display: inline-flex; align-items: center; gap: 8px; padding: 6px 10px; border-radius: 10px; background: var(--panel-2); border: 1px solid var(--hair); font-size: 13px; }
.eq-list b { font-weight: 600; }
.eq-n { min-width: 22px; text-align: center; font-weight: 700; color: var(--accent-ink); background: var(--accent); border-radius: 6px; padding: 0 6px; }
.eq-state { font-size: 11px; color: var(--ink-3); }
.eq-state.work { color: var(--ok); }
.eq-state.idle { color: var(--critical); }
</style>
