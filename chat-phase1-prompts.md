# Comentra Chat — Phase 1 Prompts für Claude Code

> Reihenfolge einhalten. Jeder Prompt ist atomar und referenziert den vorherigen.

---

## Prompt 1 — DB Migration

**Datei:** `supabase/migrations/20260417_000000_chat_phase1.sql`

```
Erstelle eine neue Supabase Migration: supabase/migrations/20260417_000000_chat_phase1.sql

Kontext: Comentra verwendet Next.js App Router, Supabase JS SDK, tenant_id-basierte Isolation.
Bestehende Tabelle: chat_messages (id, conversation_id, sender_id, content, created_at, tenant_id)
Bestehende Tabelle: chat_conversations (id, tenant_id, type, created_at)

Füge folgende Änderungen hinzu — alle mit IF NOT EXISTS / DO $$ EXCEPTION absichern:

1. chat_messages erweitern:
   - status TEXT NOT NULL DEFAULT 'sent' CHECK (status IN ('sent','delivered','read'))
   - reply_to_id UUID REFERENCES chat_messages(id) ON DELETE SET NULL
   - reactions JSONB NOT NULL DEFAULT '{}'
     Format: { "👍": ["user_id_1","user_id_2"], "❤️": ["user_id_3"] }
   - attachments JSONB NOT NULL DEFAULT '[]'
     Format: [{ url, name, size, mime_type, width?, height? }]
   - deleted_at TIMESTAMPTZ (soft-delete)
   - edited_at TIMESTAMPTZ

2. Neue Tabelle: message_read_receipts
   - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
   - message_id UUID REFERENCES chat_messages(id) ON DELETE CASCADE
   - user_id UUID NOT NULL
   - read_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
   - tenant_id UUID NOT NULL
   - UNIQUE(message_id, user_id)

3. Neue Tabelle: chat_conversation_members
   - conversation_id UUID REFERENCES chat_conversations(id) ON DELETE CASCADE
   - user_id UUID NOT NULL
   - tenant_id UUID NOT NULL
   - last_read_at TIMESTAMPTZ
   - is_muted BOOLEAN DEFAULT false
   - unread_count INT NOT NULL DEFAULT 0
   - PRIMARY KEY(conversation_id, user_id)

4. Neue Tabelle: user_presence
   - user_id UUID PRIMARY KEY
   - tenant_id UUID NOT NULL
   - status TEXT CHECK (status IN ('online','away','offline')) DEFAULT 'offline'
   - last_seen TIMESTAMPTZ DEFAULT NOW()

5. Indexes:
   - chat_messages(conversation_id, created_at DESC)
   - chat_messages(reply_to_id) WHERE reply_to_id IS NOT NULL
   - message_read_receipts(message_id)
   - user_presence(tenant_id)

6. RLS für alle neuen Tabellen:
   - Alle: tenant_id = auth.jwt()->>'tenant_id'
   - message_read_receipts INSERT: user_id = auth.uid()
   - user_presence UPDATE: user_id = auth.uid()

Kein TypeScript, keine API-Logik — nur reines SQL.
```

---

## Prompt 2 — TypeScript Types + Zod Schemas

**Datei (neu):** `types/chat.ts`

```
Erstelle types/chat.ts mit vollständigen TypeScript-Types und Zod-Schemas für das Chat-System.

Stack: TypeScript strict, Zod v3, keine externen Type-Pakete für Supabase-Tabellen nötig.

Exportiere folgende Interfaces/Types:

// Enums
export type MessageStatus = 'sent' | 'delivered' | 'read'
export type PresenceStatus = 'online' | 'away' | 'offline'

// Attachment
export interface MessageAttachment {
  url: string
  name: string
  size: number        // bytes
  mime_type: string
  width?: number
  height?: number
}

// Reactions map: emoji → user_id[]
export type ReactionsMap = Record<string, string[]>

// Message (DB row)
export interface ChatMessage {
  id: string
  conversation_id: string
  sender_id: string
  content: string
  status: MessageStatus
  reply_to_id: string | null
  reply_to?: ChatMessage | null   // joined
  reactions: ReactionsMap
  attachments: MessageAttachment[]
  deleted_at: string | null
  edited_at: string | null
  created_at: string
  tenant_id: string
  sender?: {                       // joined aus auth.users / profiles
    id: string
    full_name: string
    avatar_url?: string
  }
}

// Read Receipt
export interface MessageReadReceipt {
  message_id: string
  user_id: string
  read_at: string
}

// Conversation Member
export interface ConversationMember {
  conversation_id: string
  user_id: string
  last_read_at: string | null
  is_muted: boolean
  unread_count: number
}

// Presence
export interface UserPresence {
  user_id: string
  status: PresenceStatus
  last_seen: string
}

// Realtime Broadcast Payloads
export interface TypingPayload {
  user_id: string
  user_name: string
  conversation_id: string
  is_typing: boolean
}

// Zod schemas für API validation
import { z } from 'zod'

export const SendMessageSchema = z.object({
  conversation_id: z.string().uuid(),
  content: z.string().min(1).max(4000),
  reply_to_id: z.string().uuid().optional(),
  attachments: z.array(z.object({
    url: z.string().url(),
    name: z.string(),
    size: z.number().positive(),
    mime_type: z.string(),
    width: z.number().optional(),
    height: z.number().optional(),
  })).optional().default([]),
})

export const ToggleReactionSchema = z.object({
  message_id: z.string().uuid(),
  emoji: z.string().min(1).max(10),
})

export const MarkReadSchema = z.object({
  message_id: z.string().uuid(),
})

export const UpdatePresenceSchema = z.object({
  status: z.enum(['online','away','offline']),
})

Keine Implementierung — nur Types und Zod-Schemas exportieren.
```

---

## Prompt 3 — API Routes

**Abhängigkeit:** nach Prompt 1 + 2

**Neue Dateien:**
- `app/api/chat/messages/route.ts`
- `app/api/chat/messages/[id]/read/route.ts`
- `app/api/chat/messages/[id]/react/route.ts`
- `app/api/chat/upload/route.ts`

```
Implementiere folgende API Routes für Chat Phase 1.

Stack-Regeln (IMMER einhalten):
- requireTenant() auf jeder Route
- createServiceClient() für admin queries
- .eq('tenant_id', tenantId) explizit auf JEDER query zusätzlich zu RLS
- Zod-Validation mit SendMessageSchema / ToggleReactionSchema / MarkReadSchema aus types/chat.ts
- Fehler immer: return NextResponse.json({ error: '...' }, { status: 4xx })

─────────────────────────────────────────
1. app/api/chat/messages/route.ts
─────────────────────────────────────────
GET ?conversation_id=UUID&before=ISO&limit=50
- Paginierung: cursor-basiert via created_at DESC
- JOIN: sender (profiles Tabelle: full_name, avatar_url)
- JOIN: reply_to (rekursiv: id, content, sender.full_name)
- Filtere deleted_at IS NOT NULL → zeige "[Nachricht gelöscht]" als content, leere attachments

POST body: SendMessageSchema
- Insert message mit status='sent'
- Danach: UPDATE chat_conversation_members SET unread_count = unread_count + 1
  WHERE conversation_id = ? AND user_id != sender_id AND tenant_id = ?
- Return: vollständige Message mit sender join

─────────────────────────────────────────
2. app/api/chat/messages/[id]/read/route.ts
─────────────────────────────────────────
POST (kein body nötig)
- Upsert in message_read_receipts (message_id, user_id=currentUser, tenant_id)
- UPDATE chat_messages SET status='read' WHERE id=? AND tenant_id=?
  (nur wenn noch nicht 'read')
- UPDATE chat_conversation_members SET
    last_read_at=NOW(),
    unread_count=0
  WHERE user_id=currentUser AND conversation_id=(SELECT conversation_id FROM chat_messages WHERE id=?)
- Return: { ok: true }

─────────────────────────────────────────
3. app/api/chat/messages/[id]/react/route.ts
─────────────────────────────────────────
POST body: { emoji: string }
- Validiere mit ToggleReactionSchema
- Lade aktuelle reactions JSONB
- Toggle-Logik in TypeScript:
  - Wenn user_id bereits in reactions[emoji] → entfernen
  - Sonst → hinzufügen
  - Wenn reactions[emoji] leer nach Entfernen → Key löschen
- UPDATE chat_messages SET reactions=? WHERE id=? AND tenant_id=?
- Return: { reactions: ReactionsMap }

─────────────────────────────────────────
4. app/api/chat/upload/route.ts
─────────────────────────────────────────
POST multipart/form-data mit field 'file'
- Max 25MB, erlaubte MIME types:
  image/jpeg, image/png, image/gif, image/webp,
  application/pdf, text/plain,
  application/vnd.openxmlformats-officedocument.*
- SHA256 des File-Buffers berechnen (für Content-Addressable Storage)
- Upload zu Cloudflare R2 Bucket 'comentra-images'
  Key: chat/{tenantId}/{sha256}.{ext}
- Public URL: https://pub-5afd0726b6344560903091f23ad3707d.r2.dev/chat/{tenantId}/{sha256}.{ext}
- Return: MessageAttachment object

R2 credentials aus env: R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY, R2_ACCOUNT_ID
Verwende @aws-sdk/client-s3 (bereits im Projekt).
```

---

## Prompt 4 — Supabase Realtime Hook

**Abhängigkeit:** nach Prompt 1, 2, 3

**Neue Datei:** `hooks/useChat.ts`

```
Erstelle hooks/useChat.ts — zentraler Realtime-Hook für das Chat-System.

Stack: Supabase JS SDK (@supabase/supabase-js), React 18, TypeScript.
Importiere Types aus types/chat.ts.
Verwende createBrowserClient() aus @supabase/ssr für Client-Side.

Der Hook: useChatRealtime(conversationId: string, currentUserId: string)

Return-Value:
{
  messages: ChatMessage[]
  isLoading: boolean
  typingUsers: { user_id: string; user_name: string }[]
  sendMessage: (params: { content: string; reply_to_id?: string; attachments?: MessageAttachment[] }) => Promise<void>
  sendTyping: (isTyping: boolean) => void
  markRead: (messageId: string) => void
  toggleReaction: (messageId: string, emoji: string) => Promise<void>
  loadMore: () => Promise<void>
  hasMore: boolean
}

Implementierung:

1. MESSAGES LADEN
   - Beim Mount: fetch GET /api/chat/messages?conversation_id=X&limit=50
   - setMessages(data) — älteste zuerst (Array umkehren da API DESC liefert)
   - loadMore(): fetch nächste Seite mit before=messages[0].created_at

2. SUPABASE REALTIME — neue Nachrichten
   supabase.channel('chat:' + conversationId)
   .on('postgres_changes', {
     event: 'INSERT',
     schema: 'public',
     table: 'chat_messages',
     filter: 'conversation_id=eq.' + conversationId
   }, (payload) => {
     // Optimistic: Wenn eigene Nachricht schon temporär in state, ersetzen
     // Sonst: ans Ende anfügen
     setMessages(prev => [...prev.filter(m=>m.id!==payload.new.id), payload.new as ChatMessage])
   })
   .on('postgres_changes', {
     event: 'UPDATE',
     schema: 'public',
     table: 'chat_messages',
     filter: 'conversation_id=eq.' + conversationId
   }, (payload) => {
     // Status-Updates und Reactions live aktualisieren
     setMessages(prev => prev.map(m => m.id===payload.new.id ? {...m,...payload.new} : m))
   })

3. TYPING INDICATOR — via Broadcast (ephemeral, kein DB)
   .on('broadcast', { event: 'typing' }, ({ payload }: { payload: TypingPayload }) => {
     if (payload.user_id === currentUserId) return
     setTypingUsers(prev => {
       if (payload.is_typing) return [...prev.filter(u=>u.user_id!==payload.user_id), payload]
       return prev.filter(u=>u.user_id!==payload.user_id)
     })
     // Auto-remove nach 3s falls kein stop-Event kommt
   })
   .subscribe()

4. sendTyping(isTyping: boolean):
   - Debounce mit useRef (1500ms)
   - channel.send({ type: 'broadcast', event: 'typing', payload: { user_id, user_name, conversation_id, is_typing } })

5. sendMessage():
   - Optimistic Update: temporäre Message mit id='temp-'+Date.now(), status='sent'
   - POST /api/chat/messages
   - Bei Erfolg: temp-Message durch echte ersetzen
   - Bei Fehler: temp-Message entfernen, Toast-Error

6. markRead(messageId):
   - Sofort lokal: setMessages(prev => prev.map(m => m.id===messageId ? {...m,status:'read'} : m))
   - POST /api/chat/messages/[id]/read (fire and forget)

7. Cleanup: channel.unsubscribe() im useEffect cleanup

Kein UI — nur der Hook.
```

---

## Prompt 5 — MessageBubble + Context Menu

**Abhängigkeit:** nach Prompt 4

**Neue Dateien:**
- `components/chat/MessageBubble.tsx`
- `components/chat/MessageContextMenu.tsx`

```
Erstelle zwei Komponenten für die Chat-Nachrichtenanzeige.
Design-System: bg #F7F5F0, accent #C9521A, keine shadcn-Defaults, lucide-react Icons.

─────────────────────────────────────────
1. components/chat/MessageBubble.tsx
─────────────────────────────────────────
Props:
  message: ChatMessage
  isOwn: boolean
  onReply: (message: ChatMessage) => void
  onReact: (messageId: string, emoji: string) => void
  onMarkRead: (messageId: string) => void

Layout (wie WhatsApp):
- isOwn=true: Bubble rechtsbündig, Hintergrund #C9521A (accent), text weiß
- isOwn=false: Bubble linksbündig, Hintergrund #F7F5F0, border 0.5px

Reply-Preview (wenn message.reply_to vorhanden):
- Kleines graues Block oben in der Bubble
- 2px left-border in accent-Farbe
- Absender-Name + gekürzter Content (max 80 Zeichen)

Inhalt der Bubble:
- Wenn deleted_at: kursiver Text "Nachricht gelöscht" in grau, keine Reactions/Menü
- Content (normaler Text, Zeilenumbrüche erhalten)
- Attachments (s. unten)
- Unten rechts: Zeit (HH:mm) + Status-Icons

Status-Indikator (NUR bei isOwn=true, unten rechts in der Bubble):
- 'sent': ein grauer Haken (Check Icon 12px)
- 'delivered': zwei graue Haken (CheckCheck Icon 12px)
- 'read': zwei blaue Haken (CheckCheck Icon 12px, color #3B82F6)
Importiere Check, CheckCheck aus lucide-react.

Attachments anzeigen:
- image/*: <img> mit max-width 240px, border-radius 8px, cursor-pointer (Lightbox TODO)
- Andere: Download-Zeile mit FileIcon + Name + Größe (formatBytes) + Download-Link

Hover-Verhalten:
- Beim Hover: schmale Toolbar rechts neben der Bubble einblenden (opacity transition)
- Toolbar: Smile-Button (öffnet Quickreact), Reply-Button, More-Button (⋮)
- onClick auf More: setShowMenu(true)

Reactions anzeigen (wenn reactions nicht leer):
- Unter der Bubble: Flex-Row mit Pills
- Jede Pill: "{emoji} {count}" — wenn eigene user_id enthalten: leicht highlighted (accent bg 10%)
- onClick: onReact(messageId, emoji) zum Toggle

─────────────────────────────────────────
2. components/chat/MessageContextMenu.tsx
─────────────────────────────────────────
Props:
  message: ChatMessage
  isOwn: boolean
  position: { x: number; y: number }
  onClose: () => void
  onReply: () => void
  onReact: (emoji: string) => void
  onCopy: () => void
  onDelete: () => void  // nur bei isOwn

Menü (floating div, absolute positioniert):
- Quick-Reactions oben: 6 Emojis ['👍','❤️','😂','😮','😢','🙏'] als klickbare Buttons
- Trennlinie
- Menü-Items als Liste:
  - Antworten (Reply)
  - Kopieren
  - [wenn isOwn] Löschen (rot, disabled wenn älter als 30min via message.created_at)
- Click-Outside: onClose() via useEffect + mousedown listener
- Kein portal nötig, parent hat position:relative

Kein useState für open/close — das managed der Parent.
```

---

## Prompt 6 — EmojiPicker

**Abhängigkeit:** nach Prompt 5
**Hinweis:** `npm install emoji-mart @emoji-mart/data @emoji-mart/react` vorher ausführen

**Neue Datei:** `components/chat/EmojiPicker.tsx`

```
Installiere emoji-mart und erstelle components/chat/EmojiPicker.tsx.

Installation (in package.json eintragen + npm install):
  "emoji-mart": "^5.6.0"
  "@emoji-mart/data": "^1.2.1"
  "@emoji-mart/react": "^1.1.1"

Erstelle components/chat/EmojiPicker.tsx:

Props:
  mode: 'quick' | 'full'
  onSelect: (emoji: string) => void
  onClose: () => void

Quick-Mode (mode='quick'):
- Kompaktes Floating-Panel (200px breit)
- Zeile mit 6 Favoriten-Emojis: ['👍','❤️','😂','😮','😢','🙏']
- Darunter: "Mehr +" Button der zu full wechselt
- Jeder Emoji: 32px Button, hover-Hintergrund, font-size 22px

Full-Mode (mode='full'):
- Rendere <Picker> aus @emoji-mart/react
- data aus @emoji-mart/data (lazy import)
- Props:
    data={data}
    onEmojiSelect={(e: any) => onSelect(e.native)}
    locale="de"
    theme="light"
    previewPosition="none"
    skinTonePosition="none"
    navPosition="bottom"
- Wrap in div mit border 0.5px, border-radius 12px, overflow hidden

Click-Outside für beide Modi: useEffect mit mousedown listener → onClose()

Kein Portal — Parent positioned relativ.
Export als default.
```

---

## Prompt 7 — ChatInput

**Abhängigkeit:** nach Prompt 4, 5, 6

**Neue Datei:** `components/chat/ChatInput.tsx`

```
Erstelle components/chat/ChatInput.tsx — kompletter Message-Input-Bereich.

Props:
  onSend: (params: { content: string; reply_to_id?: string; attachments?: MessageAttachment[] }) => void
  onTyping: (isTyping: boolean) => void
  replyTo: ChatMessage | null
  onCancelReply: () => void
  disabled?: boolean

Struktur (von oben nach unten):

1. REPLY-PREVIEW (nur wenn replyTo !== null):
   - Graue Bar über dem Input
   - Links: 2px accent-border + Sender-Name (bold) + gekürzter Content
   - Rechts: X-Button → onCancelReply()
   - Hintergrund: leicht getönte Fläche (#F7F5F0)

2. ATTACHMENT-PREVIEWS (wenn attachments.length > 0):
   - Horizontale Scrollreihe mit Thumbnail-Cards (80x80px)
   - Images: <img> preview
   - Andere Files: FileIcon + Name
   - Jedes Preview hat X-Button zum Entfernen

3. INPUT-ROW:
   - Links: Paperclip-Button (öffnet file input, accept=".jpg,.png,.gif,.webp,.pdf,.txt")
   - Mitte: <textarea> auto-resize (min 1 Zeile, max 5 Zeilen)
     - onInput: Höhe anpassen via scrollHeight
     - onKeyDown: Enter ohne Shift → senden
     - onChange: text setzen + onTyping(true) debounced
   - Rechts: Send-Button (PaperPlane Icon) — disabled wenn content leer UND keine attachments

File-Upload-Flow:
  - onChange auf file input → Dateien sofort zu /api/chat/upload POSTen (multipart)
  - Während Upload: Spinner in der Preview-Card
  - Bei Erfolg: MessageAttachment zum lokalen State hinzufügen
  - Bei Fehler: Toast "Upload fehlgeschlagen"
  - Max 5 Attachments gleichzeitig

onSend():
  - Nur wenn content.trim() || attachments.length > 0
  - onTyping(false) aufrufen
  - State zurücksetzen: content='', attachments=[], replyTo wird vom Parent verwaltet
  - textarea Höhe zurücksetzen

Design:
  - Border-top 0.5px auf dem gesamten Input-Bereich
  - Padding 12px
  - Hintergrund #FFFFFF
  - Textarea: kein border, kein outline, font-size 14px, resize:none
```

---

## Prompt 8 — Sidebar Unread Badge + Online-Status + Browser Push

**Abhängigkeit:** nach Prompt 4, 7

**Neue Dateien:**
- `components/chat/ConversationListItem.tsx`
- `lib/chat/push-notifications.ts`

```
Zwei Implementierungen für Sidebar und Push-Notifications.

─────────────────────────────────────────
1. components/chat/ConversationListItem.tsx
─────────────────────────────────────────
Props:
  conversation: {
    id: string
    name: string
    avatar?: string
    last_message?: { content: string; created_at: string; sender_name: string }
    unread_count: number
    is_muted: boolean
  }
  presence?: UserPresence
  isActive: boolean
  onClick: () => void

Layout (wie WhatsApp Sidebar-Item):
- Höhe: 64px, padding 0 12px
- Avatar: 40px Kreis links — initials fallback wenn kein avatar
  - Online-Indikator: 10px grüner Punkt unten rechts am Avatar
    (presence.status === 'online' → grün #22C55E)
    (presence.status === 'away' → gelb #EAB308)
    (offline → kein Dot)
- Rechts: Flex-Column
  - Oben: Name (14px 500) + Zeit (12px grau) right-aligned
  - Unten: Last-message Preview (13px, 1 Zeile, overflow ellipsis) + Unread-Badge right-aligned
- Unread-Badge: runde Pill, bg #C9521A, weiße Zahl 11px
  - Wenn is_muted: grauer Badge statt rot
  - Nur anzeigen wenn unread_count > 0
- Active-State: bg #F7F5F0
- Hover: bg leicht getönt

─────────────────────────────────────────
2. lib/chat/push-notifications.ts
─────────────────────────────────────────
Exportiere folgende Funktionen:

async function requestPushPermission(): Promise<boolean>
  - Prüfe 'Notification' in window
  - Prüfe Notification.permission
  - Wenn 'granted': return true
  - Wenn 'default': Notification.requestPermission() → return result==='granted'
  - Wenn 'denied': return false

function showChatNotification(params: {
  senderName: string
  content: string
  conversationId: string
  avatarUrl?: string
}): void
  - Nur wenn document.hidden === true (Tab nicht aktiv)
  - Nur wenn Notification.permission === 'granted'
  - new Notification(params.senderName, {
      body: params.content.slice(0, 100),
      icon: params.avatarUrl || '/icon-192.png',
      tag: 'chat-' + params.conversationId,
      renotify: true,
    })
  - notification.onclick: window.focus() + window.location.href = '/chat/' + params.conversationId

Nutze diese Funktion in useChat.ts:
  Im Realtime-Handler für neue Nachrichten:
  if (payload.new.sender_id !== currentUserId) {
    showChatNotification({
      senderName: payload.new.sender?.full_name ?? 'Jemand',
      content: payload.new.content,
      conversationId: conversationId,
    })
  }

Kein Service Worker nötig für diesen MVP-Push.
```
