<script setup lang="ts">
const props = defineProps<{
  data: any
}>()
</script>

<template>text-column {{ props.data }}</template>
