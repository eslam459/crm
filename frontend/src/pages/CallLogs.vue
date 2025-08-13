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
      <Button
        size="sm"
        :variant="filters.tagResidential ? 'solid' : 'subtle'"
        @click="filters.tagResidential = !filters.tagResidential"
      >
        <template #prefix><FeatherIcon name="home" class="h-4" /></template>
        {{ __('Residential') }}
      </Button>
      <Button
        size="sm"
        :variant="filters.tagCommercial ? 'solid' : 'subtle'"
        @click="filters.tagCommercial = !filters.tagCommercial"
      >
        <template #prefix><FeatherIcon name="briefcase" class="h-4" /></template>
        {{ __('Commercial') }}
      </Button>

      <Button size="sm" variant="ghost" class="ml-auto" @click="clearAll">
        <template #prefix><FeatherIcon name="x-circle" class="h-4" /></template>
        {{ __('Clear all') }}
      </Button>
    </div>
  </div>

  <!-- Project cards -->
  <div class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-6 p-6">
    <Card
      v-for="project in filteredRows"
      :key="project.name"
      class="cursor-pointer hover:shadow-lg transition overflow-hidden"
      @click="goToUnits(project)"
    >
      <!-- Cover with small logo -->
      <div class="relative w-full h-40 bg-gray-100 dark:bg-gray-900">
        <img
          v-if="project.cover_image"
          :src="project.cover_image"
          alt=""
          class="w-full h-full object-cover"
          loading="lazy"
        />
        <div
          v-if="project.logo"
          class="absolute left-3 top-3 w-10 h-10 rounded bg-white/90 backdrop-blur p-1 shadow"
          @click.stop
        >
          <img :src="project.logo" alt="logo" class="w-full h-full object-contain" />
        </div>
      </div>

      <template #header>
        <div class="flex items-start justify-between gap-3">
          <div>
            <div class="font-semibold text-lg truncate">
              {{ project.project_name || project.name }}
            </div>
            <div class="text-xs text-gray-500 truncate">
              {{ project.city ? project.city + (project.district ? ' • ' + project.district : '') : (project.location || '-') }}
            </div>
            <div class="text-[11px] text-gray-500 mt-1" v-if="project.status">
              {{ __('Status') }}: {{ project.status }}
            </div>
          </div>
          <div
            class="text-xs px-2 py-0.5 rounded bg-gray-100 dark:bg-gray-800 shrink-0"
            title="Units in this project"
            @click.stop
          >
            {{ project.properties_count ?? 0 }} {{ __('Properties') }}
          </div>
        </div>
      </template>

      <template #content>
        <div class="space-y-1 text-sm">
          <div v-if="project.categories">
            <span class="font-medium">{{ __('Categories') }}:</span>
            <span>{{ project.categories.split('\n').filter(Boolean).join(', ') }}</span>
          </div>

          <div class="grid grid-cols-2 gap-2">
            <div><span class="font-medium">{{ __('Min Price') }}:</span> {{ fmt(project.min_price) }}</div>
            <div><span class="font-medium">{{ __('Max Price') }}:</span> {{ fmt(project.max_price) }}</div>
          </div>

          <div class="line-clamp-2">
            <span class="font-medium">{{ __('Description') }}:</span>
            <span>{{ project.description || '-' }}</span>
          </div>
        </div>
      </template>

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

    <!-- Skeletons -->
    <Card
      v-if="!rows.length"
      v-for="n in 3"
      :key="'ph-' + n"
      class="animate-pulse opacity-70 border-dashed border-2 border-gray-200"
      style="pointer-events:none;"
    >
      <template #header>
        <div class="font-bold text-lg bg-gray-100 h-6 w-1/2 rounded"></div>
        <div class="text-xs bg-gray-100 h-4 w-1/4 rounded mt-2"></div>
      </template>
      <template #content>
        <div class="mb-2"><span class="font-semibold bg-gray-100 h-4 w-1/3 rounded inline-block"></span></div>
        <div><span class="bg-gray-100 h-4 w-1/4 rounded inline-block"></span></div>
        <div><span class="bg-gray-100 h-4 w-1/4 rounded inline-block"></span></div>
      </template>
      <template #footer>
        <Button size="sm" disabled class="opacity-50">{{ __('Edit') }}</Button>
      </template>
    </Card>
  </div>

  <!-- Empty -->
  <div v-if="!rows.length" class="flex flex-col items-center gap-3 text-xl font-medium text-ink-gray-4 py-16">
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
const rowsBuf = ref([])
const rows = computed(() => rowsBuf.value)

// Filters
const filters = ref({
  q: '',
  location: '',
  status: '',
  tagResidential: false,
  tagCommercial: false,
})

const locationOptions = computed(() => {
  const uniq = [...new Set((rows.value || []).map(r => r.city || r.location).filter(Boolean))]
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
  const wantRes = filters.value.tagResidential
  const wantCom = filters.value.tagCommercial

  return rows.value.filter(p => {
    if (q) {
      const hay = [
        p.project_name, p.name, p.developer,
        p.city, p.district, p.location, p.description
      ].filter(Boolean).join(' ').toLowerCase()
      if (!hay.includes(q)) return false
    }
    const cityOrLoc = p.city || p.location || ''
    if (loc && cityOrLoc !== loc) return false
    if (stat && p.status !== stat) return false

    const bag = (p.categories || `${p.description || ''} ${p.area || ''}`).toLowerCase()
    if (wantRes && !bag.includes('residential')) return false
    if (wantCom && !bag.includes('commercial')) return false

    return true
  })
})

function fmt(v) {
  if (v === null || v === undefined || v === '') return '-'
  const n = Number(v)
  return Number.isFinite(n) ? n : String(v)
}

function clearAll() {
  filters.value = { q: '', location: '', status: '', tagResidential: false, tagCommercial: false }
}

onMounted(fetchProjects)

async function fetchProjects() {
  try {
    const res = await call('frappe.client.get_list', {
      doctype: 'Real Estate Project',
      fields: [
        'name','project_name','status','developer',
        'location','city','district','categories',
        'min_price','max_price','area','down_payment',
        'payment_plan','description','cover_image','logo',
        'properties_count'
      ],
      limit: 300,
      order_by: 'modified desc',
    })
    rowsBuf.value = Array.isArray(res) ? res : []
    await ensurePropertyCounts() // make sure counts are correct
  } catch (e) {
    console.error('Error fetching projects:', e)
    rowsBuf.value = []
  }
}

/**
 * Count units per project.
 * No `frappe.client.get_meta` (not in your stack) — assume link field is 'project'.
 * Try grouped count first; if it fails, fall back to per-project `get_count`.
 */
async function ensurePropertyCounts() {
  if (!rowsBuf.value.length) return
  const linkField = 'project'

  // fast grouped try
  try {
    const grouped = await call('frappe.client.get_list', {
      doctype: 'Project Unit',
      fields: [linkField, 'count(name) as cnt'],
      group_by: linkField,
      limit: 10000,
    })
    if (Array.isArray(grouped) && grouped.length) {
      const map = Object.create(null)
      grouped.forEach(r => { if (r[linkField]) map[r[linkField]] = Number(r.cnt) || 0 })
      rowsBuf.value = rowsBuf.value.map(p => ({ ...p, properties_count: map[p.name] ?? 0 }))
      return
    }
  } catch {
    // ignore, we’ll fall back
  }

  // fallback: per-project count
  await Promise.all(rowsBuf.value.map(async p => {
    try {
      const c = await call('frappe.client.get_count', {
        doctype: 'Project Unit',
        filters: { [linkField]: p.name },
      })
      p.properties_count = c || 0
    } catch {
      p.properties_count = p.properties_count || 0
    }
  }))
}

// open modal
function openModal(project = null) {
  modalProject.value = project ? { ...project } : null
  showProjectModal.value = true
}

// route to tabbed project view
function goToUnits(project) {
  const name = project?.name || project?.project_name
  if (!name) return
  router.push({ name: 'ProjectView', params: { project: name }, hash: '#units' })
}

// (kept for backwards compatibility if you referenced it elsewhere)
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
.line-clamp-2 {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
</style>
