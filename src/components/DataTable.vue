<script setup lang="ts">
import draggable from 'vuedraggable'
import { watch, onBeforeMount, shallowRef, reactive } from 'vue'

// TODO:
/*
- fix paginator max (doesnt update when items are added dynamically)
- fix column reorder:
  add option to save order to local storage (maybe order parameter?)
- add displayname to columns
*/

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
    in order to add paginator to the table simply add rowCount property to the data table,
    for example:

    <DataTable ... :rowCount="10" />
*/

type CellValue = number | string | boolean | object
type RowEntry = { [key: string]: CellValue }
type DataType = RowEntry[]
type Header = string

const props = withDefaults(
  defineProps<{
    data: DataType
    headers: Header[]
    customColumns: { [key: Header]: any }
    filters: {
      [key: Header]:
        | string
        | { value: string; modifier: (filterVal: string, cellVal: CellValue) => boolean }
    } & {
      global?: string
    }
    rowCount?: number
  }>(),
  {
    customColumns: () => ({}),
    filters: () => ({})
  }
)
const displayedData = shallowRef<DataType>([])

/* COMMON */
function applyModifiers() {
  updateFilteredData()
  sortRows()
}
/* END COMMON */

/* HEADERS */
type HeadersModel = {
  __id: string
  header: string
}

const headersModel = shallowRef<HeadersModel[]>([])

function updateHeadersModel() {
  headersModel.value = props.headers.map((header) => ({
    __id: `h:${header}`,
    header
  }))
}
watch(() => props.headers, updateHeadersModel, { deep: true })
/* END HEADERS */

/* FILTERS */
function updateFilteredData() {
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
  Object.fromEntries(props.headers.map((h) => [h, 'None']))
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
        sortDirection[header] = 'None' // TODO: fix later
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
/* END PAGINATOR */

/* INIT */
onBeforeMount(() => {
  updateHeadersModel()
  updateFilteredData()
})
/* END INIT */
</script>

<template>
  <div v-if="props.rowCount">
    Page:
    <input
      type="number"
      min="1"
      :max="displayedData.length / props.rowCount"
      v-model="currentPage"
    />
  </div>
  <br />
  <table>
    <thead>
      <draggable v-model="headersModel" tag="tr" item-key="__id" @update="(x) => console.log(x)">
        <template #item="{ element }">
          <th @click="() => sortRows(element.header)">
            <slot
              :data="element.header"
              :sortDirection="sortDirection[element.header]"
              name="headerItem"
            >
              {{ element.header }}
              <span v-if="sortDirection[element.header] === 'Ascending'">↑</span>
              <span v-else-if="sortDirection[element.header] === 'Descending'">↓</span>
            </slot>
          </th>
        </template>
      </draggable>
    </thead>

    <tbody>
      <template v-for="(dataEntry, rowIdx) in displayedData" :key="`r:${rowIdx}`">
        <tr
          v-if="
            !props.rowCount ||
            (rowIdx >= props.rowCount * (currentPage - 1) && rowIdx < props.rowCount * currentPage)
          "
        >
          <td v-for="header in headersModel" :key="`r:${rowIdx}h:${header.header}`">
            <slot
              v-if="$slots[`col(${header.header})`]"
              :data="dataEntry[header.header]"
              :name="`col(${header.header})`"
            />
            <slot v-else :data="dataEntry[header.header]" name="dataItem">
              <template v-if="Object.keys(props.customColumns).includes(header.header)">
                <component
                  :is="props.customColumns[header.header]"
                  :data="dataEntry[header.header]"
                />
              </template>
              <template v-else>
                {{ dataEntry[header.header] }}
              </template>
            </slot>
          </td>
        </tr>
      </template>
    </tbody>
  </table>
</template>

<style scoped>
table,
th,
td {
  border: 1px solid white;
  border-collapse: collapse;
}
</style>
