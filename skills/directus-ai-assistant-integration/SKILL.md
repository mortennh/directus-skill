---
name: Directus AI Assistant Integration
description: Use this skill when integrating AI assistants into Directus applications. Build intelligent chat interfaces, content generation systems, context-aware suggestions, and copilot features using OpenAI, Anthropic Claude, and other AI providers. Implement real-time communication, vector search, RAG (Retrieval Augmented Generation), and natural language interfaces.
---

# Directus AI Assistant Integration

## Overview

This skill provides comprehensive guidance for integrating AI assistants into Directus applications. Build intelligent chat interfaces, content generation systems, context-aware suggestions, and copilot features using OpenAI, Anthropic Claude, and other AI providers. Implement real-time communication, vector search, RAG (Retrieval Augmented Generation), and natural language interfaces.

## When to Use This Skill

* Building AI chat interfaces in Directus panels
* Implementing content generation workflows
* Creating smart autocomplete and suggestions
* Adding natural language query interfaces
* Building AI-powered content moderation
* Implementing semantic search with embeddings
* Creating AI copilot features for users
* Setting up RAG systems with vector databases
* Building conversational interfaces
* Implementing AI-driven automation

## Architecture Overview

### AI Integration Stack

```
┌─────────────────────────────────────┐
│         Directus Frontend           │
│    (Vue 3 Chat Components)          │
└────────────┬────────────────────────┘
             │ WebSocket / REST
┌────────────▼────────────────────────┐
│       Directus Backend              │
│   (AI Service Layer)                │
├─────────────────────────────────────┤
│  • Request Queue                    │
│  • Context Management               │
│  • Token Optimization               │
│  • Response Streaming               │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│        AI Providers                 │
├─────────────────────────────────────┤
│  • OpenAI (GPT-4, Embeddings)      │
│  • Anthropic (Claude)               │
│  • Cohere (Reranking)               │
│  • Hugging Face (Open Models)      │
└─────────────────────────────────────┘
             │
┌────────────▼────────────────────────┐
│      Vector Database                │
│   (Pinecone/Weaviate/pgvector)     │
└─────────────────────────────────────┘
```

## Process: Building AI Chat Interface

### Step 1: Create Chat Panel Extension

```
<!-- src/ai-chat-panel.vue -->
<template>
  <div class="ai-chat-panel">
    <div class="chat-header">
      <div class="chat-title">
        <v-icon name="smart_toy" />
        <span>AI Assistant</span>
      </div>
      <div class="chat-actions">
        <v-button v-tooltip="'Clear conversation'" icon x-small @click="clearChat">
          <v-icon name="clear_all" />
        </v-button>
        <v-button v-tooltip="'Export conversation'" icon x-small @click="exportChat">
          <v-icon name="download" />
        </v-button>
      </div>
    </div>

    <div class="chat-messages" ref="messagesContainer">
      <transition-group name="message-fade">
        <div v-for="message in messages" :key="message.id" class="message" :class="message.role">
          <div class="message-avatar">
            <v-icon :name="message.role === 'user' ? 'person' : 'smart_toy'" />
          </div>
          <div class="message-content">
            <div class="message-text" v-html="renderMarkdown(message.content)"></div>
            <div class="message-metadata">
              <span class="message-time">{{ formatTime(message.timestamp) }}</span>
              <span v-if="message.tokens" class="message-tokens">{{ message.tokens }} tokens</span>
            </div>
            <div v-if="message.suggestions" class="message-suggestions">
              <v-chip v-for="suggestion in message.suggestions" :key="suggestion" clickable @click="sendMessage(suggestion)">
                {{ suggestion }}
              </v-chip>
            </div>
          </div>
        </div>
      </transition-group>

      <div v-if="isTyping" class="typing-indicator">
        <span></span><span></span><span></span>
      </div>

      <div v-if="streamingResponse" class="streaming-message">
        <div class="message-content">
          <div class="message-text" v-html="renderMarkdown(streamingResponse)"></div>
        </div>
      </div>
    </div>

    <div class="chat-input">
      <div class="input-container">
        <v-textarea
          v-model="inputMessage"
          placeholder="Type your message..."
          :disabled="isProcessing"
          @keydown.enter.prevent="handleEnter"
          auto-grow
          :rows="1"
          :max-rows="4"
        />
        <div class="input-actions">
          <v-button v-tooltip="'Voice input'" icon x-small @click="startVoiceInput" :disabled="!speechRecognitionSupported">
            <v-icon :name="isRecording ? 'mic' : 'mic_none'" />
          </v-button>
        </div>
      </div>
      <v-button @click="sendMessage()" :loading="isProcessing" :disabled="!inputMessage.trim()" icon>
        <v-icon name="send" />
      </v-button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, nextTick, watch } from 'vue';
import { useApi, useStores } from '@directus/extensions-sdk';
import { marked } from 'marked';
import DOMPurify from 'dompurify';

interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: Date;
  tokens?: number;
  suggestions?: string[];
}

interface Props {
  collection?: string;
  systemPrompt?: string;
  model?: string;
  maxTokens?: number;
  temperature?: number;
}

const props = withDefaults(defineProps<Props>(), {
  model: 'gpt-4-turbo-preview',
  maxTokens: 2000,
  temperature: 0.7,
});

const api = useApi();
const messages = ref<Message[]>([]);
const inputMessage = ref('');
const isProcessing = ref(false);
const isTyping = ref(false);
const streamingResponse = ref('');
const messagesContainer = ref<HTMLElement>();
const isRecording = ref(false);

const speechRecognitionSupported = computed(() => 'webkitSpeechRecognition' in window || 'SpeechRecognition' in window);

async function sendMessage(content?: string) {
  const messageContent = content || inputMessage.value.trim();
  if (!messageContent || isProcessing.value) return;

  isProcessing.value = true;
  inputMessage.value = '';

  messages.value.push({
    id: generateId(),
    role: 'user',
    content: messageContent,
    timestamp: new Date(),
  });

  scrollToBottom();

  try {
    const response = await api.post('/ai/chat', {
      message: messageContent,
      history: messages.value.slice(-10),
      config: { model: props.model, maxTokens: props.maxTokens, temperature: props.temperature },
    });

    messages.value.push({
      id: generateId(),
      role: 'assistant',
      content: response.data.content,
      timestamp: new Date(),
      tokens: response.data.tokens,
      suggestions: response.data.suggestions,
    });
  } catch (error) {
    messages.value.push({
      id: generateId(),
      role: 'system',
      content: 'Error: Failed to get response',
      timestamp: new Date(),
    });
  } finally {
    isProcessing.value = false;
    scrollToBottom();
  }
}

function renderMarkdown(content: string): string {
  return DOMPurify.sanitize(marked(content, { breaks: true, gfm: true }) as string);
}

function formatTime(timestamp: Date): string {
  return new Intl.DateTimeFormat('en-US', { hour: 'numeric', minute: 'numeric', hour12: true }).format(timestamp);
}

function handleEnter(event: KeyboardEvent) {
  if (!event.shiftKey) sendMessage();
}

function scrollToBottom() {
  nextTick(() => {
    if (messagesContainer.value) messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight;
  });
}

function generateId(): string {
  return `msg-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}

function clearChat() {
  messages.value = [];
  streamingResponse.value = '';
}

async function exportChat() {
  const conversation = messages.value.map(msg => ({ role: msg.role, content: msg.content, timestamp: msg.timestamp.toISOString() }));
  const blob = new Blob([JSON.stringify(conversation, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `chat-export-${Date.now()}.json`;
  a.click();
  URL.revokeObjectURL(url);
}

function startVoiceInput() {
  if (!speechRecognitionSupported.value) return;
  const SpeechRecognition = (window as any).webkitSpeechRecognition || (window as any).SpeechRecognition;
  const recognition = new SpeechRecognition();
  recognition.continuous = false;
  recognition.interimResults = true;
  recognition.onstart = () => { isRecording.value = true; };
  recognition.onresult = (event: any) => {
    inputMessage.value = Array.from(event.results).map((r: any) => r[0].transcript).join('');
  };
  recognition.onend = () => { isRecording.value = false; };
  recognition.start();
}
</script>

<style scoped>
.ai-chat-panel {
  height: 100%;
  display: flex;
  flex-direction: column;
  background: var(--theme--background);
  border-radius: var(--theme--border-radius);
  border: 1px solid var(--theme--border-color-subdued);
}

.chat-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--spacing-m);
  border-bottom: 1px solid var(--theme--border-color-subdued);
  background: var(--theme--background-accent);
}

.chat-title {
  display: flex;
  align-items: center;
  gap: var(--spacing-s);
  font-weight: 600;
}

.chat-messages {
  flex: 1;
  overflow-y: auto;
  padding: var(--spacing-m);
  display: flex;
  flex-direction: column;
  gap: var(--spacing-m);
}

.message {
  display: flex;
  gap: var(--spacing-m);
  animation: messageSlide 0.3s ease-out;
}

@keyframes messageSlide {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.message.user { flex-direction: row-reverse; }

.message-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--theme--primary-background);
  color: var(--theme--primary);
  flex-shrink: 0;
}

.message.user .message-avatar {
  background: var(--theme--background-accent);
  color: var(--theme--foreground);
}

.message-content {
  max-width: 70%;
  display: flex;
  flex-direction: column;
  gap: var(--spacing-xs);
}

.message.user .message-content { align-items: flex-end; }

.message-text {
  padding: var(--spacing-m);
  background: var(--theme--background-accent);
  border-radius: var(--theme--border-radius);
  color: var(--theme--foreground);
  line-height: 1.5;
}

.message.user .message-text {
  background: var(--theme--primary);
  color: white;
}

.message-text :deep(pre) {
  background: var(--theme--background);
  padding: var(--spacing-s);
  border-radius: var(--theme--border-radius);
  overflow-x: auto;
}

.message-metadata {
  display: flex;
  gap: var(--spacing-m);
  font-size: 0.75rem;
  color: var(--theme--foreground-subdued);
}

.typing-indicator {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: var(--spacing-m);
  background: var(--theme--background-accent);
  border-radius: var(--theme--border-radius);
  width: fit-content;
}

.typing-indicator span {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--theme--foreground-subdued);
  animation: typing 1.4s infinite;
}

.typing-indicator span:nth-child(2) { animation-delay: 0.2s; }
.typing-indicator span:nth-child(3) { animation-delay: 0.4s; }

@keyframes typing {
  0%, 60%, 100% { transform: translateY(0); opacity: 0.5; }
  30% { transform: translateY(-10px); opacity: 1; }
}

.chat-input {
  display: flex;
  gap: var(--spacing-s);
  padding: var(--spacing-m);
  border-top: 1px solid var(--theme--border-color-subdued);
  background: var(--theme--background-accent);
}

.input-container {
  flex: 1;
  display: flex;
  align-items: flex-end;
  gap: var(--spacing-xs);
  background: var(--theme--background);
  border-radius: var(--theme--border-radius);
  padding: var(--spacing-s);
}
</style>
```

## Process: Implementing AI Service Layer

### Step 1: Create AI Service

```
// src/services/ai.service.ts
import OpenAI from 'openai';
import Anthropic from '@anthropic-ai/sdk';
import { Pinecone } from '@pinecone-database/pinecone';

export class AIService {
  private openai: OpenAI | null = null;
  private anthropic: Anthropic | null = null;
  private pinecone: Pinecone | null = null;

  constructor(private options: any) {
    if (process.env.OPENAI_API_KEY) {
      this.openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
    }
    if (process.env.ANTHROPIC_API_KEY) {
      this.anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
    }
    if (process.env.PINECONE_API_KEY) {
      this.pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY });
    }
  }

  async chat(options: {
    messages: any[];
    model?: string;
    temperature?: number;
    maxTokens?: number;
    stream?: boolean;
    systemPrompt?: string;
  }): Promise<any> {
    const model = options.model || 'gpt-4-turbo-preview';
    const messages = options.systemPrompt
      ? [{ role: 'system', content: options.systemPrompt }, ...options.messages]
      : options.messages;

    if (model.startsWith('claude')) {
      return this.anthropicChat(messages, model, options.temperature ?? 0.7, options.maxTokens || 2000, options.stream);
    }
    return this.openaiChat(messages, model, options.temperature ?? 0.7, options.maxTokens || 2000, options.stream);
  }

  private async openaiChat(messages: any[], model: string, temperature: number, maxTokens: number, stream?: boolean): Promise<any> {
    if (!this.openai) throw new Error('OpenAI not configured');
    const completion = await this.openai.chat.completions.create({
      model, messages, temperature, max_tokens: maxTokens, stream: stream || false,
    });
    if (stream) return completion;
    const c = completion as any;
    return { content: c.choices[0].message.content, usage: c.usage, model: c.model };
  }

  private async anthropicChat(messages: any[], model: string, temperature: number, maxTokens: number, stream?: boolean): Promise<any> {
    if (!this.anthropic) throw new Error('Anthropic not configured');
    const systemPrompt = messages.find(m => m.role === 'system')?.content || '';
    const conversationMessages = messages.filter(m => m.role !== 'system');
    const response = await this.anthropic.messages.create({
      model: model.replace('claude-', ''),
      max_tokens: maxTokens,
      temperature,
      system: systemPrompt,
      messages: conversationMessages,
      stream,
    });
    if (stream) return response;
    const r = response as any;
    return { content: r.content[0].text, usage: { prompt_tokens: r.usage.input_tokens, completion_tokens: r.usage.output_tokens }, model: r.model };
  }

  async generateContent(options: {
    type: 'article' | 'description' | 'summary' | 'translation';
    input: string;
    targetLanguage?: string;
    tone?: 'formal' | 'casual' | 'technical' | 'creative';
    length?: 'short' | 'medium' | 'long';
  }): Promise<string> {
    const prompts = {
      article: `Write a comprehensive article about: ${options.input}\nTone: ${options.tone || 'formal'}\nLength: ${options.length || 'medium'}`,
      description: `Write a compelling product/service description for: ${options.input}`,
      summary: `Summarize the following content concisely: ${options.input}\nLength: ${options.length || 'short'}`,
      translation: `Translate the following to ${options.targetLanguage}: ${options.input}`,
    };
    const response = await this.chat({
      messages: [{ role: 'user', content: prompts[options.type] }],
      temperature: options.type === 'translation' ? 0.3 : 0.7,
    });
    return response.content;
  }

  async createEmbedding(options: { text: string; model?: string }): Promise<number[]> {
    if (!this.openai) throw new Error('OpenAI not configured');
    const response = await this.openai.embeddings.create({
      model: options.model || 'text-embedding-3-small',
      input: options.text,
    });
    return response.data[0].embedding;
  }

  async vectorSearch(options: { query: string; collection: string; topK?: number; filter?: any }): Promise<any[]> {
    if (!this.pinecone) throw new Error('Pinecone not configured');
    const queryEmbedding = await this.createEmbedding({ text: options.query });
    const index = this.pinecone.index(options.collection);
    const results = await index.query({
      vector: queryEmbedding,
      topK: options.topK || 10,
      filter: options.filter,
      includeMetadata: true,
    });
    return results.matches || [];
  }

  async ragQuery(options: { query: string; collection: string; systemContext?: string; topK?: number }): Promise<any> {
    const relevantDocs = await this.vectorSearch({ query: options.query, collection: options.collection, topK: options.topK || 5 });
    const context = relevantDocs.map(doc => doc.metadata?.content || '').join('\n\n---\n\n');
    const systemPrompt = `${options.systemContext || 'You are a helpful assistant.'}\n\nContext:\n${context}`;
    const response = await this.chat({
      messages: [{ role: 'user', content: options.query }],
      systemPrompt,
      temperature: 0.3,
    });
    return {
      answer: response.content,
      sources: relevantDocs.map(doc => ({ id: doc.id, score: doc.score, metadata: doc.metadata })),
      usage: response.usage,
    };
  }

  async moderateContent(content: string): Promise<{ safe: boolean; categories: any; flaggedTerms: string[] }> {
    if (!this.openai) throw new Error('OpenAI not configured');
    const moderation = await this.openai.moderations.create({ input: content });
    const result = moderation.results[0];
    const flaggedCategories = Object.entries(result.categories).filter(([_, flagged]) => flagged).map(([category]) => category);
    return { safe: !result.flagged, categories: result.category_scores, flaggedTerms: flaggedCategories };
  }

  async generateSuggestions(options: { context: string; type: 'autocomplete' | 'next_actions' | 'related_content'; count?: number }): Promise<string[]> {
    const prompts = {
      autocomplete: `Suggest ${options.count || 5} possible completions for: ${options.context}\nReturn as JSON array of strings`,
      next_actions: `Suggest ${options.count || 5} logical next actions for: ${options.context}\nReturn as JSON array`,
      related_content: `Suggest ${options.count || 5} related topics for: ${options.context}\nReturn as JSON array`,
    };
    const response = await this.chat({
      messages: [{ role: 'user', content: prompts[options.type] }],
      temperature: 0.8,
    });
    try { return JSON.parse(response.content); } catch { return []; }
  }
}
```

## AI-Powered Flows

### Custom AI Operations for Flows

```
// src/operations/ai-operation.ts
import { defineOperationApi } from '@directus/extensions-sdk';

export default defineOperationApi({
  id: 'ai-content-processor',
  handler: async (options, context) => {
    const { services } = context;
    const { ItemsService } = services;

    const aiService = new (require('../services/ai.service').AIService)({ knex: context.database });
    const itemsService = new ItemsService(options.collection, { schema: await context.getSchema() });

    const results = [];
    const items = await itemsService.readByQuery({ filter: options.filter || {}, limit: options.batchSize || 10 });

    for (const item of items) {
      try {
        let processedContent;
        switch (options.operation) {
          case 'summarize':
            processedContent = await aiService.generateContent({ type: 'summary', input: item[options.sourceField], length: options.summaryLength || 'short' });
            break;
          case 'translate':
            processedContent = await aiService.generateContent({ type: 'translation', input: item[options.sourceField], targetLanguage: options.targetLanguage });
            break;
          case 'moderate':
            const moderation = await aiService.moderateContent(item[options.sourceField]);
            processedContent = moderation.safe ? 'approved' : 'flagged';
            break;
        }
        await itemsService.updateOne(item.id, { [options.targetField]: processedContent, ai_processed_at: new Date() });
        results.push({ id: item.id, status: 'success', processed: processedContent });
      } catch (error) {
        results.push({ id: item.id, status: 'error', error: error.message });
      }
    }

    return { processed: results.length, results };
  },
});
```

## AI Content Moderation Hook

```
// src/hooks/ai-moderation.ts
import { defineHook } from '@directus/extensions-sdk';

export default defineHook(({ filter }, context) => {
  const { services, logger } = context;

  filter('items.create', async (payload, meta) => {
    const moderatedCollections = ['comments', 'posts', 'reviews'];

    if (moderatedCollections.includes(meta.collection)) {
      const { AIService } = services;
      const aiService = new AIService({ knex: context.database });

      const contentToModerate = Object.values(payload)
        .filter(value => typeof value === 'string')
        .join(' ');

      const moderation = await aiService.moderateContent(contentToModerate);

      if (!moderation.safe) {
        payload.status = 'pending_review';
        payload.moderation_flags = moderation.flaggedTerms;
        logger.warn('Content flagged for moderation:', { collection: meta.collection, flags: moderation.flaggedTerms });
      } else {
        payload.status = 'approved';
      }
    }

    return payload;
  });
});
```

## Success Metrics

* ✅ Chat interface responds in < 500ms (first token)
* ✅ AI responses are contextually relevant 95%+ of the time
* ✅ Content moderation catches inappropriate content with 98%+ accuracy
* ✅ Vector search returns relevant results in < 200ms
* ✅ RAG system provides accurate answers with source citations
* ✅ Natural language queries convert correctly 90%+ of the time
* ✅ WebSocket connections remain stable for extended sessions
* ✅ Token usage is optimized with proper truncation
* ✅ Embeddings are cached effectively reducing API calls by 70%
* ✅ Error handling prevents AI failures from breaking workflows

## Resources

* [OpenAI API Documentation](https://platform.openai.com/docs)
* [Anthropic Claude API](https://docs.anthropic.com)
* [Pinecone Vector Database](https://docs.pinecone.io)
* [Socket.io Documentation](https://socket.io/docs)
* [Directus WebSockets](https://docs.directus.io/guides/real-time/websockets)
* [Vue 3 Composition API](https://vuejs.org/guide/extras/composition-api-faq)
