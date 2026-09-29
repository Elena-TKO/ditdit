<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import { api, imageUrl } from '../api.js'
import { refreshNav, ui } from '../store.js'
import { countLabel, formatDate, formatDateTime, toLocalInput } from '../lib/format.js'
import Icon from '../components/Icon.vue'
import PhotoInspector from '../components/PhotoInspector.vue'
import StageCalc from '../components/StageCalc.vue'

// Страница этапа — только главное: сколько техники нужно (по нормам и по расчёту) и сколько по факту,
// и снимки этапа. Перенос снимка на другой этап и разметка — в окне снимка.
const props = defineProps({ id: { type: Number, required: true }, phase: { type: String, required: true } })

const data = ref(null)
const building = ref(null)
const classes = ref([])
const error = ref('')
const openId = ref(null)
const calc = ref(null)
const showUpload = ref(false)
const files = ref([])
const takenAt = ref('')
const uploading = ref(false)
const uploadMsg = ref('')
const fileInput = ref(null)

const STATE = { ok: 'ok', short: 'warn', excess: 'warn', unexpected: 'bad', missing: 'bad' }
const range = (lo, hi) => (hi === null || hi === undefined ? `от ${lo}` : lo === hi ? `${lo}` : `${lo}–${hi}`)
const stage = computed(() => data.value?.stage || {})
const rows = computed(() => (data.value?.check || []).map((c) => ({
  ...c,
  label: c.labels.join(' или '),
  norm: c.role === 'unexpected' ? '—' : range(c.min_count, c.max_count),
  calc: c.classes.map((cls) => calc.value?.result?.[cls]).find((v) => v !== undefined) ?? null,
})))
const crane = computed(() => calc.value?.kind === 'tower_crane' ? calc.value.result : null)

async function load() {
  error.value = ''
  try {
    const [d, b] = await Promise.all([api.stage(props.id, props.phase), api.building(props.id)])
    data.value = d
    building.value = b
    Object.assign(ui, { projectId: b.project.id, buildingId: props.id })
    if (!takenAt.value) {
      takenAt.value = toLocalInput(d.stage.photo_end ? `${d.stage.photo_end}T12:00` : d.stage.start ? `${d.stage.start}T09:00` : undefined)
    }
  } catch (e) {
    error.value = e.message
  }
}

function pick(event) {
  const list = [...(event.target.files || event.dataTransfer?.files || [])]
  files.value = list.filter((f) => f.type.startsWith('image/'))
  if (files.value.length) showUpload.value = true
}

async function upload() {
  if (!files.value.length) return
  uploading.value = true
  uploadMsg.value = ''
  try {
    const res = await api.uploadPhotos(props.id, files.value, { phase: props.phase, start_at: takenAt.value, interval_min: 30 })
    uploadMsg.value = res.errors?.length ? res.errors.map((e) => `${e.file} — ${e.error}`).join('; ') : ''
    files.value = []
    if (fileInput.value) fileInput.value.value = ''
    if (!uploadMsg.value) showUpload.value = false
    await load()
    refreshNav()
  } catch (e) {
    uploadMsg.value = e.message
  } finally {
    uploading.value = false
  }
}

function chipState(photo, cls) {
  const c = photo.check.find((x) => x.classes.includes(cls))
  return c ? STATE[c.state] : ''
}

onMounted(() => {
  load()
  api.methodology().then((m) => { classes.value = (m.equipment || []).map((e) => e.cls) }).catch(() => {})
})
watch(() => [props.id, props.phase], () => { data.value = null; calc.value = null; takenAt.value = ''; load() })
</script>

<template>
  <main class="stage-page">
    <p v-if="error" class="error" role="alert">{{ error }} <button class="btn small" type="button" @click="load">Повторить</button></p>
    <p v-else-if="!data" role="status">Загружаем этап…</p>
    <template v-if="data && building">
      <nav class="workspace-breadcrumbs" aria-label="Путь к этапу">
        <RouterLink to="/">Стройки</RouterLink><span>/</span>
        <RouterLink :to="`/projects/${building.project.id}`">{{ building.project.name }}</RouterLink><span>/</span>
        <RouterLink :to="`/buildings/${id}/plan`">{{ building.name }} · план работ</RouterLink><span>/</span>
        <b>{{ data.name }}</b>
      </nav>

      <header class="sp-head">
        <h1>{{ data.name }}</h1>
        <p>
          <span v-if="stage.status_label" class="sp-status"><i :style="{ background: stage.color }"></i>{{ stage.status_label }}</span>
          план {{ stage.start ? `${formatDate(stage.start)} — ${formatDate(stage.end)}` : '—' }}
          <template v-if="stage.photo_start"> · факт с {{ formatDate(stage.photo_start) }}</template>
          · {{ countLabel(data.photos.length, 'снимок', 'снимка', 'снимков') }}
        </p>
      </header>

      <section class="sp-block">
        <h2>Техника</h2>
        <table v-if="rows.length" class="sp-table">
          <thead><tr><th></th><th>Нужно по нормам</th><th v-if="calc">Нужно по расчёту</th><th>По факту</th></tr></thead>
          <tbody>
            <tr v-for="r in rows" :key="r.label" :class="STATE[r.state]">
              <td>{{ r.label }}</td>
              <td>{{ r.norm }}</td>
              <td v-if="calc">{{ r.calc ?? '—' }}</td>
              <td class="sp-fact"><i></i>{{ r.seen }}</td>
            </tr>
          </tbody>
        </table>
        <p v-else class="sp-muted">Для этого этапа техника не нормируется: {{ data.hint || 'работы не видны камерами' }}.</p>
        <p v-if="crane" class="sp-muted">Кран: высота подъёма от {{ crane.hook_height_m }} м, вылет от {{ crane.reach_m }} м, грузоподъёмность на вылете от {{ String(crane.load_t).replace('.', ',') }} т.</p>
        <div class="sp-foot">
          <StageCalc v-if="data.calc" :kind="data.calc" @result="calc = $event" />
          <details class="sc">
            <summary>Нормативная база</summary>
            <ul class="sp-docs"><li v-for="d in data.docs" :key="d">{{ d }}</li></ul>
          </details>
        </div>
      </section>

      <section class="sp-block">
        <div class="sp-block-head">
          <h2>Снимки этапа</h2>
          <button class="btn small primary" type="button" @click="showUpload = !showUpload"><Icon name="upload" />Добавить</button>
        </div>
        <form v-if="showUpload" class="sp-upload" @submit.prevent="upload">
          <input ref="fileInput" type="file" multiple hidden accept="image/jpeg,image/png,image/webp,image/bmp" @change="pick" />
          <button type="button" class="sp-drop" :disabled="uploading" @click="fileInput.click()">
            <Icon name="upload" />{{ files.length ? `Выбрано: ${files.length}` : 'Выбрать снимки' }}
          </button>
          <input v-model="takenAt" type="datetime-local" aria-label="Время первого снимка" :disabled="uploading" />
          <button class="btn primary small" type="submit" :disabled="uploading || !files.length">{{ uploading ? 'Распознаём…' : 'Загрузить' }}</button>
          <span v-if="uploadMsg" class="sp-msg">{{ uploadMsg }}</span>
        </form>
        <p v-if="!data.photos.length" class="sp-muted">Снимков этого этапа пока нет.</p>
        <ul v-else class="sp-grid">
          <li v-for="p in data.photos" :key="p.id">
            <button type="button" class="sp-card" @click="openId = p.id">
              <span class="sp-img">
                <img :src="imageUrl(p.id, 480)" :alt="`Снимок от ${formatDateTime(p.taken_at)}`" loading="lazy" />
                <span class="sp-date">{{ formatDateTime(p.taken_at) }}</span>
              </span>
              <span class="sp-chips">
                <span v-for="e in p.equipment" :key="e.cls" class="sp-chip" :class="chipState(p, e.cls)">{{ e.label }} {{ e.count }}</span>
                <span v-if="!p.equipment.length" class="sp-muted">техника не найдена</span>
              </span>
              <span v-if="p.mismatch" class="sp-note">по технике похоже на «{{ p.auto_phase_name }}»</span>
            </button>
          </li>
        </ul>
      </section>

      <PhotoInspector v-if="openId" :photo-id="openId" :classes="classes" @close="openId = null" @changed="load" />
    </template>
  </main>
</template>

<style>
.stage-page { padding: 24px 40px 60px; max-width: 1200px; margin: auto; }
.sp-head { margin: 8px 0 28px; }
.sp-head h1 { font-family: var(--serif); font-size: 40px; font-weight: 600; line-height: 1.1; letter-spacing: -0.02em; margin: 0 0 8px; }
.sp-head p { margin: 0; color: var(--ink-3); font-size: 13px; display: flex; flex-wrap: wrap; align-items: center; gap: 4px; }
.sp-status { display: inline-flex; align-items: center; gap: 6px; color: var(--ink); font-weight: 600; margin-right: 6px; }
.sp-status i { width: 8px; height: 8px; border-radius: 50%; }
.sp-block { margin-bottom: 36px; }
.sp-block h2 { font-family: var(--serif); font-size: 22px; font-weight: 600; margin: 0 0 12px; }
.sp-block-head { display: flex; align-items: center; justify-content: space-between; margin-bottom: 12px; }
.sp-block-head h2 { margin: 0; }
.sp-table { width: 100%; max-width: 720px; border-collapse: collapse; font-size: 14px; }
.sp-table th { text-align: left; font-size: 11px; font-weight: 600; color: var(--ink-3); padding: 0 12px 8px 0; }
.sp-table td { padding: 9px 12px 9px 0; border-top: 1px solid var(--hair); }
.sp-table td:first-child { font-weight: 600; }
.sp-fact { font-weight: 700; white-space: nowrap; }
.sp-fact i { display: inline-block; width: 8px; height: 8px; border-radius: 50%; margin-right: 8px; background: var(--ink-3); }
.sp-table .ok .sp-fact i { background: var(--ok); }
.sp-table .warn .sp-fact i { background: #d97706; }
.sp-table .bad .sp-fact i { background: var(--critical); }
.sp-foot { display: flex; flex-wrap: wrap; gap: 8px 28px; margin-top: 14px; }
.sp-docs { margin: 8px 0 0; padding-left: 18px; display: grid; gap: 3px; color: var(--ink-2, var(--ink)); }
.sp-muted { color: var(--ink-3); font-size: 13px; margin: 8px 0 0; }
.sp-upload { display: flex; flex-wrap: wrap; align-items: center; gap: 10px; margin-bottom: 16px; }
.sp-upload input[type="datetime-local"] { font: inherit; font-size: 12px; padding: 6px 8px; }
.sp-drop { display: inline-flex; align-items: center; gap: 8px; padding: 8px 14px; border: 1.5px dashed var(--hair-2); border-radius: 10px; background: transparent; color: var(--accent); font: inherit; font-size: 13px; cursor: pointer; }
.sp-drop svg { width: 16px; height: 16px; }
.sp-msg { font-size: 12px; color: var(--critical); }
.sp-grid { list-style: none; margin: 0; padding: 0; display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 18px; }
.sp-card { display: grid; gap: 8px; width: 100%; padding: 0; border: 0; background: none; color: inherit; font: inherit; text-align: left; cursor: zoom-in; }
.sp-img { position: relative; display: block; aspect-ratio: 4 / 3; border-radius: 12px; overflow: hidden; background: var(--panel-2); }
.sp-img img { width: 100%; height: 100%; object-fit: cover; display: block; }
.sp-date { position: absolute; left: 8px; bottom: 8px; font-size: 11px; padding: 2px 8px; border-radius: 999px; background: rgba(10, 14, 16, 0.6); color: #fff; }
.sp-chips { display: flex; flex-wrap: wrap; gap: 5px; }
.sp-chip { font-size: 11px; font-weight: 600; padding: 2px 8px; border-radius: 999px; color: #fff; background: var(--ink-3); }
.sp-chip.ok { background: var(--ok); }
.sp-chip.warn { background: #d97706; }
.sp-chip.bad { background: var(--critical); }
.sp-note { font-size: 11px; color: #d97706; }
@media (max-width: 760px) { .stage-page { padding: 20px 16px 40px; } .sp-head h1 { font-size: 30px; } }
</style>
