<script setup lang="ts">
import DataTable from '../components/DataTable.vue'
import { ref, reactive, onMounted } from 'vue'
import TextColumn from '../components/columns/TextColumn.vue'
import AlertColumn from '../components/columns/AlertColumn.vue'
import { useLocalStorage } from "@vueuse/core"

const headers = ref([
  { hidden: false, key: 'h1', displayName: "header1" },
  { hidden: false, key: 'h2', displayName: "header2" },
  { key: 'h3', }
])

const filters = reactive({
  global: '',
  h1: '',
  h2: '',
  h3: {
    value: '',
    modifier: (filterVal: string, cellVal: boolean) => {
      const v = filterVal.toLowerCase()
      if (cellVal) {
        return v === 'tak'
      } else {
        return v === 'nie'
      }
    }
  }
})

const customColumns: { [key: string]: any } = {
  h1: TextColumn,
  h2: AlertColumn
}

const data = ref<{ h1: string; h2: number; h3: boolean }[]>([])

onMounted(() => {
  for (let i = 0; i < 500; i++) {
    data.value.push(generateRandomObject())
  }
})

let x = 1
function generateRandomObject() {
  const randomObject = {
    h1: generateRandomString(),
    h2: x++,
    h3: Math.random() < 0.5 // Generate a random boolean for h3
  }
  return { ...randomObject, h4: randomObject }
}

function generateRandomString() {
  const length = Math.floor(Math.random() * 10) + 1 // Generate a random string length between 1 and 10
  let result = ''
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * characters.length))
  }
  return result
}
</script>

<template>
  <div>
    <div v-for="filterKey in Object.keys(filters) as (keyof typeof filters)[]" :key="filterKey">
      filter {{ filterKey }}
      <input v-if="typeof filters[filterKey] === 'string'" v-model="filters[filterKey]" />
      <input v-else v-model="filters[filterKey].value" />
    </div>
    <div v-for="h in headers" :key="h.key">
      <input type="checkbox" :value="h.hidden" :checked="h.hidden" @change="() => {
      h.hidden = !h.hidden
    }
      " style="width: 20px; height: 20px" />
      {{ h.displayName ?? h.key }}
    </div>
    <button @click="() => data.push(generateRandomObject())">add row</button>
  </div>

  <DataTable dataIdKey="h2" v-bind:headers="headers" :data="data" :customColumns="customColumns" :filters="filters"
    :maxRows="10">
    <!-- <template #headerItem="{ data }">
      <h2>
        {{ data }}
      </h2>
    </template> -->

    <template #col(h3)="{ data }">
      <div v-if="data">🤙</div>
      <div v-else>😭</div>
    </template>

    <template #expand="{ data }">
      {{ Object.values(data.h4).join('\t') }}
    </template>
  </DataTable>
</template>
