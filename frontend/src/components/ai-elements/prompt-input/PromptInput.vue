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
  placeholder: 'Enter your message...',
  rows: 3,
})

const emit = defineEmits<{
  'update:modelValue': [value: string]
  'submit': []
}>()

const textareaRef = ref<HTMLTextAreaElement>()
const localValue = computed({
  get: () => props.modelValue ?? '',
  set: (value) => emit('update:modelValue', value),
})

function handleKeyDown(e: KeyboardEvent) {
  if (e.key === 'Enter' && (e.metaKey || e.ctrlKey)) {
    e.preventDefault()
    emit('submit')
  }
}

function adjustHeight() {
  if (textareaRef.value) {
    textareaRef.value.style.height = 'auto'
    textareaRef.value.style.height = `${textareaRef.value.scrollHeight}px`
  }
}
</script>

<template>
  <div :class="cn('relative w-full', props.class)">
    <textarea
      ref="textareaRef"
      v-model="localValue"
      :disabled="disabled"
      :placeholder="placeholder"
      :rows="rows"
      :class="cn(
        'flex w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background',
        'placeholder:text-muted-foreground',
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2',
        'disabled:cursor-not-allowed disabled:opacity-50',
        'resize-none overflow-hidden',
        props.class
      )"
      @keydown="handleKeyDown"
      @input="adjustHeight"
    />
    <div class="absolute bottom-2 right-2 text-xs text-muted-foreground">
      <kbd class="pointer-events-none inline-flex h-5 select-none items-center gap-1 rounded border bg-muted px-1.5 font-mono text-[10px] font-medium opacity-100">
        <span class="text-xs">⌘</span>Enter
      </kbd>
    </div>
  </div>
</template>
