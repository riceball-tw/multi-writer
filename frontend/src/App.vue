<script setup lang="ts">
import { ref, onMounted, computed, onUnmounted } from 'vue';
import { Conversation, ConversationContent, ConversationEmptyState, ConversationScrollButton } from '@/components/ai-elements/conversation';
import { Message, MessageContent, MessageResponse, MessageAvatar } from '@/components/ai-elements/message';
import { PromptInput, PromptInputBody, PromptInputTextarea, PromptInputFooter, PromptInputTools, PromptInputButton, PromptInputSubmit } from '@/components/ai-elements/prompt-input';
import { ModelSelector, ModelSelectorTrigger, ModelSelectorContent, ModelSelectorInput, ModelSelectorList, ModelSelectorEmpty, ModelSelectorGroup, ModelSelectorItem, ModelSelectorName, ModelSelectorLogo } from '@/components/ai-elements/model-selector';
import { Loader } from '@/components/ai-elements/loader';
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
  loading?: boolean;
  error?: string;
}

const prompt = ref('');
const availableModels = ref<Model[]>([]);
const selectedModels = ref<string[]>([]);
const messages = ref<ChatMessage[]>([]);
const isModelSelectorOpen = ref(false);
const isLoading = ref(false);

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

const sendPrompt = async () => {
  if (!prompt.value.trim() || selectedModels.value.length === 0) return;

  const userMessage: ChatMessage = {
    id: Date.now().toString(),
    role: 'user',
    content: prompt.value,
  };
  messages.value.push(userMessage);

  const currentPrompt = prompt.value;
  prompt.value = '';
  isLoading.value = true;

  // Create assistant messages for each selected model
  const assistantMessages: ChatMessage[] = selectedModels.value.map((model, idx) => ({
    id: `${Date.now()}-${idx}`,
    role: 'assistant' as const,
    content: '',
    model,
    loading: true,
  }));

  messages.value.push(...assistantMessages);

  // Send requests to all selected models
  assistantMessages.forEach(async (msg) => {
    const msgIndex = messages.value.findIndex(m => m.id === msg.id);
    try {
      const res = await fetch(`${apiUrl}/chat/completions`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apiKey}`,
        },
        body: JSON.stringify({
          model: msg.model,
          messages: [{ role: 'user', content: currentPrompt }],
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
              if (delta && msgIndex >= 0 && msgIndex < messages.value.length) {
                const msg = messages.value[msgIndex];
                if (msg) {
                  msg.content += delta;
                  msg.loading = false;
                }
              }
            } catch (e) {
              // Ignore parse errors for incomplete JSON chunks
            }
          }
        }
      }
    } catch (err) {
      if (msgIndex >= 0 && msgIndex < messages.value.length) {
        const msg = messages.value[msgIndex];
        if (msg) {
          msg.error = (err as Error).message;
          msg.loading = false;
        }
      }
    } finally {
      if (msgIndex >= 0 && msgIndex < messages.value.length) {
        const msg = messages.value[msgIndex];
        if (msg) {
          msg.loading = false;
        }
      }
      isLoading.value = messages.value.some(m => m.loading);
    }
  });
};

const groupedMessages = computed(() => {
  const grouped: { user: ChatMessage | null; assistants: ChatMessage[] }[] = [];
  let currentGroup: { user: ChatMessage | null; assistants: ChatMessage[] } | null = null;

  messages.value.forEach((msg) => {
    if (msg.role === 'user') {
      if (currentGroup) {
        grouped.push(currentGroup);
      }
      currentGroup = { user: msg, assistants: [] };
    } else if (msg.role === 'assistant' && currentGroup) {
      currentGroup.assistants.push(msg);
    }
  });

  if (currentGroup) {
    grouped.push(currentGroup);
  }

  return grouped;
});

const getModelDisplayName = (modelId: string) => {
  return modelId.split('/').pop() || modelId;
};

// Extract provider from model ID (e.g., "openai/gpt-4" -> "openai")
function getProvider(modelId: string): string {
  const base = modelId.split('/')[0] || modelId;
  // Handle model names without provider prefix
  if (base.startsWith('gpt-')) return 'openai';
  if (base.startsWith('claude-')) return 'anthropic';
  return base.split(':')[0]; // Remove version suffix like ":20b"
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

function handleSubmit(payload: { text: string }) {
  // payload.text contains the form value, but we use prompt ref directly
  sendPrompt();
}

function handlePanelFocus(panelRef: any, totalPanels: number, allRefs: any[]) {
  
  allRefs.forEach(ref => {
    if (ref === panelRef) {
      ref?.resize(100);
    } else {
      ref?.collapse();
    }
  });
}

function resetPanels(allRefs: any[], totalPanels: number) {
  const equalSize = 100 / totalPanels;
  allRefs.forEach(ref => ref?.resize(equalSize));
}

onMounted(() => {
  fetchModels();
  document.addEventListener('mouseup', handleSelection);
});

onUnmounted(() => {
  document.removeEventListener('mouseup', handleSelection);
});
</script>

<template>
  <div class="flex flex-col bg-background pb-40">
    <!-- Conversation Area -->
    <div class="flex-1 overflow-hidden">
      <Conversation class="h-full">
        <ConversationContent>
          <template v-if="groupedMessages.length === 0">
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
          </template>

          <template v-else>
            <div
              v-for="(group, groupIndex) in groupedMessages"
              :key="groupIndex"
              class="space-y-4"
            >
              <!-- User Message -->
              <Message v-if="group.user" from="user">
                <MessageContent>
                  <MessageResponse :content="group.user.content" :markdown="isMarkdownMode" />
                </MessageContent>
              </Message>

              <!-- Assistant Messages with Resizable Panels -->
              <div
                v-if="group.assistants && group.assistants.length > 0"
                class="w-full"
              >
                <!-- Reset Button -->
                <div class="mb-2 flex justify-end">
                  <button
                    @click="() => {
                      const allRefs = group.assistants.map((_, i) => (group as any)[`panelRef${i}`]);
                      resetPanels(allRefs, group.assistants.length);
                    }"
                    class="flex items-center gap-2 rounded-md px-3 py-1.5 text-sm hover:bg-accent"
                  >
                    <LayoutGridIcon class="size-4" />
                    Reset Panels
                  </button>
                </div>

                <ResizablePanelGroup
                  direction="horizontal"
                  class="min-h-[400px]"
                >
                  <template
                    v-for="(assistant, index) in group.assistants"
                    :key="assistant.id"
                  >
                    <ResizablePanel
                      :ref="(el) => { if (el) (group as any)[`panelRef${index}`] = el }"
                      :default-size="100 / group.assistants.length"
                      :min-size="10"
                      :collapsed-size="10"
                      collapsible
                      class="flex flex-col overflow-hidden"
                      v-slot="{ isCollapsed }"
                    >
                      <div 
                        class="group relative flex h-full flex-col border border-border bg-card transition-all hover:border-ring"
                      >
                        <!-- Vertical Label for Minimized Panel -->
                        <div 
                          v-if="isCollapsed" 
                          class="h-full flex justify-center items-start py-8 cursor-pointer"
                          @click="() => {
                            const allRefs = group.assistants.map((_, i) => (group as any)[`panelRef${i}`]);
                            handlePanelFocus((group as any)[`panelRef${index}`], group.assistants.length, allRefs);
                          }"
                        >
                          <p class="text-sm font-semibold whitespace-nowrap" style="writing-mode: vertical-rl;">
                            {{ getModelDisplayName(assistant.model || '') }}
                          </p>
                        </div>

                        <!-- Normal View -->
                        <template v-else>
                          <!-- Header -->
                          <div class="flex items-center gap-3 border-b border-border bg-card px-4 py-3">
                            <MessageAvatar
                              :src="`https://models.dev/logos/${getProvider(assistant.model || '')}.svg`"
                              :name="getModelDisplayName(assistant.model || '')"
                            />
                            <div class="flex-1 min-w-0">
                              <p class="text-sm font-semibold truncate">
                                {{ getModelDisplayName(assistant.model || '') }}
                              </p>
                            </div>
                            <button
                              @click="() => {
                                const allRefs = group.assistants.map((_, i) => (group as any)[`panelRef${i}`]);
                                handlePanelFocus((group as any)[`panelRef${index}`], group.assistants.length, allRefs);
                              }"
                              class="rounded-md p-1.5 hover:bg-accent"
                            >
                              <MaximizeIcon class="size-4" />
                            </button>
                          </div>

                          <!-- Content -->
                          <div class="flex-1 overflow-hidden px-4 py-4">
                          <Message from="assistant">
                            <MessageContent class="h-full w-full max-w-none">
                              <div v-if="assistant.loading" class="space-y-3 py-4">
                                <Shimmer>
                                  <span>Generating response...</span>
                                </Shimmer>
                              </div>
                              <div v-else-if="assistant.error" class="rounded-md bg-destructive/10 p-4 text-sm text-destructive">
                                <p class="font-medium mb-1">Error</p>
                                <p>{{ assistant.error }}</p>
                              </div>
                              <div v-else class="h-full overflow-y-auto text-sm leading-relaxed pr-2 [&>*:first-child]:mt-0 [&>*:last-child]:mb-0">
                                <MessageResponse :content="assistant.content" :markdown="isMarkdownMode" />
                              </div>
                            </MessageContent>
                          </Message>
                        </div>
                        </template>
                      </div>
                    </ResizablePanel>
                    <ResizableHandle
                      v-if="index < group.assistants.length - 1"
                      with-handle
                    />
                  </template>
                </ResizablePanelGroup>
              </div>
            </div>
          </template>
        </ConversationContent>
        <ConversationScrollButton />
      </Conversation>
    </div>

    <!-- Input Area -->
    <div class="p-4 fixed z-10 bottom-0 w-full">
      <div class="mx-auto max-w-4xl bg-background">
        <PromptInput
          class="w-full"
          @submit="handleSubmit"
        >
          <PromptInputBody>
            <PromptInputTextarea
              v-model="prompt"
              placeholder="Ask a question to compare AI models..."
              :disabled="isLoading || selectedModels.length === 0"
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
                @click="isHighlightMode = !isHighlightMode"
                :class="{ 'bg-primary text-primary-foreground hover:bg-primary/30': isHighlightMode }"
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
              :disabled="!prompt.trim() || selectedModels.length === 0 || isLoading"
              :status="isLoading ? 'submitted' : 'idle'"
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
