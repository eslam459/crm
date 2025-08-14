<!-- frontend/src/pages/Inventory.vue -->
<template>
  <LayoutHeader>
    <!-- LEFT HEADER -->
    <template #left-header>
      <div class="text-base sm:text-lg font-semibold flex items-center gap-2">
        <RouterLink :to="{ name: 'Inventory' }" class="hover:underline">
          {{ __('Inventory') }}
        </RouterLink>
      </div>
    </template>

    <!-- RIGHT HEADER -->
    <template #right-header>
      <div class="flex items-center gap-2">
        <Button variant="subtle" @click="toggleFilterPanel">
          <template #prefix><FeatherIcon name="filter" class="h-4" /></template>
          {{ __('Filter') }}
        </Button>

        <!-- Contextual action per tab -->
        <Button
          v-if="activeTab==='projects'"
          variant="solid"
          :label="__('Create Project')"
          @click="openProjectModal()"
        >
          <template #prefix><FeatherIcon name="plus" class="h-4" /></template>
        </Button>

        <Button
          v-else
          variant="solid"
          :label="__('Add Unit')"
          @click="openUnitModal()"
        >
          <template #prefix><FeatherIcon name="plus" class="h-4" /></template>
        </Button>
      </div>
    </template>
  </LayoutHeader>

  <!-- Tabs -->
  <div class="px-4 pt-4">
    <div class="flex items-center gap-2 border-b">
      <button
        class="px-3 py-2 -mb-px"
        :class="activeTab === 'projects'
          ? 'border-b-2 border-gray-900 dark:border-white font-medium'
          : 'text-gray-500'"
        @click="activeTab = 'projects'"
      >
        {{ __('Projects') }}
        <span class="ml-1 text-xs opacity-60">({{ rows.length }})</span>
      </button>
      <button
        class="px-3 py-2 -mb-px"
        :class="activeTab === 'units'
          ? 'border-b-2 border-gray-900 dark:border-white font-medium'
          : 'text-gray-500'"
        @click="activeTab = 'units'"
      >
        {{ __('Units') }}
        <span class="ml-1 text-xs opacity-60">({{ units.length }})</span>
      </button>
    </div>
  </div>

  <!-- Filter bar (applies to Projects tab) -->
  <div v-if="activeTab==='projects'" class="p-4 grid grid-cols-1 lg:grid-cols-6 gap-3 border-b">
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

  <!-- Active filter chips (Projects tab) -->
  <div v-if="activeTab==='projects' && advFilters.length" class="px-6 pt-3 flex flex-wrap gap-2">
    <span
      v-for="(f, i) in advFilters"
      :key="'chip-'+i"
      class="inline-flex items-center gap-2 text-xs px-2 py-1 rounded-full bg-gray-100 dark:bg-gray-800"
    >
      <FeatherIcon name="filter" class="h-3" />
      <span class="font-medium">{{ fieldLabel(f.field) }}</span>
      <span class="opacity-70">{{ opLabel(f.op) }}</span>
      <span v-if="f.op!=='is_set' && f.op!=='is_not_set'">
        {{ f.value }}<span v-if="f.op==='between'"> → {{ f.value2 }}</span>
      </span>
      <button class="ml-1 opacity-70 hover:opacity-100" @click="removeFilter(i)">
        <FeatherIcon name="x" class="h-3" />
      </button>
    </span>
    <Button size="sm" variant="ghost" @click="clearAdvFilters">
      <template #prefix><FeatherIcon name="trash-2" class="h-4" /></template>
      {{ __('Clear Filters') }}
    </Button>
  </div>

  <!-- PROJECTS: Cards grid -->
  <div v-if="activeTab==='projects' && filteredRows.length" class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-6 p-6">
    <Card
      v-for="project in filteredRows"
      :key="project.name"
      class="cursor-pointer hover:shadow-lg transition overflow-hidden rounded-xl"
      @click="goToProject(project)"
    >
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
              <span>{{ availableCount(project) }} {{ __('Available Units') }}</span>
            </div>

            <div class="flex items-center gap-2">
              <FeatherIcon name="grid" class="h-4 shrink-0" />
              <span>{{ totalCount(project) }} {{ __('Total Units') }}</span>
            </div>

            <div class="flex items-center gap-2 truncate">
              <FeatherIcon name="tag" class="h-4 shrink-0" />
              <span class="truncate">{{ categoryLabel(project.categories) }}</span>
            </div>
          </div>
        </div>
      </template>

      <template #footer>
        <div class="flex items-center gap-2">
          <Button size="sm" @click.stop="openProjectModal(project)">{{ __('Edit') }}</Button>
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

  <!-- PROJECTS: Skeletons -->
  <div v-else-if="activeTab==='projects' && loadingRows" class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-6 p-6">
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

  <!-- PROJECTS: Empty -->
  <div v-else-if="activeTab==='projects'" class="flex flex-col items-center gap-3 text-xl font-medium text-ink-gray-4 py-16">
    <span>{{ __('No {0} Found', [__('Projects')]) }}</span>
    <Button :label="__('Create Project')" @click="openProjectModal()">
      <template #prefix><FeatherIcon name="plus" class="h-4" /></template>
    </Button>
  </div>

  <!-- UNITS TAB (from Unit doctype) -->
  <div v-if="activeTab==='units'" class="p-4">
    <div class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-6">
      <div v-for="u in units" :key="u.name" class="block">
        <Card class="hover:shadow transition cursor-pointer" @click="goToUnit(u)">
          <template #header>
            <div class="relative w-full h-40 rounded-xl overflow-hidden bg-gray-100 dark:bg-gray-900">
              <img
                v-if="u.cover_image"
                :src="imgSrc(u.cover_image)"
                alt="unit cover"
                class="w-full h-full object-cover"
                loading="lazy"
                @error="onImgError"
              />
            </div>
          </template>
          <template #content>
            <div class="pt-3">
              <div class="font-semibold text-lg truncate">
                {{ u.unit_name || u.name }}
              </div>
              <div class="text-xs opacity-70 mb-2">
                {{ u.type || '-' }}
                <span v-if="u.availability" class="mx-1">•</span>
                <span v-if="u.availability">{{ u.availability }}</span>
              </div>
              <div class="text-sm space-y-1">
                <div v-if="u.project">
                  <span class="font-medium">{{ __('Project') }}:</span>
                  <span> {{ u.project }}</span>
                </div>
                <div>
                  <span class="font-medium">{{ __('Area (sqm)') }}:</span>
                  {{ u.area_sqm ?? '—' }}
                </div>
                <div>
                  <span class="font-medium">{{ __('Price') }}:</span>
                  {{ fmt(u.price) }}
                </div>
                <div class="line-clamp-2">
                  <span class="font-medium">{{ __('Description') }}:</span>
                  {{ u.description || '—' }}
                </div>
              </div>
            </div>
          </template>
          <template #footer>
            <div class="flex gap-2">
              <Button size="sm" @click.stop="openUnitModal(u)">{{ __('Edit') }}</Button>
              <Button size="sm" variant="subtle" class="text-red-600" @click.stop="deleteUnit(u)">
                <template #prefix><FeatherIcon name="trash-2" class="h-4" /></template>
                {{ __('Delete') }}
              </Button>
            </div>
          </template>
        </Card>
      </div>
    </div>

    <div v-if="!loadingUnits && !units.length" class="text-center text-gray-500 py-12">
      {{ __('No units found.') }}
    </div>
  </div>

  <!-- Filter Panel (Projects tab only) -->
  <div v-if="filterPanelOpen" class="fixed inset-0 z-[1000]" @click.self="filterPanelOpen=false">
    <div class="absolute right-4 top=[72px] w=[min(640px,95vw)] bg-white dark:bg-gray-900 rounded-xl shadow-2xl border dark:border-gray-800 p-4">
      <div class="flex items-center justify-between mb-3">
        <div class="text-sm font-semibold">{{ __('Add Filter') }}</div>
        <Button size="sm" variant="ghost" @click="filterPanelOpen=false">
          <FeatherIcon name="x" class="h-4" />
        </Button>
      </div>

      <div class="grid grid-cols-1 md:grid-cols-12 gap-2">
        <div class="md:col-span-4">
          <FormControl
            type="select"
            :label="__('Field')"
            v-model="newFilter.field"
            :options="filterFieldOptions"
            searchable
            clearable
            :placeholder="__('Choose field')"
          />
        </div>

        <div class="md:col-span-4">
          <FormControl
            type="select"
            :label="__('Operator')"
            v-model="newFilter.op"
            :options="operatorOptionsFor(newFilter.field)"
            :placeholder="__('Choose operator')"
            :disabled="!newFilter.field"
          />
        </div>

        <div class="md:col-span-4" v-if="newFilter.op && showValueInput(newFilter.op)">
          <FormControl
            :type="inputTypeFor(newFilter.field, newFilter.op)"
            :label="__('Value')"
            v-model="newFilter.value"
            :placeholder="__('Enter value')"
          />
        </div>

        <div class="md:col-span-4" v-if="newFilter.op==='between'">
          <FormControl
            :type="inputTypeFor(newFilter.field, newFilter.op)"
            :label="__('and')"
            v-model="newFilter.value2"
            :placeholder="__('Second value')"
          />
        </div>

        <div class="md:col-span-12 flex items-center gap-2 mt-1">
          <Button size="sm" :disabled="!canAddFilter" @click="addFilter">
            <template #prefix><FeatherIcon name="plus-circle" class="h-4" /></template>
            {{ __('Add') }}
          </Button>
          <Button size="sm" variant="subtle" @click="clearAdvFilters">
            <template #prefix><FeatherIcon name="trash-2" class="h-4" /></template>
            {{ __('Clear All') }}
          </Button>
          <div class="ml-auto">
            <Button size="sm" variant="subtle" @click="filterPanelOpen=false">{{ __('Close') }}</Button>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Modals -->
  <ProjectModal
    v-if="showProjectModal"
    v-model="showProjectModal"
    :project="modalProject"
    @saved="fetchProjects"
  />
  <UnitModal
    v-if="showUnitModal"
    v-model="showUnitModal"
    :unit="editingUnit"
    @saved="onUnitSaved"
  />
</template>

<script setup>
import LayoutHeader from '@/components/LayoutHeader.vue'
import Card from '@/components/Card.vue'
import ProjectModal from '@/components/Modals/ProjectModal.vue'
import UnitModal from '@/components/Modals/UnitModal.vue'
import { Button, FormControl, FeatherIcon, call } from 'frappe-ui'
import { ref, computed, onMounted, watch } from 'vue'
import { useRouter, RouterLink } from 'vue-router'

const router = useRouter()

/* ---------- Tabs ---------- */
const activeTab = ref('projects')

/* ---------- UI ---------- */
const showProjectModal = ref(false)
const modalProject = ref(null)
const showUnitModal = ref(false)
const editingUnit = ref(null)

const loadingRows = ref(true)
const loadingUnits = ref(false)
const filterPanelOpen = ref(false)

/* ---------- Projects Data ---------- */
const rowsBuf = ref([])
const rows = computed(() => rowsBuf.value)

/* Unit counts per project (used on project cards) */
const availableCounts = ref({})
const totalCounts = ref({})

/* ---------- Units Data (Units tab from Unit doctype) ---------- */
const units = ref([])

/* ---------- Quick filters (Projects tab) ---------- */
const filters = ref({ q: '', location: '', status: '' })
const locationOptions = computed(() => {
  const uniq = [...new Set((rows.value || []).map(r => r.location).filter(Boolean))]
  return [{ label: __('All'), value: '' }, ...uniq.map(v => ({ label: v, value: v })) ]
})
const statusOptions = computed(() => {
  const uniq = [...new Set((rows.value || []).map(r => r.status).filter(Boolean))]
  return [{ label: __('All'), value: '' }, ...uniq.map(v => ({ label: v, value: v })) ]
})

/* ---------- Advanced Filters (Projects tab) ---------- */
const advFilters = ref([])
const newFilter  = ref({ field: '', op: '', value: '', value2: '' })

const filterFieldOptions = ref([
  { label: 'Project Name', value: 'project_name', fieldtype: 'Data' },
  { label: 'Developer',    value: 'developer',    fieldtype: 'Data' },
  { label: 'Location',     value: 'location',     fieldtype: 'Data' },
  { label: 'Status',       value: 'status',       fieldtype: 'Select' },
  { label: 'Categories',   value: 'categories',   fieldtype: 'Select' },
  { label: 'City',         value: 'city',         fieldtype: 'Data' },
  { label: 'District',     value: 'district',     fieldtype: 'Data' },
])

function fieldMeta(fname) { return filterFieldOptions.value.find(o => o.value === fname) || { fieldtype: 'Data', label: fname } }
function fieldLabel(fname){ return fieldMeta(fname).label || fname }

const OP_MAP = [
  { v: 'equals', l: 'equals' }, { v: 'not_equals', l: 'not equals' },
  { v: 'contains', l: 'contains' }, { v: 'not_contains', l: 'not contains' },
  { v: 'startswith', l: 'starts with' }, { v: 'endswith', l: 'ends with' },
  { v: 'gt', l: '>' }, { v: 'gte', l: '>=' }, { v: 'lt', l: '<' }, { v: 'lte', l: '<=' },
  { v: 'between', l: 'between' }, { v: 'in', l: 'in (csv)' }, { v: 'not_in', l: 'not in (csv)' },
  { v: 'is_set', l: 'is set' }, { v: 'is_not_set', l: 'is not set' },
]
function opLabel(op) { return (OP_MAP.find(o => o.v === op)?.l) || op }
function operatorOptionsFor(fname) {
  const t = fieldMeta(fname).fieldtype
  const base = ['equals','not_equals','contains','not_contains','startswith','endswith','in','not_in','is_set','is_not_set']
  const num  = ['equals','not_equals','gt','gte','lt','lte','between','is_set','is_not_set']
  const date = ['equals','not_equals','gt','gte','lt','lte','between','is_set','is_not_set']
  if (['Int','Float','Currency','Percent'].includes(t)) return num.map(v => ({ label: opLabel(v), value: v }))
  if (['Date','Datetime'].includes(t)) return date.map(v => ({ label: opLabel(v), value: v }))
  return base.map(v => ({ label: opLabel(v), value: v }))
}
function inputTypeFor(fname, op) {
  const t = fieldMeta(fname).fieldtype
  if (op === 'is_set' || op === 'is_not_set') return 'text'
  if (['Int','Float','Currency','Percent'].includes(t)) return 'number'
  if (['Date','Datetime'].includes(t)) return 'date'
  return 'text'
}
function showValueInput(op){ return !['is_set','is_not_set'].includes(op) }
const canAddFilter = computed(() => {
  if (!newFilter.value.field || !newFilter.value.op) return false
  if (newFilter.value.op === 'between') return newFilter.value.value !== '' && newFilter.value.value2 !== ''
  if (showValueInput(newFilter.value.op)) return newFilter.value.value !== ''
  return true
})
function addFilter()        { advFilters.value.push({ ...newFilter.value }); newFilter.value = { field:'', op:'', value:'', value2:'' } }
function removeFilter(i)    { advFilters.value.splice(i, 1) }
function clearAdvFilters()  { advFilters.value = [] }
function toggleFilterPanel(){ filterPanelOpen.value = !filterPanelOpen.value }

/* ---------- Data fetch ---------- */
onMounted(() => {
  fetchProjects()
  fetchUnits()
})

watch(activeTab, (t) => {
  if (t === 'units' && !units.value.length && !loadingUnits.value) fetchUnits()
})

async function fetchProjects() {
  loadingRows.value = true
  try {
    const res = await call('frappe.client.get_list', {
      doctype: 'Real Estate Project',
      fields: [
        'name','project_name','location','developer','cover_image',
        'properties_count','categories','status','city','district'
      ],
      limit_page_length: 300,
      order_by: 'modified desc',
    })
    rowsBuf.value = Array.isArray(res) ? res : (Array.isArray(res?.message) ? res.message : [])
    await fetchUnitCounts(rowsBuf.value)
  } catch (e) {
    console.error('Error fetching projects:', e)
    rowsBuf.value = []
    availableCounts.value = {}
    totalCounts.value = {}
  } finally {
    loadingRows.value = false
  }
}

/* If your project unit counts still come from Project Unit, we keep this as-is. */
async function fetchUnitCounts(projects) {
  const names = (projects || []).map(p => p.name).filter(Boolean)
  if (!names.length) {
    availableCounts.value = {}
    totalCounts.value = {}
    return
  }
  try {
    const unitsRes = await call('frappe.client.get_list', {
      doctype: 'Project Unit',
      fields: ['name', 'project', 'status'],
      filters: { project: ['in', names] },
      limit_page_length: 10000,
      order_by: 'modified desc',
    })
    const avail = {}
    const total = {}
    const banned = new Set(['sold', 'reserved'])
    for (const u of (Array.isArray(unitsRes) ? unitsRes : [])) {
      const proj = u.project
      if (!proj) continue
      const status = String(u.status || '').toLowerCase()
      total[proj] = (total[proj] || 0) + 1
      if (!banned.has(status)) avail[proj] = (avail[proj] || 0) + 1
    }
    availableCounts.value = avail
    totalCounts.value = total
  } catch (e) {
    console.error('Error fetching unit counts:', e)
    availableCounts.value = {}
    totalCounts.value = {}
  }
}

/* --------- UNITS TAB: fetch from Unit doctype --------- */
async function fetchUnits() {
  loadingUnits.value = true
  try {
    const res = await call('frappe.client.get_list', {
      doctype: 'Unit',
      fields: [
        'name','unit_name','type','area_sqm','price','description',
        'availability','cover_image' // ⬅ removed 'project'
      ],
      order_by: 'modified desc',
      limit_page_length: 1000,
    })

    let list = []
    if (Array.isArray(res)) list = res
    else if (Array.isArray(res?.message)) list = res.message
    else if (Array.isArray(res?.data)) list = res.data
    else if (Array.isArray(res?.results)) list = res.results

    units.value = list
  } catch (e) {
    console.error('[Inventory] fetchUnits error:', e)
    units.value = []
  } finally {
    loadingUnits.value = false
  }
}

/* ---------- Filtering (Projects tab) ---------- */
const filteredRows = computed(() => {
  const q   = (filters.value.q || '').toLowerCase().trim()
  const loc = filters.value.location || ''
  const stat= filters.value.status || ''
  return rows.value.filter(p => {
    if (q) {
      const hay = [p.project_name, p.name, p.developer, p.location].filter(Boolean).join(' ').toLowerCase()
      if (!hay.includes(q)) return false
    }
    if (loc && (p.location || '') !== loc) return false
    if (stat && p.status !== stat) return false
    for (const f of advFilters.value) { if (!matchFilter(p, f)) return false }
    return true
  })
})

function matchFilter(row, f) {
  const meta = fieldMeta(f.field)
  const v  = row?.[f.field]
  const sv = String(v ?? '')
  const a  = sv.toLowerCase()
  const b  = String(f.value ?? '').toLowerCase()
  const b2 = String(f.value2 ?? '').toLowerCase()

  if (f.op === 'is_set')     return sv !== ''
  if (f.op === 'is_not_set') return sv === ''

  if (['Int','Float','Currency','Percent'].includes(meta.fieldtype)) {
    const n  = Number(v); const n1 = Number(f.value); const n2 = Number(f.value2)
    if (f.op === 'equals') return n === n1
    if (f.op === 'not_equals') return n !== n1
    if (f.op === 'gt') return n > n1
    if (f.op === 'gte') return n >= n1
    if (f.op === 'lt') return n < n1
    if (f.op === 'lte') return n <= n1
    if (f.op === 'between') return n >= n1 && n <= n2
  }

  if (['Date','Datetime'].includes(meta.fieldtype)) {
    if (f.op === 'equals') return sv === f.value
    if (f.op === 'not_equals') return sv !== f.value
    if (f.op === 'gt') return sv > f.value
    if (f.op === 'gte') return sv >= f.value
    if (f.op === 'lt') return sv < f.value
    if (f.op === 'lte') return sv <= f.value
    if (f.op === 'between') return sv >= f.value && sv <= f.value2
  }

  if (f.op === 'equals')      return a === b
  if (f.op === 'not_equals')  return a !== b
  if (f.op === 'contains')    return a.includes(b)
  if (f.op === 'not_contains')return !a.includes(b)
  if (f.op === 'startswith')  return a.startsWith(b)
  if (f.op === 'endswith')    return a.endsWith(b)
  if (f.op === 'in')          { const set = new Set(b.split(',').map(s => s.trim())); return set.has(a) }
  if (f.op === 'not_in')      { const set = new Set(b.split(',').map(s => s.trim())); return !set.has(a) }
  return true
}

/* ---------- Helpers ---------- */
function availableCount(p) {
  const name = p?.name
  if (name && availableCounts.value[name] != null) return availableCounts.value[name]
  return Number(p?.properties_count ?? 0)
}
function totalCount(p) {
  const name = p?.name
  return name && totalCounts.value[name] != null ? totalCounts.value[name] : 0
}
function clearAll(){ filters.value = { q:'', location:'', status:'' }; clearAdvFilters() }
function imgSrc(val){
  if (!val) return ''
  const p = String(val).trim()
  if (/^https?:\/\//i.test(p)) return p
  if (p.startsWith('/private/files/')) {
    const enc = encodeURIComponent(p)
    return '/api/method/frappe.utils.file_manager.download_file?file_url=' + enc
  }
  return p.startsWith('/') ? p : '/' + p
}
function onImgError(e){ e.target.style.display = 'none' }
function categoryLabel(raw){
  const s = (raw || '').toLowerCase()
  const com = s.includes('commercial')
  const adm = s.includes('administrative')
  if (com && adm) return 'Commercial & Administrative'
  if (com) return 'Commercial'
  if (adm) return 'Administrative'
  return raw || '-'
}
function fmt(v){
  if (v === null || v === undefined || v === '') return '—'
  const n = Number(v)
  return Number.isFinite(n) ? n : String(v)
}

/* ---------- Actions ---------- */
function openProjectModal(project = null) { modalProject.value = project ? { ...project } : null; showProjectModal.value = true }
function openUnitModal(unit = null)      { editingUnit.value = unit ? { ...unit } : null; showUnitModal.value = true }

function goToProject(project) {
  const name = project?.name || project?.project_name
  if (!name) return
  router.push({ name: 'ProjectView', params: { project: name } })
}
function goToUnit(u) {
  if (!u?.name) return
  router.push({ name: 'UnitView', params: { unit: u.name } })
}

async function deleteProject(project) {
  if (!project?.name) return
  if (!confirm(__('Delete project “{0}”? This cannot be undone.', [project.project_name || project.name]))) return
  try {
    await call('frappe.client.delete', { doctype: 'Real Estate Project', name: project.name })
    rowsBuf.value = rowsBuf.value.filter(p => p.name !== project.name)
    const a = { ...availableCounts.value }, t = { ...totalCounts.value }
    delete a[project.name]; delete t[project.name]
    availableCounts.value = a; totalCounts.value = t
  } catch (e) {
    console.error(e)
    alert(e?.message || __('Could not delete project'))
  }
}

async function deleteUnit(u) {
  if (!u?.name) return
  if (!confirm(__('Delete unit “{0}”?', [u.unit_name || u.name]))) return
  try {
    await call('frappe.client.delete', { doctype: 'Unit', name: u.name })
    await fetchUnits()
    if (rows.value.length) await fetchUnitCounts(rows.value) // keeps project cards in sync
  } catch (e) {
    console.error(e)
    alert(e?.message || __('Could not delete unit'))
  }
}

async function onUnitSaved() {
  await fetchUnits()
  if (rows.value.length) await fetchUnitCounts(rows.value)
}
</script>

<style scoped>
:deep(.card) { border-radius: 0.75rem; }
.line-clamp-2{
  display:-webkit-box;
  -webkit-line-clamp:2;
  -webkit-box-orient:vertical;
  overflow:hidden;
}
</style>
