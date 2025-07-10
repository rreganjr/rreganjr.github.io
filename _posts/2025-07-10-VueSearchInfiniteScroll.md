---
layout: post
title: Using Vue V-Select For Selecting with Search and Infinite Scrolling of results
---

I used the v-select because of an issue using vuetify v-combobox and v-autocomplete where I couldn't get the drop down list to restore after clearing the typed filtering programatically. It would always show the last selected item.

```
> npm i vue-select@4.0.0-beta.6
```

## InstanceSelector.vue

```
<template>
  <v-select
      v-model="selectedInstance"
      :options="instanceItems"
      label="label"
      :filterable="false"
      :search="searchQuery"
      :clearable="true"
      :placeholder="'Select an Instance'"
      @search="onSearch"
      @open="onDropdownOpen"
      :key="autoKey"
  />
</template>

<script setup lang="ts">
import { ref, watch, onMounted, nextTick, onBeforeUnmount } from 'vue'
import vSelect from 'vue-select'
import 'vue-select/dist/vue-select.css'
import { debounce } from 'lodash-es'
import { useInstancesStore } from '~/stores/instances'

defineOptions({ components: { 'v-select': vSelect } })

const instancesStore = useInstancesStore()

const selectedInstance = ref<any>(null)
const searchQuery = ref<string>('')
const autoKey = ref(0)
const instanceItems = ref<{ id: string; label: string }[]>([])
const dropdownScrollTop = ref(0)
watch(
    () => instancesStore.instanceList,
    (newList) => {
      instanceItems.value = newList
          .slice() // clone array to avoid mutating the store
          .sort((a, b) => a.label.localeCompare(b.label))
          .map(i => ({
            id: i.id,
            label: i.label
          }))

      // after updating the list, put the user back to where they were
      nextTick(() => {
        const menu = document.querySelector('.vs__dropdown-menu') as HTMLElement | null
        if (menu && dropdownScrollTop.value) {
          menu.scrollTop = dropdownScrollTop.value
        }
      })
    },
    { immediate: true }
)

watch(selectedInstance, async (option) => {
  console.log('[InstanceSelector] selectedInstance changed:', option)
  if (option && option.id) {
    await instancesStore.setCurrentInstanceId(option.id)
    selectedInstance.value = null
    searchQuery.value = ''
    instancesStore.clearFilters()
    await instancesStore.fetchInstances(true)
  }
})

onMounted(async () => {
  // Only show Partner instances
  await instancesStore.setFilters([{
    field: 'accountType',
    operator: 'EQUALS',
    value: 'Partner'
  }])
})

// --- Infinite Scroll (real DOM listener) ---
function onDropdownOpen() {
  nextTick(() => {
    // Vue Select 4.x uses this class for the menu
    const menu = document.querySelector('.vs__dropdown-menu')
    if (menu) {
      menu.addEventListener('scroll', handleDropdownScroll)
    }
  })
}

async function handleDropdownScroll(e: Event) {
  const target = e.target as HTMLElement
  dropdownScrollTop.value = target.scrollTop
  const threshold = 50
  if (target.scrollTop + target.clientHeight >= target.scrollHeight - threshold) {
    await instancesStore.loadMore()
  }
}

// Cleanup on unmount
onBeforeUnmount(() => {
  const menu = document.querySelector('.vs__dropdown-menu')
  if (menu) {
    menu.removeEventListener('scroll', handleDropdownScroll)
  }
})

const debouncedFetch = debounce(async (val: string) => {
  instancesStore.filterLabel = val
  await instancesStore.fetchInstances(true)
}, 500)

function onSearch(val: string) {
  searchQuery.value = val
  debouncedFetch(val)
}
</script>

<style>
div.v-select {
  min-width: 200px;
  max-width: 600px;
}
input.vs__search {
  width: 100%;
}
.vs__dropdown-menu {
  background: #fff;
  color: #222;
  z-index: 9999;
}
.vs__dropdown-option {
  color: #222;
}
.vs__dropdown-option--highlight {
  background: #1976d2;
  color: #fff;
}
.vs__selected, .vs__dropdown-option {
  font-size: 1rem;
}
</style>
```

## Instances.ts

```
// stores/instances.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { SortDirection, SortField, FilterOperator, FilterCriterion, SearchRequest, InstanceListItem, InstanceListItem from 'instanceService.ts'

export const useInstancesStore = defineStore('instances', () => {
    // ─── State ───────────────────────────────────────────────────────────────────

    const instances = ref<Record<string, InstanceListItem>>({})
    const currentInstanceId = ref<string | null>(null)
    const filterLabel = ref<string>('')

    const currentPage = ref<number>(0)
    const pageSize = ref<number>(20)
    const hasMoreResults = ref<boolean>(true)

    const filters = ref<FilterCriterion[]>([
    ])
    // ─── Computed ────────────────────────────────────────────────────────────────

    const currentInstance = ref<Instance | null>(null)

    const instanceList = computed(() => Object.values(instances.value))

    // ─── Services ────────────────────────────────────────────────────────────────

    const {
        searchMyInstances,
        instanceDetails
    } = useInstanceService()

    const loading = ref(false)

    // ─── Actions ─────────────────────────────────────────────────────────────────

    /**
     * initialize the filters and fetch the initial selection
     * @param newFilters
     */
    async function setFilters(newFilters: FilterCriterion[]) {
        filters.value = newFilters
        await fetchInstances(true)
    }

    async function fetchInstances(reset: boolean = false) {
        if (!loading.value) {
            try {
                loading.value = true
                if (reset) {
                    currentPage.value = 0
                    hasMoreResults.value = true
                    instances.value = {}
                }

                if (!hasMoreResults.value) return

                const request: SearchRequest = {
                    page: currentPage.value,
                    size: pageSize.value,
                    filters: [...filters.value]
                }

                if (filterLabel.value.trim()) {
                    request.filters!.push({
                        field: 'label',
                        operator: 'LIKE',
                        value: `%${filterLabel.value.trim()}%`
                    })
                }

                const results = await searchMyInstances(request)

                if (results.length < pageSize.value) {
                    hasMoreResults.value = false
                }

                results.forEach(instance => {
                    if (instance.id) instances.value[instance.id] = instance
                })

                currentPage.value++
            } finally {
                loading.value = false
            }
        }
    }

    async function loadMore() {
        await fetchInstances()
    }

    async function setCurrentInstanceId(id: string | null) {
        console.log(`[instances.ts] setCurrentInstanceId(id=${id}`)
        if (id) {
            const instance = await instanceDetails(id)
            if (instance) {
                currentInstance.value = instance
            }
        }
        currentInstanceId.value = id
    }

    // ─── Helpers ─────────────────────────────────────────────────────────────────

    function clearFilters() {
        filterLabel.value = ''
        currentPage.value = 0
        hasMoreResults.value = true
    }

    return {
        // state
        instances,
        currentInstanceId,
        filterLabel,
        currentPage,
        pageSize,
        hasMoreResults,
        currentInstance,

        // computed
        instanceList,

        // actions
        setFilters,
        fetchInstances,
        loadMore,
        setCurrentInstanceId,
        clearFilters
    }
})
```

## instanceService.ts

```
import { useNuxtApp } from '#app'

const baseUrl = '/content/instance'

export interface InstanceListItem {
  id: string
  label: string
  accountType: string
  visibility: string
  logoUrl?: string
  clientViewUrl?: string
}

export interface Instance extends InstanceListItem {
  description: string
}

export type SortDirection = 'ASC' | 'DESC'

export interface SortField {
  field: string
  direction: SortDirection
}

export type FilterOperator =
  | 'EQUALS'
  | 'LIKE'
  | 'IN'
  | 'GT'
  | 'LT'
  | 'GTE'
  | 'LTE'
  | 'NEQ'

export interface FilterCriterion {
  field: string
  operator: FilterOperator
  value: string | number | boolean | Array<string | number | boolean>
}

export interface SearchRequest {
  page: number
  size: number
  sort?: SortField[]
  filters?: FilterCriterion[]
}

export function useInstanceService() {
  const { $qfetch } = useNuxtApp()

  /**
   * note: InstanceListItem maps to AccountSiteViewModel in quik
   * @returns all instances this user is assigned a role
   */
  async function listMyInstances(): Promise<InstanceListItem[]> {
    return await $qfetch(`${baseUrl}`)
  }

  /**
   * Searches instances based on provided SearchRequest criteria
   *
   note: InstanceListItem maps to AccountSiteViewModel in quik
   * @param request Search criteria including pagination, sorting, and filtering
   */
  async function searchMyInstances(request: SearchRequest): Promise<InstanceListItem[]> {
    return await $qfetch(`${baseUrl}`, {
      method: 'POST',
      body: request,
    })
  }

  async function instanceDetails(id: string): Promise<Instance> {
    return await $qfetch(`${baseUrl}/${id}`)
  }

  return {
    searchMyInstances,
    listMyInstances,
    instanceDetails
  }
}
```
