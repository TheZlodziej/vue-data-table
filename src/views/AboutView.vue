<script setup lang="ts">
import DataTable from '../components/DataTable.vue'
import { ref, computed, watch } from 'vue'
import TextColumn from '../components/columns/TextColumn.vue'
import AlertColumn from '../components/columns/AlertColumn.vue'

const headers = ref([
  { vis: true, val: 'h1' },
  { vis: true, val: 'h2' },
  { vis: true, val: 'h3' }
])

const visibleHeaders = computed(() => headers.value.filter((h) => h.vis).map((h) => h.val))

const customColumns: { [key: string]: any } = {
  h1: TextColumn,
  h2: AlertColumn
}

const data = [
  { h1: 1, h2: 2, h3: 3 },
  { h1: 4, h2: 5, h3: 6 },
  { h1: 7, h2: 8, h3: 9 }
]
</script>

<template>
  <div>
    <div v-for="h in headers" :key="h.val">
      <input
        type="checkbox"
        :value="h.vis"
        @change="
          () => {
            h.vis = !h.vis
          }
        "
        style="width: 20px; height: 20px"
      />
      {{ h.val }}
    </div>
  </div>

  <DataTable v-bind:headers="visibleHeaders" :data="data" :customColumns="customColumns">
    <template #headerItem="{ data }">
      <h2>
        {{ data }}
      </h2>
    </template>

    <template #col(h3)="{ data }">h3:{{ data }} </template>
  </DataTable>
</template>
