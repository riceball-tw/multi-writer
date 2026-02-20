<script setup lang="ts">
import type { HTMLAttributes } from 'vue'
import { cn } from '@/lib/utils'
import { CornerDownLeftIcon, Loader2Icon, SquareIcon, XIcon } from 'lucide-vue-next'
import { computed } from 'vue'
import PromptInputButton from './PromptInputButton.vue'

type ChatStatus = 'idle' | 'submitted' | 'streaming' | 'error'

interface Props {
  disabled?: boolean
  status?: ChatStatus
  class?: HTMLAttributes['class']
}

const props = withDefaults(defineProps<Props>(), {
  status: 'idle',
})

const icon = computed(() => {
  if (props.status === 'submitted') {
    return Loader2Icon
  }
  else if (props.status === 'streaming') {
    return SquareIcon
  }
  else if (props.status === 'error') {
    return XIcon
  }
  return CornerDownLeftIcon
})

const iconClass = computed(() => {
  if (props.status === 'submitted') {
    return 'size-4 animate-spin'
  }
  return 'size-4'
})
</script>

<template>
  <PromptInputButton
    type="submit"
    variant="default"
    size="icon-sm"
    :disabled="disabled"
    :class="cn('shrink-0', props.class)"
    aria-label="Submit"
  >
    <component :is="icon" :class="iconClass" />
  </PromptInputButton>
</template>
