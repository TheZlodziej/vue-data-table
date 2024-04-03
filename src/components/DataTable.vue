<script setup lang="ts">
import draggable from 'vuedraggable'
import { ref, watch, onBeforeMount } from 'vue'

/*
Example usage:
  data format:
    const data = [
      { h1: 1, h2: 2, h3: 3 },
      { h1: 4, h2: 5, h3: 6 },
      { h1: 7, h2: 8, h3: 9 }
    ]

    const headers = ['h1', 'h2', 'h3']

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
*/

const tableUUID = crypto.randomUUID()

const props = withDefaults(
  defineProps<{
    data: { [key: string]: number | string | boolean | object }[]
    headers: string[]
    customColumns: { [key: string]: any }
  }>(),
  {
    customColumns: () => ({})
  }
)

/* HEADERS */
type HeadersModel = {
  __id: string
  header: string
}

const headersModel = ref<HeadersModel[]>([])

const updateHeadersModel = (value: string[]) => {
  headersModel.value = value.map((header) => ({
    __id: `table:${tableUUID}&header:${header}`,
    header
  }))
}
watch(() => props.headers, updateHeadersModel, { deep: true })
/* END HEADERS */

/* COLUMNS */
/* END COLUMNS */

/* INIT */
onBeforeMount(() => {
  updateHeadersModel(props.headers)
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
      <tr v-for="(dataEntry, rowIdx) in props.data" :key="`${tableUUID}-r:${rowIdx}`">
        <td v-for="header in headersModel" :key="`${tableUUID}-r:${rowIdx}h:${header.header}`">
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
