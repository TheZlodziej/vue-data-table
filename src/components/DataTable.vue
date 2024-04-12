<script setup lang="ts">
import draggable from 'vuedraggable'
import { watch, onBeforeMount, shallowRef, reactive, computed, ref } from 'vue'

/*
Example usage:
  data format:
    const data = [
      { h1: 1, h2: 2, h3: true  },
      { h1: 4, h2: 5, h3: false },
      { h1: 7, h2: 8, h3: false }
    ]

    const headers = ['h1', 'h2', 'h3']

    <DataTable :data="data" :headers="headers" />


  custom columns display:
    const customColumns = {
      h1: TextColumn // TextColumn is vue component with data property
    }

    you can also override columns inside DataTable body with dynamic slot:
    <template #col(column header)={data}>,
    for example:

    <DataTable ...>
      <template #col(h1)={data}>
         <h1>{{ data }}</h1>
      <template />
    <DataTable />


  filtering data:
    filtering works by passing object of column-name:filter-value.
    you an also modify what the filter value has to be in order to show the value by defining object
    with value (filter value) and modifier (function that takes in filter value and cell value and returns boolean)
    that will define filter behavior,
    for example:

    const filters = reactive({
      global: '',
      h1: '',
      h2: '',
      h3: {
        value: '',
        modifier: (filterV: string, cellV: boolean) => {
          const v = filterV.toLowerCase()
          if (cellV) {
            return v === 'yes'
          } else {
            return v === 'no'
          }
        }
      }
    })

    <DataTable ... :filters="filters" />


  paginator:
    in order to add paginator to the table simply add maxRows property to the data table,
    for example:

    <DataTable ... :maxRows="10" />
*/

type CellValue = number | string | boolean | object
type RowEntry = { [key: string]: CellValue }
type DataType = RowEntry[]
type Header = string

type ExtractSymbolKeys<T> = {
  [K in keyof T]: T[K] extends symbol | string | number ? K : never;
}[keyof T];

const props = withDefaults(
  defineProps<{
    data: DataType,
    dataIdKey: ExtractSymbolKeys<DataType>,
    headers: {
      key: Header,
      displayName?: string,
      hidden?: boolean,
    }[]
    customColumns: { [key: Header]: any }
    filters: {
      [key: Header]:
      | string
      | { value: string; modifier: (filterVal: string, cellVal: CellValue) => boolean }
    } & {
      global?: string
    },
    maxRows?: number,
    translations: {
      of: string,
    }
  }>(),
  {
    customColumns: () => ({}),
    filters: () => ({}),
    translations: () => ({
      of: "of",
    })
  }
)
const displayedData = shallowRef<DataType>([])

/* COMMON */
function applyModifiers() {
  updateFilteredData()
  sortRows()
}
/* END COMMON */

/* FILTERS */
function updateFilteredData() {
  currentPage.value = 1; // reset paginator

  const definedFilters = Object.keys(props.filters)
  if (definedFilters.length === 0) {
    displayedData.value = props.data
    return
  }

  const isGlobalFilter = props.filters.global !== undefined

  displayedData.value = props.data.filter((dataRow) => {
    let isFilteredOut = false
    const isGloballyFilteredOut =
      isGlobalFilter &&
      props.filters.global!.length &&
      !Object.values(dataRow).some((val) => {
        return String(val).toLowerCase().includes(props.filters.global!.toLowerCase())
      })

    for (const [dataKey, dataValue] of Object.entries(dataRow)) {
      const keyInFilters = Object.prototype.hasOwnProperty.call(props.filters, dataKey)
      if (!keyInFilters) {
        continue
      }

      const filterValue = props.filters[dataKey]
      if (typeof filterValue === 'object') {
        if (filterValue.value.length && !filterValue.modifier(filterValue.value, dataValue)) {
          isFilteredOut = true
        }
      } else {
        if (!String(dataValue).toLowerCase().includes(filterValue.toLocaleLowerCase())) {
          isFilteredOut = true
        }
      }
    }

    return !isFilteredOut && !isGloballyFilteredOut
  })
}

watch(() => props.filters, applyModifiers, { deep: true })
watch(() => props.data, applyModifiers, { deep: true })
/* END FILTERS */

/* SORTING */
type SortDirection = 'None' | 'Ascending' | 'Descending'
const sortDirection = reactive<{ [key: Header]: SortDirection }>(
  Object.fromEntries(props.headers.map((h) => [h.key, 'None']))
)
const sortInvocationOrder: Header[] = []

function sortRows(maybeHeader?: Header) {
  const getOrderCorrection = (header: Header) => {
    const order = sortDirection[header]
    switch (order) {
      case 'Ascending':
        sortDirection[header] = 'Descending'
        return -1

      case 'Descending':
        sortDirection[header] = 'None'
        sortInvocationOrder.splice(sortInvocationOrder.indexOf(header), 1)
        applyModifiers()
        return 0

      case 'None':
        sortDirection[header] = 'Ascending'
        return 1
    }
  }

  const updateOrder = (header: Header, orderCorrection: number = 1) => {
    if (sortDirection[header] === 'None') {
      return
    }

    displayedData.value = [
      ...displayedData.value.sort((rowA, rowB) => {
        const [valA, valB] = [rowA[header], rowB[header]]
        if (valA > valB) return orderCorrection
        if (valA < valB) return -orderCorrection
        return 0
      })
    ]
  }

  if (maybeHeader === undefined) {
    for (const header of sortInvocationOrder) {
      updateOrder(header)
    }

    return
  }

  const sortOrderIdx = sortInvocationOrder.indexOf(maybeHeader)
  if (sortOrderIdx !== -1) {
    sortInvocationOrder.splice(sortOrderIdx, 1)
  }
  sortInvocationOrder.push(maybeHeader)

  const orderCorrection = getOrderCorrection(maybeHeader)
  updateOrder(maybeHeader, orderCorrection)
}
/* END SORTING */

/* PAGINATOR */
const currentPage = shallowRef(1)
const paginatorMaxPage = computed(() => Math.ceil(displayedData.value.length / (props.maxRows ?? 1)))
const paginatorCurrentMinItem = computed(() => (currentPage.value - 1) * (props.maxRows ?? 0) + (displayedData.value.length ? 1 : 0));
const paginatorCurrentMaxItem = computed(() => Math.min(currentPage.value * (props.maxRows ?? 0), displayedData.value.length));
/* END PAGINATOR */

/* EXPANDED ROWS */
const expandedRows = ref(Object.fromEntries(props.data.map((row) => [row[props.dataIdKey], false])))
/* END EXPANDED ROWS */

/* INIT */
onBeforeMount(() => {
  updateFilteredData()
})
/* END INIT */
</script>

<template>
  <table>
    <thead>
      <tr v-if="$slots.expand">
        <th rowspan="2"></th>
      </tr>
      <draggable :list="props.headers" tag="tr" item-key="key">
        <template #item="{ element }">
          <th @click="() => sortRows(element.key)" v-if="!element.hidden">
            <slot :data="element" :sortDirection="sortDirection[element.key]" name="headerItem">
              {{ element.displayName ?? element.key }}
              <span v-if="sortDirection[element.key] === 'Ascending'">↑</span>
              <span v-else-if="sortDirection[element.key] === 'Descending'">↓</span>
            </slot>
          </th>
        </template>
      </draggable>
    </thead>

    <tbody>
      <template v-for="(dataEntry, rowIdx) in displayedData" :key="`r:${rowIdx}`">
        <tr v-if="!props.maxRows ||
        (rowIdx >= props.maxRows * (currentPage - 1) && rowIdx < props.maxRows * currentPage)
        ">
          <td v-if="$slots.expand"
            @click="expandedRows[dataEntry[props.dataIdKey]] = !expandedRows[dataEntry[props.dataIdKey]]"
            class="expander">
            <slot name="expander">
              <span v-if="expandedRows[dataEntry[props.dataIdKey]]">▲</span>
              <span v-else>▼</span>
            </slot>
          </td>

          <template v-for="header in props.headers" :key="`r:${rowIdx}h:${header.key}`">
            <td v-if="!header.hidden">
              <slot v-if="$slots[`col(${header.key})`]" :data="dataEntry[header.key]" :name="`col(${header.key})`" />
              <slot v-else :data="dataEntry[header.key]" name="dataItem">
                <template v-if="Object.keys(props.customColumns).includes(header.key)">
                  <component :is="props.customColumns[header.key]" :data="dataEntry[header.key]" />
                </template>
                <template v-else>
                  {{ dataEntry[header.key] }}
                </template>
              </slot>
            </td>
          </template>
        </tr>

        <tr v-if="expandedRows[dataEntry[props.dataIdKey]]">
          <td :colspan="headers.length + 1">
            <slot name="expand" :data="dataEntry"></slot>
          </td>
        </tr>
      </template>
    </tbody>
    <tfoot>
      <tr v-if="props.maxRows">
        <td :colspan="headers.filter(h => !h.hidden).length + ($slots.expand ? 1 : 0)">
          {{ paginatorCurrentMinItem }} - {{ paginatorCurrentMaxItem }} {{ translations.of }} {{ displayedData.length }}
          <button class="paginator-btn" @click="currentPage = 1">⇤</button>
          <button class="paginator-btn" @click="currentPage = Math.max(currentPage - 1, 1)">←</button>
          <button class="paginator-btn" @click="currentPage = Math.min(currentPage + 1, paginatorMaxPage)">→</button>
          <button class="paginator-btn" @click="currentPage = paginatorMaxPage">⇥</button>
        </td>
      </tr>
    </tfoot>
  </table>
</template>

<style scoped>
th
{
  font-size: 1.2em;
  cursor: move;
}

table,
th,
td
{
  color: rgb(159, 159, 159);
  border: 1px solid;
  border-collapse: collapse;
}

th
{
  font-weight: bold;
}

th,
td
{
  padding: 7px 13px;
}

.paginator-btn
{
  border: 1px solid;
  border-radius: 50%;
  aspect-ratio: 1/1;
  font-size: 1.1em;
  background: transparent;
  color: inherit;
  cursor: pointer;
}

.expander
{
  cursor: pointer;
}
</style>
