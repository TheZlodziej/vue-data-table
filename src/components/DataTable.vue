<script setup lang="ts">
import draggable from 'vuedraggable'
import { watch, onBeforeMount, shallowRef } from 'vue'

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
    that will define filter behavior, eg.

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
  }>(),
  {
    customColumns: () => ({}),
    filters: () => ({})
  }
)

/* HEADERS */
type HeadersModel = {
  __id: string
  header: string
}

const headersModel = shallowRef<HeadersModel[]>([])

const updateHeadersModel = (value: Header[]) => {
  headersModel.value = value.map((header) => ({
    __id: `h:${header}`,
    header
  }))
}
watch(() => props.headers, updateHeadersModel, { deep: true })
/* END HEADERS */

/* FILTERS */
const filteredData = shallowRef<DataType>([])

const updateFilteredData = (value: DataType) => {
  const definedFilters = Object.keys(props.filters)
  if (definedFilters.length === 0) {
    filteredData.value = value
    return
  }

  // write a function that filters data based on filter values and puts the result in filteredData
  const isGlobalFilter = props.filters.global !== undefined

  filteredData.value = props.data.filter((dataRow) => {
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

watch(() => props.filters, updateFilteredData, { deep: true })
watch(() => props.data, updateFilteredData, { deep: true })
/* END FILTERS */

/* INIT */
onBeforeMount(() => {
  updateHeadersModel(props.headers)
  updateFilteredData(props.data)
})
/* END INIT */
</script>

<template>
  <table>
    <thead>
      <draggable v-model="headersModel" tag="tr" item-key="__id">
        <template #item="{ element }">
          <th>
            <slot :data="element.header" name="headerItem">{{ element.header }}</slot>
          </th>
        </template>
      </draggable>
    </thead>

    <tbody>
      <tr v-for="(dataEntry, rowIdx) in filteredData" :key="`r:${rowIdx}`">
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
