# P6 · Real-World Project: AI-Powered Application

> **Building AI-powered features with Next.js introduces an entirely new set of engineering challenges: streaming token-by-token responses to the browser without blocking, persisting conversation history while keeping token counts manageable, rate-limiting per user to control costs, handling hallucinations gracefully in the UI, and integrating RAG (Retrieval-Augmented Generation) to ground responses in your own data. This project covers the architecture of a production AI coding assistant — representative of the AI integration patterns used in most commercial AI applications.**

---

## Project Overview

**What you'll build:**

- AI coding assistant with streaming responses
- Conversation history with multi-turn context
- Document upload for RAG (answer questions about uploaded code/docs)
- Per-user rate limiting and token budget tracking
- Prompt caching for common system prompts
- Structured output (AI-generated JSON parsed into typed React components)
- Tool use (AI can search, read files, run calculations)

**Technology choices:**

- Next.js 15 (App Router)
- Vercel AI SDK (streaming, tool use, structured output)
- Anthropic Claude API (primary model)
- OpenAI-compatible fallback
- Prisma + PostgreSQL (conversation persistence)
- Upstash Redis (rate limiting via sliding window)
- Pinecone or pgvector (vector store for RAG)

---

## Architecture Decision Record

### ADR-1: Streaming Architecture

```
THE CORE CHALLENGE:
  AI model responses take 5-30 seconds to fully generate.
  Without streaming: the UI shows a blank state for 15 seconds,
  then the complete response appears at once → terrible UX.
  With streaming: tokens appear as they're generated → conversational feel.

HOW STREAMING WORKS IN NEXT.JS:
  1. Client sends message via fetch POST to a Route Handler
  2. Route Handler calls the AI API with streaming enabled
  3. The AI API returns a ReadableStream of tokens
  4. The Route Handler pipes this stream directly back to the HTTP response
  5. The browser reads the streaming response token by token
  6. React renders each token as it arrives (no full re-renders — appends to text)

VERCEL AI SDK STREAMTEXT:
  The SDK handles the streaming plumbing:
  - Route Handler: streamText() → returns a streamable response
  - Client: useChat() hook → reads the stream, manages message state,
    provides input handlers

DATA PROTOCOL:
  The AI SDK uses a custom stream protocol (not plain text streaming):
  Each "part" of the response is prefixed with a type code:
  0: "text delta"    → append to current message
  2: "data"          → structured data (tool results, metadata)
  3: "error"         → error occurred
  f: "finish"        → generation complete + token counts
  This allows multiplexing text, tool calls, and metadata in one stream.
```

```ts
// app/api/chat/route.ts
import { streamText, convertToCoreMessages } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import { checkRateLimit } from "@/lib/rate-limit";
import { getConversationHistory, appendMessage } from "@/lib/conversation";

export const maxDuration = 60; // 60 second max streaming duration (Vercel Pro+)

export async function POST(request: Request) {
  const session = await getSession();
  if (!session) return new Response("Unauthorized", { status: 401 });

  // Rate limiting (before any expensive AI call):
  const { allowed, remaining, resetAt } = await checkRateLimit(session.userId);
  if (!allowed) {
    return Response.json(
      { error: "Rate limit exceeded", resetAt },
      {
        status: 429,
        headers: {
          "Retry-After": String(Math.ceil((resetAt - Date.now()) / 1000)),
        },
      },
    );
  }

  const { messages, conversationId } = await request.json();

  // Load conversation history from DB (for multi-turn context):
  const history = conversationId
    ? await getConversationHistory(conversationId, session.userId)
    : [];

  const result = streamText({
    model: anthropic("claude-sonnet-4-6"),
    system: SYSTEM_PROMPT,
    messages: [
      ...convertToCoreMessages(history),
      ...convertToCoreMessages(messages),
    ],
    maxTokens: 4096,
    temperature: 0.7,

    // Tool definitions (AI can call these):
    tools: {
      searchDocs: {
        description: "Search the documentation for relevant information",
        parameters: z.object({ query: z.string() }),
        execute: async ({ query }) => {
          const results = await vectorSearch(query, session.userId);
          return results.map((r) => ({ content: r.text, source: r.fileName }));
        },
      },
      runCodeExample: {
        description: "Validate a code snippet by checking its syntax",
        parameters: z.object({ code: z.string(), language: z.string() }),
        execute: async ({ code, language }) => {
          return validateSyntax(code, language);
        },
      },
    },

    // Persist assistant response when stream completes:
    onFinish: async ({ text, usage, finishReason }) => {
      await appendMessage({
        conversationId:
          conversationId ?? (await createConversation(session.userId)),
        role: "assistant",
        content: text,
        inputTokens: usage.promptTokens,
        outputTokens: usage.completionTokens,
        userId: session.userId,
      });

      // Track token usage for billing/budget enforcement:
      await trackTokenUsage(
        session.userId,
        usage.promptTokens + usage.completionTokens,
      );
    },
  });

  return result.toDataStreamResponse();
}

const SYSTEM_PROMPT = `You are an expert React and Next.js engineering assistant.
You help engineers understand complex concepts, debug issues, and make architecture decisions.
When providing code examples, always use TypeScript and Next.js App Router patterns.
Be concise but thorough. Acknowledge uncertainty when you're not sure.`;
```

---

### ADR-2: Client-Side Streaming with useChat

```tsx
// features/chat/components/ChatInterface.tsx
"use client";
import { useChat } from "@ai-sdk/react";

export function ChatInterface({ conversationId }: { conversationId?: string }) {
  const {
    messages,
    input,
    handleInputChange,
    handleSubmit,
    isLoading,
    error,
    stop, // cancel the in-progress stream
    reload, // retry the last message
  } = useChat({
    api: "/api/chat",
    body: { conversationId }, // sent with every request
    initialMessages: [], // loaded from conversation history separately
    onError: (error) => {
      console.error("Chat error:", error);
      toast.error("Failed to get a response. Please try again.");
    },
    onFinish: (message) => {
      // Scroll to bottom when a new message completes:
      bottomRef.current?.scrollIntoView({ behavior: "smooth" });
    },
  });

  return (
    <div className="chat-container">
      <div
        className="messages"
        role="log"
        aria-live="polite"
        aria-label="Conversation"
      >
        {messages.map((message) => (
          <MessageBubble key={message.id} message={message} />
        ))}
        {isLoading && <TypingIndicator />}
        {error && (
          <div role="alert" className="error-message">
            <p>Something went wrong.</p>
            <button onClick={reload}>Try again</button>
          </div>
        )}
        <div ref={bottomRef} />
      </div>

      <form onSubmit={handleSubmit} className="composer">
        <textarea
          value={input}
          onChange={handleInputChange}
          placeholder="Ask about React, Next.js, debugging..."
          onKeyDown={(e) => {
            if (e.key === "Enter" && !e.shiftKey) {
              e.preventDefault();
              handleSubmit(e as any);
            }
          }}
          disabled={isLoading}
        />
        <div className="composer-actions">
          {isLoading ? (
            <button type="button" onClick={stop}>
              Stop generating
            </button>
          ) : (
            <button type="submit" disabled={!input.trim()}>
              Send
            </button>
          )}
        </div>
      </form>
    </div>
  );
}

// MessageBubble shows streaming text with a cursor while generating:
function MessageBubble({ message }: { message: Message }) {
  return (
    <article
      className={`message message--${message.role}`}
      aria-label={`${message.role === "user" ? "You" : "Assistant"}: ${message.content}`}
    >
      {message.role === "assistant" ? (
        <MarkdownRenderer content={message.content} />
      ) : (
        <p>{message.content}</p>
      )}
      {/* Show tool invocations when AI uses tools: */}
      {message.toolInvocations?.map((tool) => (
        <ToolCallDisplay key={tool.toolCallId} invocation={tool} />
      ))}
    </article>
  );
}
```

---

### ADR-3: RAG (Retrieval-Augmented Generation)

```
THE PROBLEM RAG SOLVES:
  LLMs have a knowledge cutoff and don't know about YOUR codebase,
  YOUR documentation, or YOUR proprietary data. RAG grounds AI responses
  in real documents by:
  1. Splitting documents into chunks (~500 tokens each)
  2. Embedding each chunk as a high-dimensional vector (semantic representation)
  3. Storing vectors in a vector database
  4. At query time: embed the user's question, find the most semantically
     similar chunks, inject them into the AI's context as "here's relevant info"

IMPLEMENTATION:

INDEXING PIPELINE (runs when a user uploads a document):
  Upload → split into chunks → embed each chunk → store in vector DB

RETRIEVAL (happens on every AI query that might benefit from docs):
  User message → embed query → search vector DB → retrieve top-K chunks
  → inject into system prompt: "Here is relevant context: [chunks]"
  → AI generates response grounded in the chunks
```

```ts
// lib/rag/index.ts
import { embed, embedMany } from "ai";
import { openai } from "@ai-sdk/openai";
import { db } from "@/lib/db";

const EMBEDDING_MODEL = openai.embedding("text-embedding-3-small");
const CHUNK_SIZE = 500; // tokens per chunk
const CHUNK_OVERLAP = 50; // overlap between consecutive chunks

// Index a document:
export async function indexDocument(
  fileContent: string,
  fileName: string,
  userId: string,
) {
  // Split into overlapping chunks:
  const chunks = splitIntoChunks(fileContent, CHUNK_SIZE, CHUNK_OVERLAP);

  // Embed all chunks in parallel (batch API call):
  const { embeddings } = await embedMany({
    model: EMBEDDING_MODEL,
    values: chunks.map((c) => c.text),
  });

  // Store in PostgreSQL with pgvector:
  await db.$executeRaw`
    INSERT INTO document_chunks (user_id, file_name, content, embedding)
    VALUES ${chunks
      .map(
        (chunk, i) =>
          `(${userId}, ${fileName}, ${chunk.text}, ${JSON.stringify(embeddings[i])}::vector)`,
      )
      .join(", ")}
  `;
}

// Retrieve relevant context for a query:
export async function retrieveContext(
  query: string,
  userId: string,
): Promise<string> {
  const { embedding } = await embed({
    model: EMBEDDING_MODEL,
    value: query,
  });

  // Cosine similarity search (pgvector):
  const results = await db.$queryRaw<
    { content: string; file_name: string; similarity: number }[]
  >`
    SELECT content, file_name, 1 - (embedding <=> ${JSON.stringify(embedding)}::vector) AS similarity
    FROM document_chunks
    WHERE user_id = ${userId}
    ORDER BY embedding <=> ${JSON.stringify(embedding)}::vector
    LIMIT 5
  `;

  if (results.length === 0) return "";

  return results
    .filter((r) => r.similarity > 0.7) // threshold: only include relevant chunks
    .map((r) => `[From ${r.file_name}]\n${r.content}`)
    .join("\n\n---\n\n");
}

// Enhanced system prompt with RAG context:
async function buildSystemPrompt(
  query: string,
  userId: string,
): Promise<string> {
  const context = await retrieveContext(query, userId);

  if (!context) return BASE_SYSTEM_PROMPT;

  return `${BASE_SYSTEM_PROMPT}

RELEVANT CONTEXT FROM UPLOADED DOCUMENTS:
${context}

Use the above context to inform your response when relevant. 
Cite the source file when referencing specific information from the context.`;
}
```

---

### ADR-4: Rate Limiting and Token Budget

```ts
// lib/rate-limit.ts
import { Redis } from "@upstash/redis";
import { Ratelimit } from "@upstash/ratelimit";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Sliding window: 20 requests per hour per user
const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(20, "1 h"),
  analytics: true, // track usage in Upstash dashboard
});

export async function checkRateLimit(userId: string) {
  const { success, remaining, reset } = await ratelimit.limit(userId);
  return { allowed: success, remaining, resetAt: reset };
}

// Token budget: enforce a daily token limit per plan
const DAILY_TOKEN_LIMITS = {
  FREE: 50_000, // ~50 messages/day
  PRO: 500_000, // ~500 messages/day
  ENTERPRISE: Infinity,
};

export async function checkTokenBudget(
  userId: string,
  plan: string,
): Promise<boolean> {
  const key = `tokens:${userId}:${new Date().toISOString().split("T")[0]}`; // daily key
  const usedToday = parseInt((await redis.get<string>(key)) ?? "0");
  const limit =
    DAILY_TOKEN_LIMITS[plan as keyof typeof DAILY_TOKEN_LIMITS] ?? 50_000;
  return usedToday < limit;
}

export async function trackTokenUsage(userId: string, tokensUsed: number) {
  const key = `tokens:${userId}:${new Date().toISOString().split("T")[0]}`;
  await redis.incrby(key, tokensUsed);
  await redis.expire(key, 86400 * 2); // keep for 2 days (in case of timezone edge cases)
}
```

---

## Structured Output: AI-Generated Typed Data

```tsx
// Instead of asking the AI to explain something in prose,
// ask it to return structured JSON that drives rich UI components:

// app/api/analyze/route.ts
import { generateObject } from "ai";
import { z } from "zod";

const CodeAnalysisSchema = z.object({
  summary: z.string(),
  issues: z.array(
    z.object({
      severity: z.enum(["error", "warning", "info"]),
      line: z.number().optional(),
      message: z.string(),
      suggestion: z.string(),
    }),
  ),
  refactoredCode: z.string().optional(),
  performanceScore: z.number().min(0).max(10),
});

export async function POST(request: Request) {
  const { code, language } = await request.json();

  const { object } = await generateObject({
    model: anthropic("claude-sonnet-4-6"),
    schema: CodeAnalysisSchema,
    prompt: `Analyze this ${language} code for issues, bugs, and performance problems:

\`\`\`${language}
${code}
\`\`\`

Return a structured analysis.`,
  });

  // `object` is fully typed as CodeAnalysis:
  return Response.json(object);
}

// features/code-analyzer/components/AnalysisResult.tsx
("use client");
function AnalysisResult({ analysis }: { analysis: CodeAnalysis }) {
  return (
    <div>
      <p>{analysis.summary}</p>
      <PerformanceGauge score={analysis.performanceScore} />
      {analysis.issues.map((issue, i) => (
        <IssueCard key={i} issue={issue} />
      ))}
      {analysis.refactoredCode && (
        <CodeBlock code={analysis.refactoredCode} language="typescript" />
      )}
    </div>
  );
}
```

---

## Conversation History and Context Window Management

```ts
// Managing long conversations: older messages must be pruned
// because every AI model has a context window limit (e.g., 200K tokens for Claude)

export async function getConversationHistory(
  conversationId: string,
  userId: string,
): Promise<Message[]> {
  const messages = await db.message.findMany({
    where: { conversationId, conversation: { userId } },
    orderBy: { createdAt: "asc" },
    // Don't load ALL messages — limit to last N that fit in context:
  });

  // Estimate token count and prune from the beginning if too long:
  let totalTokens = 0;
  const MAX_HISTORY_TOKENS = 100_000; // leave room for the new message + response

  const selectedMessages: Message[] = [];
  for (const message of [...messages].reverse()) {
    const estimatedTokens = Math.ceil(message.content.length / 4); // rough estimate
    if (totalTokens + estimatedTokens > MAX_HISTORY_TOKENS) break;
    selectedMessages.unshift(message);
    totalTokens += estimatedTokens;
  }

  return selectedMessages;
}
```

---

## Testing Strategy for AI Applications

```
THE CHALLENGE: AI responses are non-deterministic. You can't assert
"the response equals X" because it won't be the same twice.

WHAT YOU CAN TEST DETERMINISTICALLY:
  - The request shape (correct model, correct system prompt structure)
  - Rate limiting enforcement (assert 429 on the 21st request)
  - Token budget tracking (assert the token count is updated in Redis)
  - RAG retrieval (assert the right chunks are retrieved for known queries)
  - Structured output schema compliance (assert the response matches Zod schema)
  - Conversation history truncation (assert history doesn't exceed token budget)

MOCKING THE AI API IN TESTS:
  vi.mock('@ai-sdk/anthropic', () => ({
    anthropic: () => ({
      // Return a predictable stream for testing:
    }),
  }));

  OR: use AI SDK's built-in test utilities:
  import { simulateReadableStream } from 'ai/test';
  const mockStream = simulateReadableStream({
    chunks: ['Hello', ' there', '!'],
  });

SNAPSHOT TESTING FOR PROMPTS:
  The system prompt and RAG-enhanced prompt should be snapshot tested —
  if someone changes the system prompt in a way that degrades AI behavior,
  the snapshot test catches the change before it ships.

  test('system prompt includes RAG context when documents are uploaded', async () => {
    const prompt = await buildSystemPrompt('How does auth work?', userId);
    expect(prompt).toMatchSnapshot();
    // Snapshot includes the RAG context format, grounding instructions, etc.
  });
```

---

## Security Considerations for AI Features

```
PROMPT INJECTION:
  Users can attempt to override your system prompt by including instructions
  in their messages: "Ignore all previous instructions and..."
  MITIGATIONS:
  - Put the system prompt in the `system` parameter (not in messages)
  - Use models that are trained to resist injection (Claude is particularly good)
  - Validate/sanitize user input before including in prompts
  - Never include sensitive server data in prompts that user input touches

DATA LEAKAGE THROUGH AI:
  If your RAG indexes sensitive documents, the AI might reveal them to
  unauthorized users via the AI's responses.
  MITIGATION: Always filter RAG results by userId — only index
  and retrieve documents belonging to the authenticated user.
  Never share a vector store across users.

COST ATTACKS:
  A malicious user could send thousands of API requests, running up your
  AI API bill. The rate limiter (20 req/hour, daily token budget) is critical.
  Also: validate input length before sending to AI (reject messages >10,000 chars)

CONTENT MODERATION:
  AI can be prompted to produce harmful content.
  Claude has built-in safety training, but for user-facing apps:
  - Use the AI provider's content filtering options
  - Log flagged responses for review
  - Implement your own filter layer for known harmful patterns
```

---

## Key Learning Outcomes

After building this project, you should be able to articulate:

1. **The AI streaming architecture** — how `streamText` in a Route Handler pipes the model's token stream directly to the HTTP response, and how `useChat` on the client reads and renders the stream incrementally

2. **RAG implementation** — the full pipeline from document chunking → embedding → vector storage → retrieval → context injection, and why this is more reliable than fine-tuning for most use cases

3. **Non-deterministic testing strategies** — what can be deterministically tested (rate limiting, token tracking, RAG retrieval, schema compliance) versus what can only be evaluated qualitatively

4. **Structured output with Zod** — how `generateObject` constrains AI output to a typed schema, enabling AI-generated data to drive rich UI components with full TypeScript safety

5. **Context window management** — why long conversation histories must be pruned before sending to the AI, and the token estimation and selection strategy for keeping history within budget

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
