<script setup lang="ts">
import type { HTMLAttributes } from 'vue'
import { cn } from '@/lib/utils'

const props = defineProps<{ class?: HTMLAttributes['class'] }>()
</script>

<template>
  <div
    :class="cn('flex items-center justify-between gap-1 border-t border-border px-2 py-1.5', props.class)"
    v-bind="$attrs"
  >
    <slot />
  </div>
</template>
