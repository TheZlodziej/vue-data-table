<script setup lang="ts">
const props = defineProps<{
  data: number
}>()

function showAlert() {
  alert(props.data)
}
</script>

<template>
  <button @click="showAlert">{{ props.data }}</button>
</template>
