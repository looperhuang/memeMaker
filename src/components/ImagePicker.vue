<template>
  <q-file filled v-model="model" label="上傳圖片" accept="image/*" />
</template>

<script setup lang="ts">
import { useVModel } from '@vueuse/core';

const props = defineProps<{
  modelValue: File | null;
}>();
const emit = defineEmits(['update:modelValue']);
const model = useVModel(props, 'modelValue', emit);
</script>

<style scoped></style>
