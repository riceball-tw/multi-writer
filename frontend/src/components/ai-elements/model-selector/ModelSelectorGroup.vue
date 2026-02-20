<script setup lang="ts">
import type { HTMLAttributes } from 'vue'
import { CommandGroup } from '@/components/ui/command'

interface Props {
  heading?: string
  class?: HTMLAttributes['class']
}

const props = defineProps<Props>()
</script>

<template>
  <CommandGroup :heading="heading" v-bind="$attrs">
    <slot />
  </CommandGroup>
</template>
