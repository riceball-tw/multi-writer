<script setup lang="ts">
import { CommandItem } from '@/components/ui/command'
import type { ListboxItemProps } from 'reka-ui'

interface Props extends ListboxItemProps {
  // value is required by ListboxItemProps
}

const props = defineProps<Props>()
</script>

<template>
  <CommandItem v-bind="props">
    <slot />
  </CommandItem>
</template>
