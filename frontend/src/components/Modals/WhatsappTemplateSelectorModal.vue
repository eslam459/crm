<template>
  <div
    v-if="modelValue"
    class="fixed inset-0 z-[1000] flex items-center justify-center"
    @keydown.esc="close"
  >
    <!-- Backdrop -->
    <div class="absolute inset-0 bg-black/40" @click="close"></div>

    <!-- Panel -->
    <div class="relative z-10 w-[95vw] max-w-5xl max-h-[90vh] bg-white dark:bg-gray-900 rounded-2xl shadow-xl overflow-hidden">
      <!-- Header -->
      <div class="flex items-center justify-between px-5 py-4 border-b dark:border-gray-800">
        <h2 class="text-lg font-semibold">
          {{ isEdit ? __('Edit Project') : __('Create Project') }}
        </h2>
        <Button variant="subtle" @click="close">{{ __('Close') }}</Button>
      </div>

      <!-- Body -->
      <div class="p-5 overflow-y-auto" style="max-height: calc(90vh - 120px)">
        <div v-if="loading" class="text-sm text-ink-gray-5">{{ __('Loading…') }}</div>

        <div v-else class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4">
          <!-- Main fields -->
          <div>
            <label class="block text-sm mb-1">
              {{ __('Project Name') }} <span class="text-red-500">*</span>
            </label>
            <Input v-model="form.project_name" />
          </div>

          <div>
            <label class="block text-sm mb-1">{{ __('Status') }}</label>
            <Select :options="statusOptions" v-model="form.status" placeholder="Select" />
          </div>

          <div>
            <label class="block text-sm mb-1">{{ __('Developer') }}</label>
            <Input v-model="form.developer" />
          </div>

          <div>
            <label class="block text-sm mb-1">{{ __('City') }}</label>
            <Input v-model="form.city" />
          </div>

          <div>
            <label class="block text-sm mb-1">{{ __('District') }}</label>
            <Input v-model="form.district" />
          </div>

          <div>
            <label class="block text-sm mb-1">{{ __('Location') }}</label>
            <Input v-model="form.location" />
          </div>

          <div>
            <label class="block text-sm mb-1">{{ __('Min Price') }}</label>
            <Input v-model="form.min_price" type="number" />
          </div>

          <div>
            <label class="block text-sm mb-1">{{ __('Max Price') }}</label>
            <Input v-model="form.max_price" type="number" />
          </div>

          <div>
            <label class="block text-sm mb-1">{{ __('Area') }}</label>
            <Input v-model="form.area" />
          </div>

          <div class="md:col-span-2 xl:col-span-3">
            <label class="block text-sm mb-1">{{ __('Down Payment (%)') }}</label>
            <Input v-model="form.down_payment" type="number" />
          </div>

          <div class="md:col-span-2 xl:col-span-3">
            <label class="block text-sm mb-1">{{ __('Categories') }}</label>
            <Input v-model="form.categories" placeholder="Residential\nCommercial" />
            <div class="text-xs text-gray-500 mt-1">
              {{ __('Use line breaks between values (e.g., Residential↵Commercial)') }}
            </div>
          </div>

          <div class="md:col-span-2 xl:col-span-3">
            <label class="block text-sm mb-1">{{ __('Payment Plan') }}</label>
            <Textarea v-model="form.payment_plan" rows="3" />
          </div>

          <div class="md:col-span-2 xl:col-span-3">
            <label class="block text-sm mb-1">{{ __('Description') }}</label>
            <Textarea v-model="form.description" rows="4" />
          </div>

          <div class="md:col-span-2 xl:col-span-3">
            <label class="block text-sm mb-1">{{ __('Location on Google Maps (URL)') }}</label>
            <Input v-model="form.location_map_url" placeholder="https://maps.google.com/..." />
          </div>

          <!-- Uploads -->
          <div class="md:col-span-2 xl:col-span-3 grid grid-cols-1 md:grid-cols-3 gap-4">
            <!-- Cover image -->
            <div>
              <label class="block text-sm mb-1">{{ __('Cover Image') }}</label>
              <div class="flex gap-3 items-start">
                <div class="w-40 h-24 bg-gray-100 rounded overflow-hidden flex items-center justify-center">
                  <img
                    v-if="preview.cover || resolveImage(form.cover_image)"
                    :src="preview.cover || resolveImage(form.cover_image)"
                    alt=""
                    class="w-full h-full object-cover"
                  />
                  <span v-else class="text-xs text-gray-500">{{ __('No image') }}</span>
                </div>
                <div class="space-y-2">
                  <input type="file" accept="image/*" @change="onFilePicked($event, 'cover')" />
                  <Button v-if="preview.cover" size="sm" variant="ghost" @click="clearPicked('cover')">
                    {{ __('Clear') }}
                  </Button>
                </div>
              </div>
            </div>

            <!-- Logo -->
            <div>
              <label class="block text-sm mb-1">{{ __('Logo') }}</label>
              <div class="flex gap-3 items-start">
                <div class="w-20 h-20 bg-gray-100 rounded overflow-hidden flex items-center justify-center">
                  <img
                    v-if="preview.logo || resolveImage(form.logo)"
                    :src="preview.logo || resolveImage(form.logo)"
                    alt=""
                    class="w-full h-full object-contain"
                  />
                  <span v-else class="text-xs text-gray-500">{{ __('No image') }}</span>
                </div>
                <div class="space-y-2">
                  <input type="file" accept="image/*" @change="onFilePicked($event, 'logo')" />
                  <Button v-if="preview.logo" size="sm" variant="ghost" @click="clearPicked('logo')">
                    {{ __('Clear') }}
                  </Button>
                </div>
              </div>
            </div>

            <!-- Brochure -->
            <div>
              <label class="block text-sm mb-1">{{ __('Brochure (PDF)') }}</label>
              <div class="space-y-2">
                <div class="text-xs text-gray-600">
                  <span v-if="form.brochure">{{ form.brochure }}</span>
                  <span v-else>{{ __('No file') }}</span>
                </div>
                <input type="file" accept="application/pdf" @change="onFilePicked($event, 'brochure')" />
                <Button v-if="preview.brochureName" size="sm" variant="ghost" @click="clearPicked('brochure')">
                  {{ __('Clear') }}
                </Button>
              </div>
            </div>
          </div>
        </div>

        <p v-if="errorMsg" class="text-red-600 mt-3 text-sm">{{ errorMsg }}</p>
      </div>

      <!-- Footer -->
      <div class="flex items-center justify-between px-5 py-4 border-t dark:border-gray-800">
        <div>
          <Button
            v-if="isEdit"
            variant="danger"
            :loading="deleting"
            @click="confirmDelete"
          >
            {{ __('Delete') }}
          </Button>
        </div>
        <div class="flex items-center gap-3">
          <Button variant="subtle" @click="close">{{ __('Cancel') }}</Button>
          <Button :loading="saving" variant="solid" @click="save">
            {{ isEdit ? __('Save') : __('Create') }}
          </Button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { Button, Input, Textarea, Select, call } from 'frappe-ui'
import { ref, watch, computed } from 'vue'

const props = defineProps({
  modelValue: { type: Boolean, default: false },
  project: { type: Object, default: null },
})
const emit = defineEmits(['update:modelValue', 'saved'])

const modelValue = computed({
  get: () => props.modelValue,
  set: (v) => emit('update:modelValue', v),
})

const statusOptions = [
  { label: 'Planning', value: 'Planning' },
  { label: 'In Progress', value: 'In Progress' },
  { label: 'Completed', value: 'Completed' },
  { label: 'On Hold', value: 'On Hold' },
  { label: 'Cancelled', value: 'Cancelled' },
]

const blank = () => ({
  doctype: 'Real Estate Project',
  name: null,
  modified: null,
  project_name: '',
  status: 'Planning',
  developer: '',
  city: '',
  district: '',
  location: '',
  min_price: null,
  max_price: null,
  area: '',
  down_payment: null,
  categories: '',
  payment_plan: '',
  description: '',
  location_map_url: '',
  cover_image: '',
  logo: '',
  brochure: '',
})

const form = ref(blank())
const saving = ref(false)
const deleting = ref(false)
const loading = ref(false)
const errorMsg = ref('')

const isEdit = computed(() => !!form.value.name)

/** picked files (not yet uploaded) */
const picked = ref({
  cover: null,    // { name, base64 }
  logo: null,     // { name, base64 }
  brochure: null, // { name, base64 }
})
/** local previews (object URLs / existing URLs) */
const preview = ref({
  cover: '',
  logo: '',
  brochureName: '',
})

watch(
  () => props.project,
  async (p) => {
    errorMsg.value = ''
    clearPickedAll()
    if (p?.name) {
      loading.value = true
      try {
        const doc = await call('frappe.client.get', {
          doctype: 'Real Estate Project',
          name: p.name,
        })
        form.value = Object.assign(blank(), doc)
      } catch (e) {
        console.error(e)
        errorMsg.value = e?.message || __('Could not load project')
        form.value = Object.assign(blank(), p)
        form.value.name = p.name || null
      } finally {
        loading.value = false
      }
    } else {
      form.value = blank()
    }
  },
  { immediate: true }
)

function close() {
  errorMsg.value = ''
  modelValue.value = false
}

function toNumberOrNull(v) {
  return v === '' || v === null || v === undefined ? null : Number(v)
}
function normaliseNumbers() {
  form.value.min_price = toNumberOrNull(form.value.min_price)
  form.value.max_price = toNumberOrNull(form.value.max_price)
  form.value.down_payment = toNumberOrNull(form.value.down_payment)
}

function extractServerMessage(e) {
  if (e?._server_messages) {
    try {
      const arr = JSON.parse(e._server_messages)
      if (Array.isArray(arr) && arr.length) return arr[0]
    } catch {}
  }
  return e?.message || __('Something went wrong')
}

function resolveImage(u) {
  if (!u) return ''
  if (u.startsWith('http')) return u
  if (u.startsWith('/files/') || u.startsWith('/private/files/')) return u
  return `/files/${u}`
}

/* ---------- file pick / preview helpers ---------- */
function clearPicked(which) {
  picked.value[which] = null
  if (which === 'cover') preview.value.cover = ''
  if (which === 'logo') preview.value.logo = ''
  if (which === 'brochure') preview.value.brochureName = ''
}
function clearPickedAll() {
  picked.value = { cover: null, logo: null, brochure: null }
  preview.value = { cover: '', logo: '', brochureName: '' }
}

function onFilePicked(e, which) {
  const file = e.target.files && e.target.files[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = () => {
    const dataUrl = reader.result // data:<mime>;base64,XXXX
    const base64 = String(dataUrl).split(',')[1] || ''
    picked.value[which] = { name: file.name, base64 }
    if (which === 'cover') preview.value.cover = URL.createObjectURL(file)
    if (which === 'logo') preview.value.logo = URL.createObjectURL(file)
    if (which === 'brochure') preview.value.brochureName = file.name
  }
  reader.readAsDataURL(file)
}

/* ---------- upload utility ---------- */
async function attachAndSetField(docname, which, fieldname) {
  const f = picked.value[which]
  if (!f) return null
  try {
    const attached = await call('frappe.client.attach_file', {
      doctype: 'Real Estate Project',
      docname,
      filename: f.name,
      filedata: f.base64,
      decode_base64: 1,
      is_private: 0,
    })
    const file_url = attached?.file_url
    if (file_url) {
      await call('frappe.client.set_value', {
        doctype: 'Real Estate Project',
        name: docname,
        fieldname,
        value: file_url,
      })
      // reflect in local form & preview
      form.value[fieldname] = file_url
      if (which === 'cover') preview.value.cover = resolveImage(file_url)
      if (which === 'logo') preview.value.logo = resolveImage(file_url)
      return file_url
    }
  } catch (e) {
    console.error(`attach ${which} failed`, e)
    throw e
  }
  return null
}

/* ---------- save flow ---------- */
async function save() {
  errorMsg.value = ''
  if (!form.value.project_name?.trim()) {
    errorMsg.value = __('Project Name is required')
    return
  }

  normaliseNumbers()
  saving.value = true
  try {
    let saved
    if (isEdit.value) {
      // Step 1: save doc fields
      saved = await call('frappe.client.save', { doc: form.value })
      // Step 2: handle uploads (if any)
      if (picked.value.cover) await attachAndSetField(saved.name, 'cover', 'cover_image')
      if (picked.value.logo) await attachAndSetField(saved.name, 'logo', 'logo')
      if (picked.value.brochure) await attachAndSetField(saved.name, 'brochure', 'brochure')
      // Step 3: refetch fresh doc (modified, fields)
      saved = await call('frappe.client.get', { doctype: 'Real Estate Project', name: saved.name })
    } else {
      // Step 1: insert
      const toInsert = { ...form.value, name: undefined, modified: undefined }
      saved = await call('frappe.client.insert', { doc: toInsert })
      // Step 2: uploads need a real docname now
      if (picked.value.cover) await attachAndSetField(saved.name, 'cover', 'cover_image')
      if (picked.value.logo) await attachAndSetField(saved.name, 'logo', 'logo')
      if (picked.value.brochure) await attachAndSetField(saved.name, 'brochure', 'brochure')
      // Step 3: refetch
      saved = await call('frappe.client.get', { doctype: 'Real Estate Project', name: saved.name })
    }

    // keep modal state fresh and emit to parent
    form.value = Object.assign({}, form.value, saved)
    emit('saved', saved)
    close()
  } catch (e) {
    console.error(e)
    errorMsg.value = extractServerMessage(e)
  } finally {
    saving.value = false
  }
}

async function confirmDelete() {
  if (!isEdit.value) return
  if (!window.confirm(__('Delete this project? This cannot be undone.'))) return

  deleting.value = true
  errorMsg.value = ''
  try {
    await call('frappe.client.delete', {
      doctype: 'Real Estate Project',
      name: form.value.name,
    })
    emit('saved', { deleted: true, name: form.value.name })
    close()
  } catch (e) {
    console.error(e)
    errorMsg.value = extractServerMessage(e)
  } finally {
    deleting.value = false
  }
}
</script>
