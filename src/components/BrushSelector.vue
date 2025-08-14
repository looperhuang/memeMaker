<template>
  <!-- 筆刷模式選擇 -->
  <q-select
    v-model="brush"
    :options="brushOptions"
    label="Brush"
    filled
    emit-value
    map-options
  />
</template>

<script setup lang="ts">
import type { QSelectOption } from 'quasar';
import { useVModel } from '@vueuse/core';

const props = defineProps<{
  modelValue: BrushKey;
}>();
const emit = defineEmits(['update:modelValue']);
const brush = useVModel(props, 'modelValue', emit);

export type BrushKey =
  | 'PencilBrush'
  | 'CircleBrush'
  | 'SprayBrush'
  | 'PatternBrush';

// 筆刷模式
const brushOptions: QSelectOption<BrushKey>[] = [
  { label: 'Pencil', value: 'PencilBrush' },
  { label: 'Circle', value: 'CircleBrush' },
  { label: 'Spray', value: 'SprayBrush' },
  { label: 'Pattern', value: 'PatternBrush' },
];
</script>
