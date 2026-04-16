# Comentra Chat — Phase 1 Prompts Teil 2 (Prompts 9–12)

> Anhang zu chat-phase1-prompts.md — erst ausführen wenn Prompts 1–8 abgeschlossen sind.

---

## Prompt 9 — Haupt-Chat-Page (alles zusammenführen)

**Abhängigkeit:** nach Prompt 4, 5, 6, 7, 8

**Dateien:**
- `app/(dashboard)/chat/page.tsx` (Redirect-Wrapper)
- `app/(dashboard)/chat/[conversationId]/page.tsx` (Haupt-Page)

```
Erstelle die Chat-Hauptseite die alle bisherigen Komponenten zusammenführt.

Stack-Regeln: Next.js App Router, requireTenant(), createServiceClient() für Server-Daten,
alle Client-Komponenten mit 'use client'. Design: bg #F7F5F0, accent #C9521A.

─────────────────────────────────────────
1. app/(dashboard)/chat/page.tsx
─────────────────────────────────────────
Server Component:
- Lade erste Conversation des Tenants aus chat_conversations
  (ORDER BY updated_at DESC LIMIT 1)
- Wenn vorhanden: redirect('/chat/' + conversation.id)
- Wenn keine: zeige leeren State "Noch keine Chats" mit Avatar-Icon

─────────────────────────────────────────
2. app/(dashboard)/chat/[conversationId]/page.tsx
─────────────────────────────────────────
Layout: Zwei-Spalten, volle Viewport-Höhe (h-screen), kein overflow auf body

LINKE SPALTE (280px, border-right 0.5px):
  - Render <ConversationList activeConversationId={conversationId} />

RECHTE SPALTE (flex-1, flex-col):
  - HEADER (60px, border-bottom 0.5px, px-4):
    - Avatar + Name der aktiven Conversation
    - Online-Status-Text ("Online" / "Zuletzt gesehen HH:mm") aus presence
    - Rechts: Search-Icon (TODO), MoreVertical-Icon (TODO)

  - MESSAGE-LISTE (flex-1, overflow-y-auto, px-4, py-2):
    Client-Komponente <MessageList>:
    - Rendere messages.map() → <MessageBubble>
      isOwn: message.sender_id === currentUserId
      onReply: (msg) => setReplyTo(msg)
      onReact: (id, emoji) => toggleReaction(id, emoji)
      onMarkRead: (id) => markRead(id)
    - Typing-Indicator unter letzter Nachricht:
      Wenn typingUsers.length > 0:
        Bubble mit "..." Animation (3 Punkte, CSS keyframes opacity)
        Text: "{name} schreibt..."
    - Scroll-to-bottom bei neuer eigener Nachricht (useRef + scrollIntoView)
    - "Load more" Button oben wenn hasMore=true
    - Datum-Trenner zwischen Nachrichten verschiedener Tage:
      Graue Pill mit Datum ("Heute", "Gestern", "DD.MM.YYYY")

  - INPUT-BEREICH (auto-height, border-top 0.5px):
    <ChatInput
      onSend={sendMessage}
      onTyping={sendTyping}
      replyTo={replyTo}
      onCancelReply={() => setReplyTo(null)}
    />

State in der Client-Komponente:
  const { messages, isLoading, typingUsers, sendMessage, sendTyping,
          markRead, toggleReaction, loadMore, hasMore } = useChatRealtime(conversationId, currentUserId)
  const [replyTo, setReplyTo] = useState<ChatMessage | null>(null)

Intersection Observer auf erste Nachricht → loadMore() aufrufen (für auto-pagination beim Hochscrollen).

Beim Mounten: markRead für letzte sichtbare Nachricht aufrufen.
```

---

## Prompt 10 — Presence Heartbeat Hook

**Abhängigkeit:** nach Prompt 1, 3

**Neue Dateien:**
- `app/api/chat/presence/route.ts`
- `hooks/usePresence.ts`

```
Implementiere Presence-System (online/away/offline) für Chat.

─────────────────────────────────────────
1. app/api/chat/presence/route.ts
─────────────────────────────────────────
POST body: { status: 'online' | 'away' | 'offline' }
- requireTenant(), createServiceClient()
- Upsert in user_presence:
  { user_id: currentUser, tenant_id, status, last_seen: NOW() }
  ON CONFLICT (user_id) DO UPDATE SET status=EXCLUDED.status, last_seen=NOW()
- Return: { ok: true }

GET ?tenant_id=UUID (optional: &user_ids=id1,id2,id3)
- Lade user_presence WHERE tenant_id=? AND last_seen > NOW() - INTERVAL '5 minutes'
- Wenn user_ids param: zusätzlich filter
- Setze status='offline' für User mit last_seen älter als 5min (client-side, nicht in DB)
- Return: UserPresence[]

─────────────────────────────────────────
2. hooks/usePresence.ts
─────────────────────────────────────────
Hook: usePresence(userIds: string[])

Return: { presenceMap: Record<string, UserPresence> }

Implementierung:
- Beim Mount: POST /api/chat/presence { status: 'online' }
- Heartbeat: setInterval alle 30s → POST { status: 'online' }
- Page Visibility API:
  document.addEventListener('visibilitychange', () => {
    POST { status: document.hidden ? 'away' : 'online' }
  })
- Fetch presenceMap: GET /api/chat/presence?user_ids={userIds.join(',')} alle 60s
- Cleanup:
  clearInterval(heartbeat)
  POST { status: 'offline' }  // fire-and-forget beim Unmount
  navigator.sendBeacon('/api/chat/presence', JSON.stringify({ status: 'offline' }))
  // sendBeacon wichtig: funktioniert auch beim Tab-Schließen

Nutze usePresence in der Chat-Page:
  const conversationUserIds = conversations.map(c => c.other_user_id)
  const { presenceMap } = usePresence(conversationUserIds)
  // presenceMap[userId] → UserPresence | undefined
```

---

## Prompt 11 — ConversationList Sidebar

**Abhängigkeit:** nach Prompt 8, 10

**Neue Datei:** `components/chat/ConversationList.tsx`

```
Erstelle components/chat/ConversationList.tsx — die linke Sidebar mit allen Chats.

'use client'
Props:
  activeConversationId: string

Daten laden:
GET /api/chat/conversations (neue Route — muss auch erstellt werden)

Erstelle dafür app/api/chat/conversations/route.ts:
- requireTenant(), createServiceClient()
- Query:
  SELECT
    cc.id,
    cc.type,
    cc.name,
    ccm.unread_count,
    ccm.is_muted,
    cm_last.content as last_message_content,
    cm_last.created_at as last_message_at,
    p.full_name as last_message_sender
  FROM chat_conversations cc
  JOIN chat_conversation_members ccm ON ccm.conversation_id = cc.id
    AND ccm.user_id = {currentUserId}
  LEFT JOIN LATERAL (
    SELECT content, created_at, sender_id
    FROM chat_messages
    WHERE conversation_id = cc.id AND deleted_at IS NULL
    ORDER BY created_at DESC LIMIT 1
  ) cm_last ON true
  LEFT JOIN profiles p ON p.id = cm_last.sender_id
  WHERE cc.tenant_id = {tenantId}
  ORDER BY last_message_at DESC NULLS LAST
- Return: ConversationListItem[]

Komponente ConversationList:
- State: conversations[], searchQuery, isLoading
- Supabase Realtime auf chat_conversation_members für unread_count Updates:
  supabase.channel('sidebar')
  .on('postgres_changes', { event: 'UPDATE', table: 'chat_conversation_members',
      filter: 'user_id=eq.' + currentUserId }, (payload) => {
    setConversations(prev => prev.map(c =>
      c.id === payload.new.conversation_id
        ? { ...c, unread_count: payload.new.unread_count }
        : c
    ))
  }).subscribe()

Layout (volle Höhe, flex-col):
  HEADER (56px):
    - Titel "Chat" (Syne font, 18px)
    - New-Chat Button (Plus Icon, rechts)

  SEARCH (px-3, py-2):
    - Input mit Search-Icon links
    - Filter: conversations.filter(c => c.name.toLowerCase().includes(query))

  LISTE (flex-1, overflow-y-auto):
    - conversations.map() → <ConversationListItem
        conversation={c}
        presence={presenceMap[c.other_user_id]}
        isActive={c.id === activeConversationId}
        onClick={() => router.push('/chat/' + c.id)}
      />
    - Wenn leer + kein Search: "Keine Chats vorhanden"
    - Wenn leer + Search: "Keine Ergebnisse für '{query}'"

  Gesamtzahl unread (optional): document.title = unread > 0 ? `(${unread}) Chat` : 'Chat'
```

---

## Prompt 12 — Delete API + 0-Error Smoke Test

**Abhängigkeit:** nach allen vorherigen Prompts

**Dateien:**
- `app/api/chat/messages/[id]/route.ts`
- Smoke Test inline

```
Zwei abschließende Aufgaben für Chat Phase 1.

─────────────────────────────────────────
1. app/api/chat/messages/[id]/route.ts
─────────────────────────────────────────
DELETE (Soft-Delete einer eigenen Nachricht)
- requireTenant(), createServiceClient()
- Lade message: SELECT sender_id, created_at FROM chat_messages WHERE id=? AND tenant_id=?
- Prüfe: message.sender_id === currentUserId → sonst 403
- Prüfe: created_at > NOW() - INTERVAL '30 minutes' → sonst 403 "Zu alt zum Löschen"
- UPDATE chat_messages SET
    deleted_at = NOW(),
    content = '',
    attachments = '[]',
    reactions = '{}'
  WHERE id=? AND tenant_id=?
- Return: { ok: true }

PATCH (Nachricht bearbeiten — optional aber sauber):
- body: { content: string } (min 1, max 4000 Zeichen via Zod)
- Prüfe: sender_id === currentUserId → sonst 403
- Prüfe: created_at > NOW() - INTERVAL '15 minutes' → sonst 403
- UPDATE chat_messages SET content=?, edited_at=NOW() WHERE id=? AND tenant_id=?
- Return: aktualisierte Message

─────────────────────────────────────────
2. TypeScript Smoke Test
─────────────────────────────────────────
Führe aus: npx tsc --noEmit

Erwartetes Ergebnis: 0 Fehler.

Falls Fehler auftreten, behebe sie in dieser Priorität:
1. Fehlende Imports in hooks/useChat.ts (MessageAttachment, TypingPayload)
2. Type-Mismatch zwischen API Response und ChatMessage Interface
3. Fehlende 'use client' Direktiven auf interaktiven Komponenten
4. Implicit any in Supabase Realtime payload handlers
   → Lösung: (payload: { new: ChatMessage }) statt (payload: any)
5. Fehlende return types auf async API Route Handler

Nach 0 Fehlern: docker compose up --build -d auf dem VPS ausführen.
Deployment-Pfad: /opt/comentra-amazon
```
