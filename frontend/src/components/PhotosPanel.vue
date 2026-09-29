<script setup>
import { onMounted, ref } from 'vue'
import { api, imageUrl } from '../api.js'
import { countLabel, formatDateTime, toLocalInput } from '../lib/format.js'
import CameraPanel from './CameraPanel.vue'
import Icon from './Icon.vue'
import PhotoInspector from './PhotoInspector.vue'

// Экран «Фото и техника»: «Камеры» и «Снимки объекта» (архив загруженных и проанализированных кадров).
// Загрузка открывается одной кнопкой; этап у каждого снимка определяется по технике автоматически.
const props = defineProps({
  building: { type: Object, required: true },
  classes: { type: Array, default: () => [] },
  equipment: { type: Array, default: () => [] },
  phases: { type: Array, default: () => [] },
})
const emit = defineEmits(['changed'])

const PAGE = 48
const photos = ref([])
const total = ref(0)
const cameras = ref([])
const filters = ref({ camera_id: '', phase: '' })
const openId = ref(null)
const error = ref('')
const camerasPanel = ref(null)

const showUpload = ref(false)
const files = ref([])
const fileInput = ref(null)
const dragging = ref(false)
const up = ref({ camera_id: '', phase: '', start_at: toLocalInput(), interval: 30 })
const progress = ref(null)
const uploadErrors = ref([])

async function loadPhotos(append = false) {
  try {
    const res = await api.photos(props.building.id, { limit: PAGE, offset: append ? photos.value.length : 0, ...filters.value })
    photos.value = append ? [...photos.value, ...res.items] : res.items
    total.value = res.total
    error.value = ''
  } catch (e) {
    error.value = e.message
  }
}

async function loadCameras() {
  cameras.value = (await api.cameras(props.building.project_id).catch(() => []))
    .filter((c) => !c.building_id || c.building_id === props.building.id)
}

function openUpload() {
  showUpload.value = true
  // новая серия по умолчанию продолжает ленту: через 30 минут после последнего снимка
  if (photos.value.length) up.value.start_at = toLocalInput(new Date(new Date(photos.value[0].taken_at).getTime() + 30 * 60000))
  if (!up.value.camera_id && filters.value.camera_id) up.value.camera_id = filters.value.camera_id
}

function pick(event) {
  dragging.value = false
  const list = [...(event.target.files || event.dataTransfer?.files || [])]
  files.value = list.filter((f) => f.type.startsWith('image/'))
  if (files.value.length && !showUpload.value) openUpload()
}

async function upload() {
  if (!files.value.length) return
  uploadErrors.value = []
  const list = files.value
  const base = new Date(up.value.start_at)
  const step = list.length > 1 ? Number(up.value.interval) || 0 : 0
  progress.value = { done: 0, total: list.length }
  try {
    for (let i = 0; i < list.length; i += 3) {
      const res = await api.uploadPhotos(props.building.id, list.slice(i, i + 3), {
        camera_id: up.value.camera_id, phase: up.value.phase,
        start_at: toLocalInput(new Date(base.getTime() + i * step * 60000)), interval_min: step,
      })
      uploadErrors.value.push(...res.errors)
      progress.value = { done: Math.min(list.length, i + 3), total: list.length }
    }
    files.value = []
    if (fileInput.value) fileInput.value.value = ''
    if (!uploadErrors.value.length) showUpload.value = false
    await loadPhotos()
    camerasPanel.value?.load()
    emit('changed')
  } catch (e) {
    uploadErrors.value.push({ file: '', error: e.message })
  } finally {
    progress.value = null
  }
}

function selectCamera(id) {
  filters.value.camera_id = id
  loadPhotos()
}

function onChanged() {
  loadPhotos()
  loadCameras()
  camerasPanel.value?.load()
  emit('changed')
}

onMounted(() => {
  loadPhotos()
  loadCameras()
})
</script>

<template>
  <div class="monitor">
    <CameraPanel ref="camerasPanel" :project-id="building.project_id" :building-id="building.id" :buildings="[building]"
      :selected="filters.camera_id" @select="selectCamera" @changed="onChanged" />

    <section id="shots" class="panel shots" :class="{ over: dragging }" @dragover.prevent="dragging = true"
      @dragleave.self="dragging = false" @drop.prevent="pick">
      <div class="shots-head">
        <h2>Снимки объекта</h2>
        <span class="shots-count">{{ countLabel(total, 'снимок', 'снимка', 'снимков') }}</span>
        <select v-model="filters.camera_id" aria-label="Камера" @change="loadPhotos()">
          <option value="">Все камеры</option>
          <option v-for="c in cameras" :key="c.id" :value="String(c.id)">{{ c.name }}</option>
        </select>
        <select v-model="filters.phase" aria-label="Этап" @change="loadPhotos()">
          <option value="">Все этапы</option>
          <option v-for="p in phases" :key="p.id" :value="p.id">{{ p.name }}</option>
          <option value="none">Этап не определён</option>
        </select>
        <button class="btn primary small" type="button" @click="showUpload ? (showUpload = false) : openUpload()"><Icon name="upload" />Загрузить</button>
      </div>

      <form v-if="showUpload" class="shots-upload" @submit.prevent="upload">
        <input ref="fileInput" type="file" accept="image/jpeg,image/png,image/webp,image/bmp" multiple hidden @change="pick" />
        <button type="button" class="shots-drop" :disabled="!!progress" @click="fileInput.click()">
          <Icon :name="files.length ? 'image' : 'upload'" />{{ files.length ? countLabel(files.length, 'файл выбран', 'файла выбрано', 'файлов выбрано') : 'Выбрать или перетащить снимки' }}
        </button>
        <label>Камера<select v-model="up.camera_id"><option value="">Без камеры</option><option v-for="c in cameras" :key="c.id" :value="String(c.id)">{{ c.name }}</option></select></label>
        <label>Этап<select v-model="up.phase"><option value="">Определить по технике</option><option v-for="p in phases" :key="p.id" :value="p.id">{{ p.name }}</option></select></label>
        <label>Время съёмки<input v-model="up.start_at" type="datetime-local" required /></label>
        <label v-if="files.length > 1">Шаг, мин<input v-model.number="up.interval" type="number" min="0" max="1440" /></label>
        <button class="btn primary" type="submit" :disabled="!files.length || !!progress">{{ progress ? `Распознаём ${progress.done} из ${progress.total}` : 'Загрузить и распознать' }}</button>
        <ul v-if="uploadErrors.length" class="shots-errors"><li v-for="(e, i) in uploadErrors" :key="i">{{ e.file ? e.file + ': ' : '' }}{{ e.error }}</li></ul>
      </form>

      <p v-if="error" class="error" role="alert">{{ error }}</p>
      <div v-if="!photos.length" class="shots-empty">
        <Icon name="image" />
        <p>{{ filters.camera_id || filters.phase ? 'По выбранным условиям снимков нет.' : 'Снимков пока нет — загрузите кадры с камер площадки или перетащите их сюда.' }}</p>
      </div>
      <ul v-else class="shots-grid">
        <li v-for="p in photos" :key="p.id">
          <button type="button" class="shot-card" @click="openId = p.id">
            <span class="shot-img">
              <img :src="imageUrl(p.id, 480)" :alt="`Снимок от ${formatDateTime(p.taken_at)}`" loading="lazy" />
              <span class="shot-time">{{ formatDateTime(p.taken_at) }}</span>
            </span>
            <span class="shot-meta">
              <span v-if="p.phase" class="shot-stage" :class="{ manual: p.phase_source === 'manual' }">{{ p.phase_source === 'auto' && !p.phase_confirmed ? '≈ ' : '' }}{{ p.phase_name }}</span>
              <span class="shot-eq">{{ p.equipment.map((e) => e.count > 1 ? `${e.label} ×${e.count}` : e.label).join(', ') || 'техника не найдена' }}</span>
            </span>
          </button>
        </li>
      </ul>
      <div v-if="photos.length < total" class="shots-more"><button class="btn small" type="button" @click="loadPhotos(true)">Показать ещё</button></div>
    </section>

    <PhotoInspector v-if="openId" :photo-id="openId" :classes="classes" @close="openId = null" @changed="onChanged" />
  </div>
</template>

<style>
.monitor { display: grid; gap: 20px; }
.shots { padding: 20px 22px; }
.shots.over { outline: 2px dashed var(--accent); outline-offset: -6px; }
.shots-head { display: flex; flex-wrap: wrap; align-items: center; gap: 10px; margin-bottom: 14px; }
.shots-head h2 { font-size: 22px; margin: 0; }
.shots-count { color: var(--ink-3); font-size: 12px; margin-right: auto; }
.shots-head select { font: inherit; font-size: 12px; padding: 6px 8px; max-width: 220px; }
.shots-upload { display: flex; flex-wrap: wrap; align-items: flex-end; gap: 10px; padding: 14px; margin-bottom: 16px; border-radius: 12px; background: var(--panel-2); border: 1px solid var(--hair); }
.shots-upload label { display: grid; gap: 4px; font-size: 11px; color: var(--ink-3); }
.shots-upload select, .shots-upload input { font: inherit; font-size: 12px; padding: 6px 8px; }
.shots-upload input[type="number"] { width: 80px; }
.shots-drop { display: inline-flex; align-items: center; gap: 8px; padding: 9px 14px; border: 1.5px dashed var(--hair-2); border-radius: 10px; background: transparent; color: var(--accent); font: inherit; font-size: 13px; cursor: pointer; }
.shots-drop svg { width: 16px; height: 16px; }
.shots-errors { flex-basis: 100%; margin: 0; padding-left: 18px; color: var(--critical); font-size: 12px; }
.shots-empty { display: grid; justify-items: center; gap: 6px; padding: 36px 16px; color: var(--ink-3); text-align: center; font-size: 13px; }
.shots-empty svg { width: 28px; height: 28px; }
.shots-empty p { margin: 0; }
.shots-grid { list-style: none; margin: 0; padding: 0; display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 16px; }
.shot-card { display: grid; gap: 8px; width: 100%; padding: 0; border: 0; background: none; color: inherit; font: inherit; text-align: left; cursor: zoom-in; }
.shot-img { position: relative; display: block; aspect-ratio: 4 / 3; border-radius: 12px; overflow: hidden; background: var(--panel-2); }
.shot-img img { width: 100%; height: 100%; object-fit: cover; display: block; transition: transform 0.6s ease; }
.shot-card:hover img { transform: scale(1.04); }
.shot-time { position: absolute; left: 8px; bottom: 8px; font-size: 11px; padding: 2px 8px; border-radius: 999px; background: rgba(10, 14, 16, 0.6); color: #fff; }
.shot-meta { display: grid; gap: 4px; }
.shot-stage { justify-self: start; font-size: 11px; font-weight: 600; padding: 2px 8px; border-radius: 999px; color: var(--accent); background: color-mix(in srgb, var(--accent) 12%, transparent); }
.shot-stage.manual { color: var(--accent-ink); background: var(--accent); }
.shot-eq { font-size: 12px; color: var(--ink-3); }
.shots-more { display: flex; justify-content: center; margin-top: 18px; }
</style>
