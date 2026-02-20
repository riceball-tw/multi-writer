<script setup lang="ts">
import type { HTMLAttributes } from 'vue'
import { cn } from '@/lib/utils'
import { ref, computed } from 'vue'

interface Props {
  modelValue?: string
  placeholder?: string
  disabled?: boolean
  class?: HTMLAttributes['class']
  rows?: number
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: 'What would you like to know?',
  rows: 1,
})

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

const textareaRef = ref<HTMLTextAreaElement>()
const localValue = computed({
  get: () => props.modelValue ?? '',
  set: (value) => emit('update:modelValue', value),
})

function adjustHeight() {
  if (textareaRef.value) {
    textareaRef.value.style.height = 'auto'
    textareaRef.value.style.height = `${textareaRef.value.scrollHeight}px`
  }
}

function handleKeyDown(e: KeyboardEvent) {
  if (e.key === 'Enter' && (e.metaKey || e.ctrlKey)) {
    e.preventDefault()
    const form = textareaRef.value?.closest('form')
    if (form) {
      form.requestSubmit()
    }
  }
  adjustHeight()
}
</script>

<template>
  <textarea
    ref="textareaRef"
    v-model="localValue"
    name="message"
    :disabled="disabled"
    :placeholder="placeholder"
    :rows="rows"
    :class="cn(
      'field-sizing-content max-h-48 min-h-16 w-full resize-none border-0 bg-transparent px-3 py-2 text-sm outline-none',
      'placeholder:text-muted-foreground',
      'disabled:cursor-not-allowed disabled:opacity-50',
      props.class
    )"
    @keydown="handleKeyDown"
    @input="adjustHeight"
  />
</template>
