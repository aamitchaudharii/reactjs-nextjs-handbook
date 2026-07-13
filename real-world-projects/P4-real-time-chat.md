# P4 · Real-World Project: Real-Time Chat Application

> **Real-time chat is the canonical WebSocket project — but the interesting engineering challenges aren't in the WebSocket connection itself (that's a few lines of code), they're in everything around it: optimistic message delivery with rollback on failure, presence detection across multiple server instances, message history loading with cursor-based pagination, read receipts synchronized across devices, and the fundamental tension between Next.js's serverless-first architecture and WebSocket's requirement for persistent long-lived connections. This project covers each of these challenges with production-appropriate solutions.**

---

## Project Overview

**What you'll build:**

- Direct messages and group channels
- Real-time message delivery (WebSocket)
- Typing indicators and online presence
- Read receipts
- File attachments (images, documents)
- Message search (full-text)
- Push notifications (Web Push API)
- Message reactions (emoji)

**Technology choices:**

- Next.js 15 (App Router — UI and REST endpoints)
- Separate Node.js WebSocket server (Socket.io or native ws)
- Redis (pub/sub for multi-instance presence; message queuing)
- Prisma + PostgreSQL (message persistence, channel membership)
- AWS S3 / Cloudflare R2 (file attachments)
- web-push (Web Push Notifications)

---

## Architecture Decision Record

### ADR-1: Why a Separate WebSocket Server

```
THE FUNDAMENTAL CONFLICT:
  Next.js on Vercel runs as serverless functions — each request spins up
  a function, handles the request, and terminates. Serverless functions
  have execution time limits (10s on Vercel Hobby, 5 minutes on Pro).
  WebSocket connections must persist for the ENTIRE chat session (minutes
  to hours). These requirements are fundamentally incompatible.

  Even on self-hosted Next.js (next start), handling thousands of concurrent
  WebSocket connections in the Next.js server process creates memory and
  resource pressure that interferes with page rendering.

THE SOLUTION: separate the concerns
  Next.js server: handles HTTP routes, page rendering, REST API, auth
  WebSocket server: handles real-time connections only (separate process/service)

ARCHITECTURE:
  Browser ←→ Next.js (HTTP)     : page loads, auth, REST API for history
  Browser ←→ WS Server (WSS)   : real-time message delivery, presence

  The WS server is a standalone Node.js process:
  - Express or bare http.createServer + ws library
  - Socket.io for room management and reconnection handling
  - Connects to the same PostgreSQL database and Redis as Next.js

WHY NOT JUST USE NEXT.JS API ROUTES + SSE FOR REAL-TIME:
  SSE is server → client only. Chat requires bidirectional: the client
  sends messages TO the server AND receives messages FROM the server.
  WebSocket is the right transport for bidirectional real-time communication.
  (SSE would work for notifications, not for chat message delivery.)
```

```
DEPLOYMENT:
  Next.js: Vercel / Railway / Fly.io (any serverless-compatible host)
  WS Server: Railway / Fly.io / EC2 / any persistent Node.js host
  Redis: Upstash (serverless-compatible) or Railway Redis
  PostgreSQL: Neon / Supabase / Railway PostgreSQL
```

---

### ADR-2: Message Delivery Architecture

```
THE OPTIMISTIC MESSAGE FLOW:

1. User types message and hits Enter
2. UI immediately shows message with "sending" status (optimistic — no round trip wait)
3. Client sends message to WS server
4. WS server:
   a. Validates user is member of the channel
   b. Saves message to PostgreSQL
   c. Publishes to Redis pub/sub channel
   d. Acknowledges to sender ("message saved, id=msg-123")
5. All OTHER subscribers in the channel receive the message from Redis
6. Sender receives acknowledgment → updates message status to "sent"

FAILURE HANDLING:
  If step 4 fails (DB error, auth failure):
  WS server sends error acknowledgment to sender
  Client removes the optimistic message and shows error toast:
  "Message failed to send. Tap to retry."

WHY REDIS PUB/SUB:
  If 1000 users are in a channel, they might be connected to DIFFERENT
  WebSocket server instances (for horizontal scaling).
  Instance A handles the sender.
  Instance A publishes to Redis channel "channel:general".
  Instance B (handling 500 other users) subscribes to that Redis channel
  and broadcasts to its connected users.
  Without Redis: messages only reach users on the SAME instance.
```

```ts
// ws-server/index.ts — the standalone WebSocket server
import { createServer } from "http";
import { Server } from "socket.io";
import { createClient } from "redis";
import { db } from "./lib/db";
import { verifySessionToken } from "./lib/auth";

const httpServer = createServer();
const io = new Server(httpServer, {
  cors: { origin: process.env.NEXT_PUBLIC_APP_URL, credentials: true },
});

// Redis clients: separate pub and sub (required by Redis pub/sub protocol)
const redisPublisher = createClient({ url: process.env.REDIS_URL });
const redisSubscriber = createClient({ url: process.env.REDIS_URL });

await redisPublisher.connect();
await redisSubscriber.connect();

// Subscribe to all channel messages from Redis:
await redisSubscriber.pSubscribe("channel:*", (message, pattern) => {
  const channelId = pattern.replace("channel:", "");
  const data = JSON.parse(message);
  // Broadcast to all Socket.io clients in this room:
  io.to(`channel:${channelId}`).emit("message:new", data);
});

io.use(async (socket, next) => {
  // Authenticate via session token (passed in auth handshake):
  const token = socket.handshake.auth.token;
  const session = token ? await verifySessionToken(token) : null;
  if (!session) return next(new Error("Unauthorized"));
  socket.data.userId = session.userId;
  next();
});

io.on("connection", (socket) => {
  const { userId } = socket.data;

  // Join channels the user is a member of:
  socket.on("join:channels", async (channelIds: string[]) => {
    // Verify membership for each channel:
    const memberships = await db.channelMember.findMany({
      where: { userId, channelId: { in: channelIds } },
    });
    const authorizedChannelIds = memberships.map((m) => m.channelId);
    authorizedChannelIds.forEach((id) => socket.join(`channel:${id}`));
    socket.emit("join:channels:ack", { joined: authorizedChannelIds });
  });

  // Handle incoming messages:
  socket.on("message:send", async (payload, ack) => {
    try {
      const { channelId, content, tempId } = payload;

      // Verify the user is a member of this channel:
      const isMember = await db.channelMember.findFirst({
        where: { channelId, userId },
      });
      if (!isMember) return ack({ error: "UNAUTHORIZED" });

      // Persist to database:
      const message = await db.message.create({
        data: { channelId, authorId: userId, content },
        include: {
          author: { select: { id: true, name: true, avatarUrl: true } },
        },
      });

      // Publish to Redis (reaches all instances):
      await redisPublisher.publish(
        `channel:${channelId}`,
        JSON.stringify({ message, channelId }),
      );

      // Acknowledge to sender with the real message ID:
      ack({ success: true, messageId: message.id, tempId });
    } catch (error) {
      ack({ error: "SEND_FAILED" });
    }
  });

  // Typing indicators:
  socket.on("typing:start", ({ channelId }) => {
    socket.to(`channel:${channelId}`).emit("typing:update", {
      userId,
      channelId,
      isTyping: true,
    });
  });

  socket.on("typing:stop", ({ channelId }) => {
    socket.to(`channel:${channelId}`).emit("typing:update", {
      userId,
      channelId,
      isTyping: false,
    });
  });

  // Presence: mark user as offline on disconnect
  socket.on("disconnect", async () => {
    await redisPublisher.hSet("presence", userId, "offline");
    // Notify all channels this user was in:
    io.emit("presence:update", { userId, status: "offline" });
  });
});

httpServer.listen(process.env.WS_PORT ?? 3001);
```

---

### ADR-3: Client-Side WebSocket Connection Management

```tsx
// hooks/useSocket.ts
"use client";
import { useEffect, useRef, useState } from "react";
import { io, Socket } from "socket.io-client";
import { useSession } from "next-auth/react";

export function useSocket() {
  const { data: session } = useSession();
  const socketRef = useRef<Socket | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    if (!session?.sessionToken) return;

    const socket = io(process.env.NEXT_PUBLIC_WS_URL!, {
      auth: { token: session.sessionToken }, // passed to server auth middleware
      reconnectionAttempts: 5,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
    });

    socket.on("connect", () => {
      setIsConnected(true);
      // Re-join channels after reconnection:
      socket.emit("join:channels", subscribedChannelIds);
    });

    socket.on("disconnect", (reason) => {
      setIsConnected(false);
      // If the server disconnected us (auth failure, server restart):
      if (reason === "io server disconnect") {
        socket.connect(); // manual reconnect
      }
      // For transport errors: Socket.io auto-reconnects
    });

    socketRef.current = socket;

    return () => {
      socket.disconnect();
      socketRef.current = null;
    };
  }, [session?.sessionToken]);

  return { socket: socketRef.current, isConnected };
}
```

---

### ADR-4: Message History Loading

```tsx
// Message history uses cursor-based pagination:
// Latest messages load first; scrolling up loads older messages

// hooks/useMessages.ts
import { useInfiniteQuery } from "@tanstack/react-query";

function useMessages(channelId: string) {
  return useInfiniteQuery({
    queryKey: ["messages", channelId],
    queryFn: ({ pageParam: cursor }) =>
      fetch(
        `/api/channels/${channelId}/messages?cursor=${cursor ?? ""}&limit=50`,
      ).then((r) => r.json()),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    // Start from the most recent messages:
    initialPageParam: undefined,
    // Combine pages for display (oldest first in the UI):
    select: (data) => ({
      pages: [...data.pages].reverse(),
      pageParams: data.pageParams,
    }),
  });
}

// The API endpoint:
// app/api/channels/[channelId]/messages/route.ts
export async function GET(
  request: Request,
  { params }: { params: { channelId: string } },
) {
  const session = await getSession();
  if (!session) return new Response("Unauthorized", { status: 401 });

  const url = new URL(request.url);
  const cursor = url.searchParams.get("cursor");
  const limit = parseInt(url.searchParams.get("limit") ?? "50");

  // Verify membership:
  const isMember = await db.channelMember.findFirst({
    where: { channelId: params.channelId, userId: session.userId },
  });
  if (!isMember) return new Response("Forbidden", { status: 403 });

  const messages = await db.message.findMany({
    where: {
      channelId: params.channelId,
      ...(cursor ? { id: { lt: cursor } } : {}), // messages older than cursor
    },
    orderBy: { createdAt: "desc" }, // newest first from DB
    take: limit,
    include: {
      author: { select: { id: true, name: true, avatarUrl: true } },
      reactions: { include: { user: { select: { id: true, name: true } } } },
    },
  });

  const nextCursor =
    messages.length === limit ? messages[messages.length - 1].id : null;

  return Response.json({
    messages: messages.reverse(), // flip to chronological for client
    nextCursor,
  });
}
```

---

### ADR-5: Optimistic Message UI

```tsx
// features/chat/components/MessageComposer.tsx
"use client";
import { useSocket } from "@/hooks/useSocket";
import { useQueryClient } from "@tanstack/react-query";
import { nanoid } from "nanoid";

function MessageComposer({ channelId }: { channelId: string }) {
  const { socket } = useSocket();
  const queryClient = useQueryClient();
  const [draft, setDraft] = useState("");

  const sendMessage = () => {
    if (!draft.trim() || !socket) return;

    const tempId = nanoid(); // temporary client-side ID
    const optimisticMessage: OptimisticMessage = {
      id: tempId,
      tempId,
      content: draft,
      authorId: currentUserId,
      author: currentUser,
      channelId,
      createdAt: new Date().toISOString(),
      status: "sending", // ← "sending" shows a spinner/faded state
    };

    // Immediately add to the message list (optimistic):
    queryClient.setQueryData(
      ["messages", channelId],
      addOptimisticMessage(optimisticMessage),
    );

    setDraft("");

    // Emit to WebSocket server:
    socket.emit(
      "message:send",
      { channelId, content: draft, tempId },
      (ack) => {
        if (ack.success) {
          // Replace temp message with real message (update id, status):
          queryClient.setQueryData(
            ["messages", channelId],
            confirmOptimisticMessage(tempId, ack.messageId),
          );
        } else {
          // Rollback: mark message as failed:
          queryClient.setQueryData(
            ["messages", channelId],
            markMessageFailed(tempId),
          );
        }
      },
    );
  };

  return (
    <div className="composer">
      <textarea
        value={draft}
        onChange={(e) => {
          setDraft(e.target.value);
          // Typing indicator (debounced):
          debouncedTypingIndicator(channelId, socket);
        }}
        onKeyDown={(e) => {
          if (e.key === "Enter" && !e.shiftKey) {
            e.preventDefault();
            sendMessage();
          }
        }}
        placeholder="Type a message..."
      />
      <button onClick={sendMessage} disabled={!draft.trim()}>
        Send
      </button>
    </div>
  );
}
```

---

## File Attachments

```ts
// Secure file upload flow using pre-signed URLs:
// 1. Client requests a pre-signed URL from our server
// 2. Our server validates the file type and generates the URL (never expires tokens to client)
// 3. Client uploads directly to S3/R2 (our server never handles the bytes)
// 4. Client sends the S3 URL to the message

// app/api/uploads/presign/route.ts
export async function POST(request: Request) {
  const session = await getSession();
  if (!session) return new Response("Unauthorized", { status: 401 });

  const { fileName, contentType, fileSize } = await request.json();

  // Validate:
  const ALLOWED_TYPES = [
    "image/jpeg",
    "image/png",
    "image/gif",
    "application/pdf",
  ];
  const MAX_SIZE = 10 * 1024 * 1024; // 10MB

  if (!ALLOWED_TYPES.includes(contentType)) {
    return Response.json({ error: "File type not allowed" }, { status: 400 });
  }
  if (fileSize > MAX_SIZE) {
    return Response.json(
      { error: "File too large (max 10MB)" },
      { status: 400 },
    );
  }

  const key = `uploads/${session.userId}/${Date.now()}-${fileName}`;
  const signedUrl = await getSignedUploadUrl({ key, contentType }); // S3/R2 SDK

  return Response.json({ uploadUrl: signedUrl, key });
}
```

---

## Testing Strategy

```
THE HARDEST PART TO TEST: WebSocket behavior

UNIT TESTS:
  - Message formatting utilities
  - Cursor pagination helpers
  - Optimistic update reducers

INTEGRATION TESTS (mock Socket.io):
  vi.mock('socket.io-client', () => ({
    io: () => ({
      on: vi.fn(),
      emit: vi.fn((event, data, callback) => {
        if (event === 'message:send') callback({ success: true, messageId: 'msg-123' });
      }),
      disconnect: vi.fn(),
    }),
  }));
  - Test that sending a message adds an optimistic entry
  - Test that a successful ack updates the message status
  - Test that a failed ack shows error state

E2E TESTS (Playwright with two browser contexts):
  test('two users can exchange messages in real time', async ({ browser }) => {
    const contextA = await browser.newContext();
    const contextB = await browser.newContext();
    const pageA = await contextA.newPage();
    const pageB = await contextB.newPage();

    await loginAs(pageA, userA);
    await loginAs(pageB, userB);

    await pageA.goto('/channels/general');
    await pageB.goto('/channels/general');

    await pageA.getByRole('textbox').fill('Hello from A');
    await pageA.keyboard.press('Enter');

    // pageB should see the message without refreshing:
    await expect(pageB.getByText('Hello from A')).toBeVisible({ timeout: 3000 });
  });
```

---

## Key Learning Outcomes

After building this project, you should be able to articulate:

1. **The serverless/WebSocket incompatibility** — why long-lived connections require a persistent Node.js process and can't be handled by serverless functions, and how to architect the separation

2. **Redis pub/sub for horizontal scaling** — how a single logical "channel" spans multiple WebSocket server instances via Redis, allowing horizontal scaling without message loss

3. **Optimistic UI with acknowledgment** — the three-state message lifecycle (sending → sent → failed) and how Socket.io's callback acknowledgment pattern implements reliable delivery confirmation

4. **Cursor-based pagination for chat history** — why chat history uses cursor pagination (not page numbers), how to combine it with infinite scroll, and how new WebSocket messages integrate with the paginated history

5. **Pre-signed URL file uploads** — why the server generates pre-signed S3 URLs rather than proxying file bytes through the application, and the security validation that must happen before generating the URL

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
