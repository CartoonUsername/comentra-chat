# Comentra Team Chat — Phase 1 & 2 Implementation

## Context

You are working on Comentra, a multi-tenant B2B SaaS marketplace management platform.

**Stack:** Next.js App Router, Supabase (JS SDK only, never direct pg), BullMQ + Redis, Docker Compose on Hostinger VPS.

**Rules:**
- All server-side Supabase calls use `createServiceClient()` with `requireTenant()`
- Always add explicit `.eq('tenant_id', tenantId)` alongside RLS
- Use `app.tenant_id` session variable for multi-tenancy
- No Netlify, no Vercel — manual deploy via `docker compose up --build -d`
- Design: background `#F7F5F0`, accent `#C9521A`, Syne headings (800), JetBrains Mono for numbers/IDs, lucide-react icons, no shadcn defaults

**Supabase project:** `ojxnvvgukhnzbkctypwb`
**Test tenant:** EmsCraft24 (`6163af6f-dadf-446b-af24-c0f4498351e5`)
**R2 public URL:** `https://pub-5afd0726b6344560903091f23ad3707d.r2.dev`

The existing Team Chat has Supabase Realtime, reply threading, and conversation list already partially built.

---

## Goal

Implement Phase 1 (UX polish) and Phase 2 (Ops-Context) for the Team Chat module.

---

## Phase 1 — Fundament

### 1. Konversationsliste

Upgrade the sidebar conversation list:

- Show avatar (initials circle, colored by user)
- Preview: first 40 chars of last message
- Timestamp: "14:02" if today, "Mo" if this week, "12.04" otherwise
- Unread badge: red dot if `unread_count > 0`
- Search input filters by conversation name or participant name live (client-side)
- Section headers: "Direkt" and "Gruppen" separating DM vs group conversations

DB query for sidebar (server component):

```typescript
const { data: conversations } = await supabase
  .from('conversations')
  .select(`
    id, name, type, updated_at,
    conversation_participants!inner(user_id, unread_count),
    messages(content, created_at, sender_id)
  `)
  .eq('tenant_id', tenantId)
  .eq('conversation_participants.user_id', currentUserId)
  .order('updated_at', { ascending: false })
  .limit(1, { foreignTable: 'messages' })
```

### 2. Gruppen-Chats

Add group conversation support:

Migration:
```sql
ALTER TABLE conversations ADD COLUMN IF NOT EXISTS type text DEFAULT 'direct' CHECK (type IN ('direct', 'group'));
ALTER TABLE conversations ADD COLUMN IF NOT EXISTS name text;
ALTER TABLE conversation_participants ADD COLUMN IF NOT EXISTS role text DEFAULT 'member' CHECK (role IN ('admin', 'member'));
```

API route `POST /api/chat/conversations`:
- Accept `{ type: 'group', name: string, participant_ids: string[] }`
- Insert into `conversations` then bulk insert `conversation_participants`
- Return new conversation

UI: "Neue Gruppe" button in sidebar header opens a modal with name input + member picker (search existing team members).

### 3. Typing Indicator

Use Supabase Realtime Presence:

```typescript
// In chat input component
const channel = supabase.channel(`chat:${conversationId}`)

// On input focus/change
channel.track({ typing: true, user_id: currentUserId, name: currentUserName })

// On blur or send
channel.track({ typing: false })

// Subscribe to others typing
channel.on('presence', { event: 'sync' }, () => {
  const state = channel.presenceState()
  const typing = Object.values(state)
    .flat()
    .filter((p: any) => p.typing && p.user_id !== currentUserId)
  setTypingUsers(typing.map((p: any) => p.name))
})

channel.subscribe()
```

Show below composer: `"Laura tippt..."` — disappears after 3s of no update.

### 4. Reactions

Migration:
```sql
CREATE TABLE IF NOT EXISTS message_reactions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  message_id uuid NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  user_id uuid NOT NULL REFERENCES users(id),
  emoji text NOT NULL,
  created_at timestamptz DEFAULT now(),
  UNIQUE(message_id, user_id, emoji)
);

ALTER TABLE message_reactions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "tenant_isolation" ON message_reactions
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

API `POST /api/chat/messages/[messageId]/reactions`:
- Body: `{ emoji: string }`
- Upsert/delete toggle: if reaction exists delete it, else insert

UI: On hover show reaction picker with 6 quick emojis: 👍 ✅ 👀 ❤️ 😂 🔥
Show reaction counts grouped by emoji below bubble.

### 5. File Attachments

Extend composer with file upload:

- Accept: images (jpg, png, webp), PDF, max 10MB
- Upload to R2 under path `chat/{tenantId}/{conversationId}/{timestamp}_{filename}`
- SHA256 dedup already in place — reuse existing R2 upload utility
- Store in `messages` with `type: 'file'` and `metadata: { url, filename, size, mime_type }`

Render in bubble:
- Images: inline preview max 300px wide, click to open full
- PDF: file card with icon, filename, size, download link

---

## Phase 2 — Ops-Kontext

### 6. Entity Linking (Bestellung + Produkt)

Migration:
```sql
ALTER TABLE conversations 
  ADD COLUMN IF NOT EXISTS linked_entity_type text CHECK (linked_entity_type IN ('order', 'product', 'return')),
  ADD COLUMN IF NOT EXISTS linked_entity_id uuid;

CREATE INDEX IF NOT EXISTS idx_conversations_entity 
  ON conversations(tenant_id, linked_entity_type, linked_entity_id);
```

API `PATCH /api/chat/conversations/[id]/link`:
- Body: `{ entity_type: 'order' | 'product', entity_id: string }`
- Updates `linked_entity_type` and `linked_entity_id`

In chat header, if conversation has linked entity:
- Fetch entity name + status server-side
- Render chip: for orders `Bestellung #1043291 · Shipped`, for products `ASIN B06WGLKFJF · 12 auf Lager`
- Chip is clickable — scrolls to Kontext-Sidebar

### 7. Kontext-Sidebar

Right panel that slides in when entity is linked. Toggle with header icon button.

For linked order — fetch and display:
```typescript
const { data: order } = await supabase
  .from('orders')
  .select(`
    id, order_number, status, created_at, total_amount, currency,
    shipping_address, carrier, tracking_number,
    order_items(product_name, quantity, price)
  `)
  .eq('id', linkedEntityId)
  .eq('tenant_id', tenantId)
  .single()
```

Show: order number, status badge, items list, shipping address, tracking link, timeline of status changes.

For linked product — fetch and display:
- Product name, ASIN/SKU, current stock, price, active channels, last 7 days sales.

### 8. In-Chat-Aktionen

Show action buttons in composer toolbar when entity is linked:

For orders:
- "Rücksendung anlegen" → calls existing returns API
- "Label drucken" → calls existing GLS/DHL label API  
- "Status setzen" → dropdown with order status options

For products:
- "Preis anpassen" → inline input with confirm
- "Lager aktualisieren" → inline number input

Each action posts to the existing API routes and sends a system message into the conversation confirming the action:

```typescript
await supabase.from('messages').insert({
  tenant_id: tenantId,
  conversation_id: conversationId,
  sender_id: null, // system message
  type: 'system_event',
  content: 'Rücksendung #RT-00123 wurde angelegt',
  metadata: { action: 'return_created', entity_id: returnId }
})
```

### 9. Event Notifications (System Messages)

BullMQ worker `chat-events-worker.ts`:

Subscribe to order status change events (already fired by order sync workers). When an order changes status and has a linked conversation:

```typescript
// In existing order status change handler, add:
const { data: conv } = await supabase
  .from('conversations')
  .select('id')
  .eq('tenant_id', tenantId)
  .eq('linked_entity_type', 'order')
  .eq('linked_entity_id', orderId)
  .single()

if (conv) {
  await supabase.from('messages').insert({
    tenant_id: tenantId,
    conversation_id: conv.id,
    sender_id: null,
    type: 'system_event',
    content: `Bestellung auf "${newStatus}" gewechselt`,
    metadata: { event: 'order_status_changed', old_status: oldStatus, new_status: newStatus }
  })
}
```

System messages render as a centered pill in the chat: gray background, smaller font, no avatar — clearly distinct from user messages.

### 10. @Mentions

In composer, trigger mention picker when `@` is typed:

- Show dropdown of team members filtered by what follows `@`
- On select: insert `@[Name](userId)` token into message
- Store mentions in `messages.metadata.mentions: string[]` (array of user_ids)

On message insert, for each mentioned user_id:
- Insert notification into `notifications` table (if exists) OR
- Fire Discord DM via existing Discord worker with message preview and link

Render in bubble: `@Max` highlighted in accent color `#C9521A`.

---

## File Structure

Create or modify these files:

```
src/
  app/
    (dashboard)/
      chat/
        page.tsx                    — main layout, sidebar + main
        [conversationId]/
          page.tsx                  — conversation view
  components/
    chat/
      ConversationList.tsx          — sidebar list with search
      ConversationItem.tsx          — single row with avatar/preview/badge
      MessageBubble.tsx             — message with reactions, file preview
      TypingIndicator.tsx           — "X tippt..." component
      ReactionPicker.tsx            — emoji hover picker
      ComposerToolbar.tsx           — file, mention, entity-link buttons
      ContextSidebar.tsx            — right panel with order/product details
      EntityChip.tsx                — header chip for linked entity
      SystemMessage.tsx             — centered event pill
      MentionPicker.tsx             — @mention dropdown
  app/
    api/
      chat/
        conversations/
          route.ts                  — GET list, POST create
          [id]/
            route.ts                — GET, PATCH
            link/
              route.ts              — PATCH entity link
        messages/
          route.ts                  — GET, POST
          [messageId]/
            reactions/
              route.ts              — POST toggle reaction
  workers/
    chat-events-worker.ts           — BullMQ: order events → system messages
  lib/
    supabase/
      chat.ts                       — shared chat query helpers
```

---

## Migrations to run

Collect all migrations above into one file:

```sql
-- /supabase/migrations/20260418_000000_chat_phase1_phase2.sql

ALTER TABLE conversations 
  ADD COLUMN IF NOT EXISTS type text DEFAULT 'direct' CHECK (type IN ('direct', 'group')),
  ADD COLUMN IF NOT EXISTS name text,
  ADD COLUMN IF NOT EXISTS linked_entity_type text CHECK (linked_entity_type IN ('order', 'product', 'return')),
  ADD COLUMN IF NOT EXISTS linked_entity_id uuid;

ALTER TABLE conversation_participants 
  ADD COLUMN IF NOT EXISTS role text DEFAULT 'member' CHECK (role IN ('admin', 'member'));

CREATE TABLE IF NOT EXISTS message_reactions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  message_id uuid NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  user_id uuid NOT NULL REFERENCES users(id),
  emoji text NOT NULL,
  created_at timestamptz DEFAULT now(),
  UNIQUE(message_id, user_id, emoji)
);

ALTER TABLE message_reactions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "tenant_isolation" ON message_reactions
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE INDEX IF NOT EXISTS idx_conversations_entity 
  ON conversations(tenant_id, linked_entity_type, linked_entity_id);

CREATE INDEX IF NOT EXISTS idx_message_reactions_message
  ON message_reactions(message_id);
```

Run via Supabase dashboard SQL editor or migration tool.

---

## Implementation order

1. Migration file — apply first
2. API routes (conversations CRUD, reactions, link)
3. ConversationList + ConversationItem (sidebar)
4. MessageBubble with reactions
5. TypingIndicator via Realtime Presence
6. File attachments (composer + bubble)
7. EntityChip + ContextSidebar
8. ComposerToolbar with In-Chat-Aktionen
9. System messages + chat-events-worker
10. @Mentions

Start with step 1 and proceed in order. Ask if anything is unclear before starting.
