<script setup>
import { DESIGNS, setDesign, ui } from '../store.js'

// Переключатель темы: светлая ↔ тёмная. Стоит в самом низу боковой панели (и в углу страницы входа).
defineProps({ floating: { type: Boolean, default: false } })
</script>

<template>
  <div class="design-switch" :class="{ floating }" role="radiogroup" aria-label="Тема интерфейса">
    <button v-for="d in DESIGNS" :key="d.key" type="button" role="radio" :aria-checked="ui.design === d.key"
      :class="{ on: ui.design === d.key }" @click="setDesign(d.key)">
      <svg v-if="d.key === 'light'" viewBox="0 0 24 24" aria-hidden="true"><circle cx="12" cy="12" r="4" /><path d="M12 2.5v2M12 19.5v2M4.6 4.6l1.4 1.4M18 18l1.4 1.4M2.5 12h2M19.5 12h2M4.6 19.4 6 18M18 6l1.4-1.4" /></svg>
      <svg v-else viewBox="0 0 24 24" aria-hidden="true"><path d="M20 14.5A8 8 0 0 1 9.5 4a8 8 0 1 0 10.5 10.5z" /></svg>
      {{ d.label }}
    </button>
  </div>
</template>

<style>
.design-switch { display: grid; grid-template-columns: 1fr 1fr; gap: 2px; padding: 3px; border: 1px solid var(--hair-2); border-radius: 12px; font-size: 12px; width: 100%; }
.design-switch button { display: inline-flex; align-items: center; justify-content: center; gap: 6px; font: inherit; border: 0; background: transparent;
  color: var(--ink-2, var(--ink)); padding: 7px 10px; border-radius: 9px; cursor: pointer; }
.design-switch svg { width: 14px; height: 14px; fill: none; stroke: currentColor; stroke-width: 1.8; stroke-linecap: round; }
.design-switch button.on { background: var(--accent); color: var(--accent-ink); font-weight: 600; }
.design-switch.floating { position: fixed; right: 20px; top: 20px; z-index: 50; width: auto; background: rgba(10, 14, 16, 0.55); border-color: rgba(236, 235, 230, 0.2);
  backdrop-filter: blur(12px); -webkit-backdrop-filter: blur(12px); }
.design-switch.floating button:not(.on) { color: rgba(236, 235, 230, 0.85); }
.design-switch.floating button.on { background: #ECEBE6; color: #17130D; }
</style>
