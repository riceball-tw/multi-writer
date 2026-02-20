<script setup lang="ts">
import type { HTMLAttributes } from 'vue'
import { cn } from '@/lib/utils'

interface Props {
  class?: HTMLAttributes['class']
}

const props = defineProps<Props>()

const emit = defineEmits<{
  'submit': [payload: { text: string }]
}>()

function onSubmit(e: Event) {
  e.preventDefault()
  const form = e.target as HTMLFormElement
  const formData = new FormData(form)
  const text = (formData.get('message') as string) || ''
  emit('submit', { text })
}
</script>

<template>
  <form
    :class="cn('w-full', props.class)"
    @submit.prevent="onSubmit"
  >
    <div class="rounded-md border border-input bg-background overflow-hidden">
      <slot />
    </div>
  </form>
</template>
