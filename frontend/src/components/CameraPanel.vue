<script setup>
import { computed, onMounted, ref } from 'vue'
import { api, imageUrl } from '../api.js'
import { formatDateTime } from '../lib/format.js'
import Icon from './Icon.vue'

// «Камеры»: плитки с последним кадром каждой камеры. Клик по плитке — показать снимки этой камеры.
// Камера — точка съёмки (название, зона, объект); кадры привязываются к ней при загрузке, и по соседним
// кадрам одной камеры определяется, работает техника или стоит.
const props = defineProps({
  projectId: { type: Number, required: true },
  buildings: { type: Array, default: () => [] },
  buildingId: { type: Number, default: null },
  selected: { type: [Number, String], default: '' },
})
const emit = defineEmits(['changed', 'select'])

const cameras = ref([])
const error = ref('')
const busyId = ref(null)
const showForm = ref(false)
const editId = ref(null)
const edit = ref({ name: '', zone: '' })
const form = ref({ name: '', zone: '', building_id: props.buildingId || null })

const visible = computed(() => props.buildingId
  ? cameras.value.filter((c) => !c.building_id || c.building_id === props.buildingId)
  : cameras.value)
const buildingName = (id) => props.buildings.find((b) => b.id === id)?.name || 'все объекты'

async function load() {
  try {
    cameras.value = await api.cameras(props.projectId)
    error.value = ''
  } catch (e) {
    error.value = e.message
  }
}

async function act(camera, fn) {
  busyId.value = camera.id
  error.value = ''
  try {
    await fn()
    await load()
    emit('changed')
  } catch (e) {
    error.value = `${camera.name}: ${e.message}`
  } finally {
    busyId.value = null
  }
}

async function create() {
  if (!form.value.name.trim()) { error.value = 'Укажите название камеры.'; return }
  try {
    await api.createCamera(props.projectId, {
      name: form.value.name.trim(), zone: form.value.zone.trim(), building_id: form.value.building_id || null,
    })
    form.value = { name: '', zone: '', building_id: props.buildingId || null }
    showForm.value = false
    await load()
    emit('changed')
  } catch (e) {
    error.value = e.message
  }
}

function startEdit(camera) {
  editId.value = camera.id
  edit.value = { name: camera.name, zone: camera.zone || '' }
}

function saveEdit(camera) {
  if (!edit.value.name.trim()) { error.value = 'Название камеры не может быть пустым.'; return }
  act(camera, async () => {
    await api.updateCamera(camera.id, { name: edit.value.name.trim(), zone: edit.value.zone.trim() })
    editId.value = null
  })
}

function remove(camera) {
  if (window.confirm(`Удалить камеру «${camera.name}»? Снимки останутся, но без привязки к камере.`)) {
    act(camera, () => api.deleteCamera(camera.id))
  }
}

function pick(camera) {
  emit('select', String(props.selected) === String(camera.id) ? '' : String(camera.id))
}

onMounted(load)
defineExpose({ load })
</script>

<template>
  <section class="panel cams-panel">
    <div class="cams-head">
      <h2>Камеры</h2>
      <span class="cams-sub">{{ visible.length ? 'последний кадр каждой камеры · нажмите, чтобы показать её снимки' : '' }}</span>
      <button class="btn small" type="button" @click="showForm = !showForm"><Icon name="plus" />Камера</button>
    </div>

    <form v-if="showForm" class="cams-form" @submit.prevent="create">
      <input v-model="form.name" required placeholder="Название, например «Мачта, север»" aria-label="Название камеры" />
      <input v-model="form.zone" placeholder="Зона: котлован, въезд…" aria-label="Зона площадки" />
      <select v-if="!buildingId" v-model="form.building_id" aria-label="Объект">
        <option :value="null">Все объекты</option>
        <option v-for="b in buildings" :key="b.id" :value="b.id">{{ b.name }}</option>
      </select>
      <button class="btn primary small" type="submit">Добавить</button>
      <button class="btn ghost small" type="button" @click="showForm = false">Отмена</button>
    </form>

    <p v-if="error" class="error" role="alert">{{ error }}</p>
    <p v-if="!visible.length && !showForm" class="cams-empty">Камер нет. Добавьте камеру и выбирайте её при загрузке снимков.</p>

    <ul v-else class="cams-grid">
      <li v-for="c in visible" :key="c.id" class="cam-tile" :class="{ on: String(selected) === String(c.id) }">
        <button type="button" class="cam-view" :aria-pressed="String(selected) === String(c.id)" @click="pick(c)">
          <img v-if="c.last_photo_id" :src="imageUrl(c.last_photo_id, 480)" :alt="`Последний кадр камеры ${c.name}`" loading="lazy" />
          <span v-else class="cam-nosignal"><Icon name="camera" />кадров пока нет</span>
          <span v-if="c.last_photo_at" class="cam-time">{{ formatDateTime(c.last_photo_at) }}</span>
        </button>
        <div v-if="editId === c.id" class="cam-edit">
          <input v-model="edit.name" aria-label="Название камеры" />
          <input v-model="edit.zone" aria-label="Зона площадки" placeholder="Зона" />
          <div><button class="btn small primary" type="button" :disabled="busyId === c.id" @click="saveEdit(c)">Сохранить</button>
            <button class="btn small ghost" type="button" @click="editId = null">Отмена</button></div>
        </div>
        <div v-else class="cam-info">
          <span><b>{{ c.name }}</b><small>{{ c.zone || 'зона не указана' }}<template v-if="!buildingId"> · {{ buildingName(c.building_id) }}</template> · {{ c.photos_count }} сним.</small></span>
          <span class="cam-actions">
            <button type="button" title="Изменить" aria-label="Изменить камеру" :disabled="busyId === c.id" @click="startEdit(c)"><Icon name="edit" /></button>
            <button type="button" title="Удалить" aria-label="Удалить камеру" :disabled="busyId === c.id" @click="remove(c)"><Icon name="close" /></button>
          </span>
        </div>
      </li>
    </ul>
  </section>
</template>

<style>
.cams-panel { padding: 20px 22px; }
.cams-head { display: flex; align-items: baseline; gap: 12px; margin-bottom: 14px; }
.cams-head h2 { font-size: 22px; margin: 0; }
.cams-sub { color: var(--ink-3); font-size: 12px; margin-right: auto; }
.cams-form { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 14px; }
.cams-form input, .cams-form select { font: inherit; font-size: 13px; padding: 7px 10px; min-width: 180px; }
.cams-empty { color: var(--ink-3); font-size: 13px; margin: 0; }
.cams-grid { list-style: none; margin: 0; padding: 0; display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 14px; }
.cam-tile { display: grid; gap: 8px; }
.cam-view { position: relative; aspect-ratio: 16 / 10; padding: 0; border: 2px solid transparent; border-radius: 12px; overflow: hidden; background: var(--panel-2); cursor: pointer; }
.cam-tile.on .cam-view { border-color: var(--accent); }
.cam-view img { width: 100%; height: 100%; object-fit: cover; display: block; }
.cam-nosignal { position: absolute; inset: 0; display: grid; place-content: center; justify-items: center; gap: 6px; color: var(--ink-3); font-size: 12px; }
.cam-nosignal svg { width: 22px; height: 22px; }
.cam-time { position: absolute; left: 8px; bottom: 8px; font-size: 11px; padding: 2px 8px; border-radius: 999px; background: rgba(10, 14, 16, 0.6); color: #fff; }
.cam-info { display: flex; justify-content: space-between; gap: 8px; align-items: flex-start; }
.cam-info b { display: block; font-size: 13px; font-weight: 600; }
.cam-info small { color: var(--ink-3); font-size: 11px; }
.cam-actions { display: flex; gap: 2px; }
.cam-actions button { background: none; border: 0; padding: 4px; color: var(--ink-3); cursor: pointer; border-radius: 6px; }
.cam-actions button:hover { color: var(--ink); background: var(--panel-2); }
.cam-actions svg { width: 14px; height: 14px; }
.cam-edit { display: grid; gap: 6px; }
.cam-edit input { font: inherit; font-size: 12px; padding: 6px 8px; }
</style>
