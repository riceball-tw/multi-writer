<script setup lang="ts">
import { ref, onMounted, computed, onUnmounted, watch } from 'vue';
import { Conversation, ConversationContent, ConversationEmptyState, ConversationScrollButton } from '@/components/ai-elements/conversation';
import { Message, MessageContent, MessageResponse, MessageAvatar } from '@/components/ai-elements/message';
import { PromptInput, PromptInputBody, PromptInputTextarea, PromptInputFooter, PromptInputTools, PromptInputButton, PromptInputSubmit } from '@/components/ai-elements/prompt-input';
import { ModelSelector, ModelSelectorTrigger, ModelSelectorContent, ModelSelectorInput, ModelSelectorList, ModelSelectorEmpty, ModelSelectorGroup, ModelSelectorItem, ModelSelectorName, ModelSelectorLogo } from '@/components/ai-elements/model-selector';
import { Shimmer } from '@/components/ai-elements/shimmer';
import { ResizablePanelGroup, ResizablePanel, ResizableHandle } from '@/components/ui/resizable';
import { CheckIcon, SparklesIcon, MaximizeIcon, LayoutGridIcon, HighlighterIcon, StickyNoteIcon, FileTextIcon } from 'lucide-vue-next';
import { Sheet, SheetContent, SheetHeader, SheetTitle } from '@/components/ui/sheet';
import { VueMonacoEditor } from '@guolao/vue-monaco-editor';

interface Model {
  id: string;
}

interface ChatMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  model?: string;
  targetModels?: string[];
  loading?: boolean;
  error?: string;
}

const prompt = ref('');
const modelPrompts = ref<Record<string, string>>({});
const availableModels = ref<Model[]>([]);
const selectedModels = ref<string[]>([]);
const messages = ref<ChatMessage[]>([]);
const isModelSelectorOpen = ref(false);
const isModelLoading = ref<Record<string, boolean>>({});
const isGlobalLoading = computed(() => selectedModels.value.some(m => isModelLoading.value[m]));
const panelRefs = ref<any[]>([]);

const isHighlightMode = ref(false);
const isSheetOpen = ref(false);
const isMarkdownMode = ref(true);

const monacoContent = ref('');

const handleSelection = () => {
  if (!isHighlightMode.value) return;
  const text = window.getSelection()?.toString().trim();
  if (text) {
    if (monacoContent.value) {
      monacoContent.value += `\n\n${text}`;
    } else {
      monacoContent.value = text;
    }
    // We intentionally do not auto-open the sheet anymore
    window.getSelection()?.removeAllRanges();
  }
};

const apiUrl = import.meta.env.VITE_LITELLM_API_URL || 'http://localhost:4000/v1';
const apiKey = import.meta.env.VITE_LITELLM_API_KEY || 'sk-1234';

const fetchModels = async () => {
  try {
    const res = await fetch(`${apiUrl}/models`, {
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    const data = await res.json();
    availableModels.value = data.data || [];
    if (selectedModels.value.length === 0) {
      selectedModels.value = availableModels.value.map(m => m.id);
    }
  } catch (err) {
    console.error('Failed to fetch models:', err);
  }
};

const sendPrompt = async (targetModelId?: string | Event) => {
  const isEvent = targetModelId && typeof targetModelId === 'object';
  const resolvedTargetId = isEvent ? undefined : (targetModelId as string | undefined);
  const isGlobalFixed = !resolvedTargetId;
  
  const currentPrompt = isGlobalFixed ? prompt.value : (modelPrompts.value[resolvedTargetId] || '');
  if (!currentPrompt?.trim()) return;

  const activeModels = isGlobalFixed ? selectedModels.value : [resolvedTargetId];
  if (activeModels.length === 0) return;

  const userMessage: ChatMessage = {
    id: Date.now().toString(),
    role: 'user',
    content: currentPrompt,
    model: resolvedTargetId,
    targetModels: isGlobalFixed ? [...selectedModels.value] : undefined,
  };
  messages.value.push(userMessage);

  if (isGlobalFixed) prompt.value = '';
  else modelPrompts.value[resolvedTargetId] = '';

  activeModels.forEach(m => {
    if (m) isModelLoading.value[m] = true;
  });

  const assistantMessages: ChatMessage[] = activeModels.map((model, idx) => ({
    id: `${Date.now()}-${idx}`,
    role: 'assistant' as const,
    content: '',
    model,
    loading: true,
  }));

  messages.value.push(...assistantMessages);

  assistantMessages.forEach(async (msg) => {
    // Get History BEFORE we fetch! We exclude the newly created un-fetched assistant msg
    const history = messages.value.filter(m => {
      if (m.id === msg.id) return false;
      if (m.role === 'assistant') return m.model === msg.model;
      if (m.role === 'user') {
        if (m.model) return m.model === msg.model;
        if (m.targetModels) return m.targetModels.includes(msg.model!);
        return true;
      }
      return false;
    }).map(m => ({ role: m.role, content: m.content }));

    try {
      const res = await fetch(`${apiUrl}/chat/completions`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apiKey}`,
        },
        body: JSON.stringify({
          model: msg.model,
          messages: history,
          stream: true,
        }),
      });

      if (!res.ok) throw new Error(`Status: ${res.status}`);
      
      const reader = res.body?.getReader();
      if (!reader) throw new Error('No reader available');

      const decoder = new TextDecoder();
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value, { stream: true });
        const lines = chunk.split('\n');

        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = line.slice(6);
            if (data === '[DONE]') break;
            try {
              const parsed = JSON.parse(data);
              const delta = parsed.choices[0]?.delta?.content || '';
              if (delta) {
                const currentMsgIndex = messages.value.findIndex(m => m.id === msg.id);
                if (currentMsgIndex !== -1) {
                  const m = messages.value[currentMsgIndex];
                  if (m) {
                    m.content += delta;
                    m.loading = false;
                  }
                }
              }
            } catch (e) {
              // Ignore parse errors for incomplete JSON chunks
            }
          }
        }
      }
    } catch (err) {
      const currentMsgIndex = messages.value.findIndex(m => m.id === msg.id);
      if (currentMsgIndex !== -1) {
        const m = messages.value[currentMsgIndex];
        if (m) {
          m.error = (err as Error).message;
          m.loading = false;
        }
      }
    } finally {
      const currentMsgIndex = messages.value.findIndex(m => m.id === msg.id);
      if (currentMsgIndex !== -1) {
        const m = messages.value[currentMsgIndex];
        if (m) {
          m.loading = false;
        }
      }
      if (msg.model) {
        isModelLoading.value[msg.model] = messages.value.some(m => m.model === msg.model && m.loading);
      }
    }
  });
};

const getHistoryForModelDisplay = (modelId: string) => {
  return messages.value.filter(msg => {
    if (msg.role === 'assistant') {
      return msg.model === modelId;
    }
    if (msg.role === 'user') {
      if (msg.model) return msg.model === modelId;
      if (msg.targetModels) return msg.targetModels.includes(modelId);
      return true;
    }
    return false;
  });
};

const getModelDisplayName = (modelId: string) => {
  return modelId.split('/').pop() || modelId;
};

// Extract provider from model ID (e.g., "openai/gpt-4" -> "openai")
function getProvider(modelId: string): string {
  const base = modelId.split('/')[0] || modelId;
  // Handle model names without provider prefix
  if (base.startsWith('gpt-')) return 'openai';
  if (base.startsWith('claude-')) return 'anthropic';
  return base.split(':')[0] || base; // Remove version suffix like ":20b"
}

// Get display name from model ID (e.g., "openai/gpt-4" -> "gpt-4")
function getDisplayName(modelId: string): string {
  const parts = modelId.split('/');
  if (parts.length > 1) {
    const lastPart = parts[parts.length - 1];
    return lastPart || modelId;
  }
  return modelId;
}

// Get selected model data for display in trigger
const selectedModelData = computed(() => {
  if (selectedModels.value.length === 0) return null;
  const firstSelected = selectedModels.value[0];
  const model = availableModels.value.find(m => m.id === firstSelected);
  if (!model) return null;
  return {
    id: model.id,
    provider: getProvider(model.id),
    name: getDisplayName(model.id),
  };
});

// Group models by provider
const groupedModels = computed(() => {
  const groups: Record<string, Model[]> = {};
  availableModels.value.forEach(model => {
    const provider = getProvider(model.id);
    if (!groups[provider]) {
      groups[provider] = [];
    }
    groups[provider].push(model);
  });
  return groups;
});

function toggleModel(modelId: string) {
  const current = [...selectedModels.value];
  if (current.includes(modelId)) {
    selectedModels.value = current.filter(m => m !== modelId);
  } else {
    selectedModels.value = [...current, modelId];
  }
}

function handleSubmit(_payload?: { text: string }) {
  sendPrompt();
}

function toggleHighlightMode() {
  if (isSheetOpen.value) return;
  isHighlightMode.value = !isHighlightMode.value;
}

function handlePanelFocus(panelRef: any) {
  panelRefs.value.forEach(ref => {
    if (ref === panelRef) {
      ref?.resize(100);
    } else {
      ref?.collapse();
    }
  });
}

function resetPanels(totalPanels: number) {
  const equalSize = 100 / totalPanels;
  panelRefs.value.forEach(ref => ref?.resize(equalSize));
}

onMounted(() => {
  fetchModels();
  document.addEventListener('mouseup', handleSelection);
});

onUnmounted(() => {
  document.removeEventListener('mouseup', handleSelection);
});

watch(isSheetOpen, (open) => {
  if (open) isHighlightMode.value = false;
});
</script>

<template>
  <div class="flex flex-col bg-background h-dvh w-full overflow-hidden">
    <!-- Conversation Area -->
    <div class="flex-1 overflow-hidden">
      <Conversation v-if="messages.length === 0" class="h-full">
        <ConversationContent>
          <ConversationEmptyState>
            <template #icon>
              <SparklesIcon class="size-12 text-muted-foreground" />
            </template>
            <template #default>
              <h3 class="font-medium text-sm">Start a conversation</h3>
              <p class="text-muted-foreground text-sm">
                Select models and ask a question to compare their responses
              </p>
            </template>
          </ConversationEmptyState>
        </ConversationContent>
      </Conversation>

      <div v-else class="h-full flex flex-col pt-4">
        <!-- Reset Button -->
        <div class="mb-2 flex justify-end px-4">
          <button
            @click="resetPanels(selectedModels.length)"
            class="flex items-center gap-2 rounded-md px-3 py-1.5 text-sm hover:bg-accent"
          >
            <LayoutGridIcon class="size-4" />
            Reset Panels
          </button>
        </div>

        <ResizablePanelGroup
          direction="horizontal"
          class="flex-1 px-4 pb-4"
        >
          <template
            v-for="(modelId, index) in selectedModels"
            :key="modelId"
          >
            <ResizablePanel
              :ref="(el) => { if (el) panelRefs[index] = el }"
              :default-size="100 / selectedModels.length"
              :min-size="10"
              :collapsed-size="10"
              collapsible
              class="flex flex-col overflow-hidden px-2"
              v-slot="{ isCollapsed }"
            >
              <div 
                class="group relative flex h-full flex-col border border-border bg-card transition-all hover:border-ring rounded-lg overflow-hidden"
              >
                <!-- Vertical Label for Minimized Panel -->
                <div 
                  v-if="isCollapsed" 
                  class="h-full flex justify-center items-start py-8 cursor-pointer"
                  @click="handlePanelFocus(panelRefs[index])"
                >
                  <p class="text-sm font-semibold whitespace-nowrap" style="writing-mode: vertical-rl;">
                    {{ getModelDisplayName(modelId) }}
                  </p>
                </div>

                <!-- Normal View -->
                <template v-else>
                  <!-- Header -->
                  <div class="flex items-center gap-3 border-b border-border bg-card px-4 py-3 shrink-0">
                    <MessageAvatar
                      :src="`https://models.dev/logos/${getProvider(modelId)}.svg`"
                      :name="getModelDisplayName(modelId)"
                    />
                    <div class="flex-1 min-w-0">
                      <p class="text-sm font-semibold truncate">
                        {{ getModelDisplayName(modelId) }}
                      </p>
                    </div>
                    <button
                      @click="handlePanelFocus(panelRefs[index])"
                      class="rounded-md p-1.5 hover:bg-accent"
                    >
                      <MaximizeIcon class="size-4" />
                    </button>
                  </div>

                  <!-- Content -->
                  <Conversation class="flex-1 overflow-y-auto w-full bg-background/50">
                    <ConversationContent class="p-4 space-y-6 w-full h-full pr-4">
                      <template v-for="msg in getHistoryForModelDisplay(modelId)" :key="msg.id">
                        <!-- Apply max-w-none so it uses full width in its container -->
                        <Message :from="msg.role" class="w-full">
                          <MessageContent v-if="msg.role === 'assistant'" class="w-[95%] max-w-none">
                            <div v-if="msg.loading" class="space-y-3 py-4">
                              <Shimmer>
                                <span>Generating response...</span>
                              </Shimmer>
                            </div>
                            <div v-else-if="msg.error" class="rounded-md bg-destructive/10 p-4 text-sm text-destructive">
                              <p class="font-medium mb-1">Error</p>
                              <p>{{ msg.error }}</p>
                            </div>
                            <div v-else class="text-sm leading-relaxed [&>*:first-child]:mt-0 [&>*:last-child]:mb-0">
                              <MessageResponse :content="msg.content" :markdown="isMarkdownMode" />
                            </div>
                          </MessageContent>
                          <MessageContent v-else class="w-[95%] max-w-none ml-auto pt-2 bg-muted rounded-xl p-3">
                             <MessageResponse :content="msg.content" :markdown="isMarkdownMode" />
                          </MessageContent>
                        </Message>
                      </template>
                    </ConversationContent>
                    <ConversationScrollButton />
                  </Conversation>
                  
                  <!-- Assistant specific Prompt -->
                  <div class="p-3 border-t border-border bg-card shrink-0">
                    <PromptInput @submit="() => sendPrompt(modelId)" class="min-h-15 w-full">
                      <PromptInputBody>
                        <PromptInputTextarea
                          v-model="modelPrompts[modelId]"
                          :placeholder="`Reply to ${getDisplayName(modelId)}...`"
                          :disabled="isModelLoading[modelId]"
                          :rows="1"
                        />
                      </PromptInputBody>
                      <PromptInputFooter>
                        <PromptInputTools />
                        <PromptInputSubmit :disabled="!modelPrompts[modelId]?.trim() || isModelLoading[modelId]" :status="isModelLoading[modelId] ? 'submitted' : 'idle'" />
                      </PromptInputFooter>
                    </PromptInput>
                  </div>
                </template>
              </div>
            </ResizablePanel>
            <ResizableHandle
              v-if="index < selectedModels.length - 1"
              with-handle
            />
          </template>
        </ResizablePanelGroup>
      </div>
    </div>

    <!-- Global Input Area -->
    <div class="shrink-0 p-4 border-t bg-background">
      <div class="mx-auto max-w-4xl">
        <PromptInput class="w-full" @submit="handleSubmit">
          <PromptInputBody>
            <PromptInputTextarea
              v-model="prompt"
              placeholder="Ask a question to compare AI models..."
              :disabled="isGlobalLoading || selectedModels.length === 0"
            />
          </PromptInputBody>

          <PromptInputFooter>
            <PromptInputTools>
              <ModelSelector v-model:open="isModelSelectorOpen">
                <ModelSelectorTrigger as-child>
                  <PromptInputButton>
                    <ModelSelectorLogo
                      v-if="selectedModelData?.provider"
                      :provider="selectedModelData.provider"
                    />
                    <ModelSelectorName v-if="selectedModelData?.name">
                      {{ selectedModelData.name }}
                      <span v-if="selectedModels.length > 1" class="text-muted-foreground">
                        +{{ selectedModels.length - 1 }}
                      </span>
                    </ModelSelectorName>
                    <span v-else class="text-muted-foreground">Select model</span>
                  </PromptInputButton>
                </ModelSelectorTrigger>

                <ModelSelectorContent>
                  <ModelSelectorInput placeholder="Search models..." />
                  <ModelSelectorList>
                    <ModelSelectorEmpty>No models found.</ModelSelectorEmpty>

                    <ModelSelectorGroup
                      v-for="(models, provider) in groupedModels"
                      :key="provider"
                      :heading="provider.charAt(0).toUpperCase() + provider.slice(1)"
                    >
                      <ModelSelectorItem
                        v-for="model in models"
                        :key="model.id"
                        :value="model.id"
                        @select="() => toggleModel(model.id)"
                      >
                        <ModelSelectorLogo :provider="getProvider(model.id)" />
                        <ModelSelectorName>{{ getDisplayName(model.id) }}</ModelSelectorName>
                        <CheckIcon
                          v-if="selectedModels.includes(model.id)"
                          class="ml-auto size-4"
                        />
                        <div v-else class="ml-auto size-4" />
                      </ModelSelectorItem>
                    </ModelSelectorGroup>
                  </ModelSelectorList>
                </ModelSelectorContent>
              </ModelSelector>

              <PromptInputButton
                @click="toggleHighlightMode"
                :class="{ 'bg-primary text-primary-foreground hover:bg-primary/30': isHighlightMode }"
                :disabled="isSheetOpen"
                title="Highlight Mode"
              >
                <HighlighterIcon class="size-4" />
                <span class="ml-1.5">Highlight Mode</span>
              </PromptInputButton>

              <PromptInputButton
                @click="isMarkdownMode = !isMarkdownMode"
                :title="isMarkdownMode ? 'Switch to Plaintext' : 'Switch to Markdown'"
              >
                <FileTextIcon class="size-4" />
                <span class="ml-1.5">{{ isMarkdownMode ? 'Markdown' : 'Plaintext' }}</span>
              </PromptInputButton>
            </PromptInputTools>

            <PromptInputSubmit
              :disabled="!prompt.trim() || selectedModels.length === 0 || isGlobalLoading"
              :status="isGlobalLoading ? 'submitted' : 'idle'"
            />
          </PromptInputFooter>
        </PromptInput>
        <p v-if="selectedModels.length === 0" class="mt-2 text-center text-xs text-muted-foreground">
          Please select at least one model to start chatting
        </p>
      </div>
    </div>

    <!-- Sheet for Sidenotes -->
    <Sheet v-model:open="isSheetOpen">
      <SheetContent class="sm:max-w-xl w-[90vw] flex flex-col p-4">
        <SheetHeader class="mb-4 shrink-0">
          <SheetTitle>Sidenotes</SheetTitle>
        </SheetHeader>
        <div class="flex-1 min-h-0 relative border rounded-md overflow-hidden bg-background">
          <VueMonacoEditor
            v-model:value="monacoContent"
            theme="vs-dark"
            language="markdown"
            :options="{
              minimap: { enabled: false },
              wordWrap: 'on',
              lineNumbers: 'off',
              padding: { top: 16, bottom: 16 },
              fontSize: 14
            }"
          />
        </div>
      </SheetContent>
    </Sheet>

    <!-- Floating Sidenote Toggle Button -->
    <button
      @click="isSheetOpen = !isSheetOpen"
      class="fixed bottom-6 right-6 z-50 flex h-12 w-12 items-center justify-center rounded-full bg-primary text-primary-foreground shadow-lg transition-transform hover:scale-105 active:scale-95"
      title="Toggle Sidenotes"
    >
      <StickyNoteIcon class="size-5" />
    </button>
  </div>
</template>
