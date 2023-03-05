<script setup lang="ts">
const { modelValue } = defineModel<{
  modelValue: string
}>()
</script>

<template>
  <input
    id="input"
    v-model="modelValue"
    type="text"
    v-bind="$attrs"
    p="x-4 y-2"
    w-80
    text="center"
    bg="transparent"
    border rounded border-gray-2 dark:border-gray-7
    outline="none active:none"
  >
</template>
