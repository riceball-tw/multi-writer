<script setup lang="ts">
import type { HTMLAttributes } from 'vue'
import { Button } from '@/components/ui/button'
import { cn } from '@/lib/utils'

interface Props {
  class?: HTMLAttributes['class']
  variant?: 'default' | 'outline' | 'ghost' | 'link' | 'destructive' | 'secondary'
  size?: 'default' | 'sm' | 'lg' | 'icon'
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'ghost',
  size: 'default',
})
</script>

<template>
  <Button
    type="button"
    :variant="variant"
    :size="size"
    :class="cn(props.class)"
    v-bind="$attrs"
  >
    <slot />
  </Button>
</template>
