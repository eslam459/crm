<!-- frontend/src/pages/Inventory.vue -->
<template>
  <LayoutHeader>
    <template #left-header>
      <ViewBreadcrumbs :items="[{ label: __('Inventory'), route: { name: 'Inventory' } }]" />
    </template>

    <template #right-header>
      <Button variant="solid" :label="__('Create Project')" @click="openModal()">
        <template #prefix><FeatherIcon name="plus" class="h-4" /></template>
      </Button>
    </template>
  </LayoutHeader>

  <!-- Filter bar -->
  <div class="p-4 grid grid-cols-1 lg:grid-cols-6 gap-3 border-b">
    <div class="lg:col-span-3">
      <FormControl
        type="text"
        :label="__('Search')"
        :placeholder="__('Search by project, developer or location')"
        v-model="filters.q"
        :debounce="250"
      >
        <template #prefix>
          <FeatherIcon name="search" class="h-4 text-gray-500" />
        </template>
      </FormControl>
    </div>

    <FormControl
      class="lg:col-span-1"
      type="select"
      :label="__('Location')"
      v-model="filters.location"
      :options="locationOptions"
      clearable
      :placeholder="__('All')"
    />

    <FormControl
      class="lg:col-span-1"
      type="select"
      :label="__('Status')"
      v-model="filters.status"
      :options="statusOptions"
      clearable
      :placeholder="__('All')"
    />

    <div class="flex items-end gap-2">
      <Button size="sm" variant="ghost" class="ml-auto" @click="clearAll">
        <template #prefix><FeatherIcon name="x-circle" class="h-4" /></template>
        {{ __('Clear all') }}
      </Button>
    </div>
  </div>

  <!-- Cards grid (only when there is data after filtering) -->
  <div v-if="filteredRows.length" class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-6 p-6">
    <Card
      v-for="project in filteredRows"
      :key="project.name"
      class="cursor-pointer hover:shadow-lg transition overflow-hidden rounded-xl"
      @click="goToUnits(project)"
    >
      <!-- Header: cover image -->
      <template #header>
        <div class="relative w-full h-40 rounded-xl overflow-hidden bg-gray-100 dark:bg-gray-900">
          <img
            v-if="project.cover_image"
            :src="imgSrc(project.cover_image)"
            alt="cover"
            class="w-full h-full object-cover"
            loading="lazy"
            @error="onImgError"
          />
        </div>
      </template>

      <!-- Content -->
      <template #content>
        <div class="pt-3">
          <div class="font-semibold text-lg truncate mb-1">
            {{ project.project_name || project.name }}
          </div>

          <div class="space-y-2 text-sm text-gray-700 dark:text-gray-300">
            <div class="flex items-center gap-2 truncate">
              <FeatherIcon name="map-pin" class="h-4 shrink-0" />
              <span class="truncate">{{ project.location || '-' }}</span>
            </div>

            <div class="flex items-center gap-2 truncate">
              <FeatherIcon name="briefcase" class="h-4 shrink-0" />
              <span class="truncate">{{ project.developer || '-' }}</span>
            </div>

            <div class="flex items-center gap-2">
              <FeatherIcon name="layers" class="h-4 shrink-0" />
              <span>{{ project.number_of_unites || 0 }} {{ __('Properties') }}</span>
            </div>

            <div class="flex items-center gap-2 truncate">
              <FeatherIcon name="tag" class="h-4 shrink-0" />
              <span class="truncate">{{ categoryLabel(project.categories) }}</span>
            </div>
          </div>
        </div>
      </template>

      <!-- Footer -->
      <template #footer>
        <div class="flex items-center gap-2">
          <Button size="sm" @click.stop="openModal(project)">{{ __('Edit') }}</Button>
          <Button
            size="sm"
            variant="subtle"
            class="text-red-600"
            @click.stop="deleteProject(project)"
          >
            <template #prefix><FeatherIcon name="trash-2" class="h-4" /></template>
            {{ __('Delete') }}
          </Button>
        </div>
      </template>
    </Card>
  </div>

  <!-- Skeletons: show ONLY while loading -->
  <div v-else-if="loadingRows" class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-6 p-6">
    <Card v-for="n in 3" :key="'ph-' + n" class="animate-pulse opacity-70 border-dashed border-2 border-gray-200 rounded-xl" style="pointer-events:none;">
      <template #header>
        <div class="w-full h-40 bg-gray-100 rounded-xl"></div>
      </template>
      <template #content>
        <div class="font-bold text-lg bg-gray-100 h-6 w-1/2 rounded mb-2"></div>
        <div class="bg-gray-100 h-4 w-3/4 rounded mb-1"></div>
        <div class="bg-gray-100 h-4 w-2/3 rounded mb-1"></div>
        <div class="bg-gray-100 h-4 w-1/2 rounded"></div>
      </template>
      <template #footer>
        <Button size="sm" disabled class="opacity-50">{{ __('Edit') }}</Button>
      </template>
    </Card>
  </div>

  <!-- Empty state: appears only when NOT loading and no projects -->
  <div v-else class="flex flex-col items-center gap-3 text-xl font-medium text-ink-gray-4 py-16">
    <span>{{ __('No {0} Found', [__('Projects')]) }}</span>
    <Button :label="__('Create Project')" @click="openModal()">
      <template #prefix><FeatherIcon name="plus" class="h-4" /></template>
    </Button>
  </div>

  <!-- Modal -->
  <ProjectModal
    v-if="showProjectModal"
    v-model="showProjectModal"
    :project="modalProject"
    @saved="fetchProjects"
  />
</template>

<script setup>
import LayoutHeader from '@/components/LayoutHeader.vue'
import ViewBreadcrumbs from '@/components/ViewBreadcrumbs.vue'
import Card from '@/components/Card.vue'
import ProjectModal from '@/components/Modals/ProjectModal.vue'
import { Button, FormControl, FeatherIcon, call } from 'frappe-ui'
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

// UI state
const showProjectModal = ref(false)
const modalProject = ref(null)
const loadingRows = ref(true)          // <<<<<< NEW
const rowsBuf = ref([])
const rows = computed(() => rowsBuf.value)

// Filters
const filters = ref({ q: '', location: '', status: '' })

const locationOptions = computed(() => {
  const uniq = [...new Set((rows.value || []).map(r => r.location).filter(Boolean))]
  return [{ label: __('All'), value: '' }, ...uniq.map(v => ({ label: v, value: v }))]
})

const statusOptions = computed(() => {
  const uniq = [...new Set((rows.value || []).map(r => r.status).filter(Boolean))]
  return [{ label: __('All'), value: '' }, ...uniq.map(v => ({ label: v, value: v }))]
})

const filteredRows = computed(() => {
  const q = (filters.value.q || '').toLowerCase().trim()
  const loc = filters.value.location || ''
  const stat = filters.value.status || ''

  return rows.value.filter(p => {
    if (q) {
      const hay = [p.project_name, p.name, p.developer, p.location]
        .filter(Boolean).join(' ').toLowerCase()
      if (!hay.includes(q)) return false
    }
    if (loc && (p.location || '') !== loc) return false
    if (stat && p.status !== stat) return false
    return true
  })
})

function clearAll() {
  filters.value = { q: '', location: '', status: '' }
}

onMounted(fetchProjects)

async function fetchProjects() {
  loadingRows.value = true
  try {
    const res = await call('frappe.client.get_list', {
      doctype: 'Real Estate Project',
      fields: [
        'name',
        'project_name',
        'location',
        'developer',
        'cover_image',
        'number_of_unites',
        'categories',
        'status'
      ],
      limit: 300,
      order_by: 'modified desc',
    })
    rowsBuf.value = Array.isArray(res) ? res : []
  } catch (e) {
    console.error('Error fetching projects:', e)
    rowsBuf.value = []
  } finally {
    loadingRows.value = false   // <<<<<< stop skeletons
  }
}

/* images */
function imgSrc(val) {
  if (!val) return ''
  const p = String(val).trim()
  if (/^https?:\/\//i.test(p)) return p
  if (p.startsWith('/private/files/')) {
    const enc = encodeURIComponent(p)
    return `/api/method/frappe.utils.file_manager.download_file?file_url=${enc}`
  }
  return p.startsWith('/') ? p : '/' + p
}
function onImgError(e) { e.target.style.display = 'none' }

/* categories */
function categoryLabel(raw) {
  const s = (raw || '').toLowerCase()
  const com = s.includes('commercial')
  const adm = s.includes('administrative')
  if (com && adm) return 'Commercial & Administrative'
  if (com) return 'Commercial'
  if (adm) return 'Administrative'
  return raw || '-'
}

/* actions */
function openModal(project = null) {
  modalProject.value = project ? { ...project } : null
  showProjectModal.value = true
}
function goToUnits(project) {
  const name = project?.name || project?.project_name
  if (!name) return
  router.push({ name: 'ProjectView', params: { project: name }, hash: '#units' })
}
const goToProject = goToUnits

async function deleteProject(project) {
  if (!project?.name) return
  if (!confirm(__('Delete project “{0}”? This cannot be undone.', [project.project_name || project.name]))) return
  try {
    await call('frappe.client.delete', {
      doctype: 'Real Estate Project',
      name: project.name,
    })
    rowsBuf.value = rowsBuf.value.filter(p => p.name !== project.name)
  } catch (e) {
    console.error(e)
    alert(e?.message || __('Could not delete project'))
  }
}
</script>

<style scoped>
:deep(.card) { border-radius: 0.75rem; }
</style>
