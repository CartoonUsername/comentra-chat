# Comentra Chat — Phase 2: Gruppen & Kanäle

> Voraussetzung: Phase 1 vollständig deployed + SQL-Migration 20260417_000000_chat_phase1.sql applied.

---

## Prompt 1 — DB Migration Phase 2

**Datei:** `supabase/migrations/20260418_000000_chat_phase2.sql`

```
Erstelle supabase/migrations/20260418_000000_chat_phase2.sql

Alle Änderungen mit IF NOT EXISTS / DO $$ EXCEPTION absichern.

1. chat_conversations erweitern:
   - type TEXT CHECK (type IN ('direct','group','channel')) DEFAULT 'direct'
     (Falls Spalte schon existiert: nur CHECK-Constraint ergänzen)
   - topic TEXT (Kanal-Beschreibung, max 200 Zeichen)
   - is_archived BOOLEAN DEFAULT false
   - created_by UUID REFERENCES auth.users(id)
   - avatar_url TEXT

2. chat_conversation_members erweitern:
   - role TEXT CHECK (role IN ('member','admin','owner')) DEFAULT 'member'
   - joined_at TIMESTAMPTZ DEFAULT NOW()

3. Neue Tabelle: chat_pinned_messages
   - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
   - conversation_id UUID REFERENCES chat_conversations(id) ON DELETE CASCADE
   - message_id UUID REFERENCES chat_messages(id) ON DELETE CASCADE
   - pinned_by UUID NOT NULL
   - pinned_at TIMESTAMPTZ DEFAULT NOW()
   - tenant_id UUID NOT NULL
   - UNIQUE(conversation_id, message_id)

4. Neue Tabelle: chat_mentions
   - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
   - message_id UUID REFERENCES chat_messages(id) ON DELETE CASCADE
   - mentioned_user_id UUID NOT NULL
   - conversation_id UUID REFERENCES chat_conversations(id) ON DELETE CASCADE
   - is_read BOOLEAN DEFAULT false
   - tenant_id UUID NOT NULL
   - created_at TIMESTAMPTZ DEFAULT NOW()

5. chat_messages erweitern:
   - is_pinned BOOLEAN DEFAULT false (denormalisiert für schnelle Abfrage)
   - mentions UUID[] DEFAULT '{}' (Array der erwähnten user_ids)

6. Neue RPC: create_channel(name TEXT, topic TEXT, member_ids UUID[], tenant_id UUID)
   - INSERT chat_conversations (type='channel', name, topic, tenant_id, created_by=auth.uid())
   - INSERT chat_conversation_members für alle member_ids + caller als 'owner'
   - RETURN conversation_id UUID

7. Indexes:
   - chat_pinned_messages(conversation_id)
   - chat_mentions(mentioned_user_id, is_read) WHERE is_read = false
   - chat_conversations(tenant_id, type)
   - chat_messages USING GIN (to_tsvector('german', content))
     (Für Volltext-Suche)

8. RLS:
   - chat_pinned_messages: tenant_id = auth.jwt()->>'tenant_id'
   - chat_mentions: mentioned_user_id = auth.uid() OR tenant_id = auth.jwt()->>'tenant_id'
   - chat_conversations SELECT: Mitglied in chat_conversation_members
     EXISTS (SELECT 1 FROM chat_conversation_members
             WHERE conversation_id = id AND user_id = auth.uid())
```

---

## Prompt 2 — Types Phase 2 erweitern

**Datei:** `types/chat.ts` (erweitern, nicht ersetzen)

```
Erweitere types/chat.ts um Phase-2-Types. Bestehende Exports nicht anfassen.

Füge am Ende der Datei hinzu:

// Conversation type erweitern
export type ConversationType = 'direct' | 'group' | 'channel'

export interface ChatConversation {
  id: string
  type: ConversationType
  name: string
  topic?: string
  avatar_url?: string
  is_archived: boolean
  created_by: string
  tenant_id: string
  created_at: string
  // joined fields
  unread_count: number
  is_muted: boolean
  my_role: 'member' | 'admin' | 'owner'
  last_message?: {
    content: string
    created_at: string
    sender_name: string
  }
  member_count?: number
}

export interface PinnedMessage {
  id: string
  conversation_id: string
  message_id: string
  message: ChatMessage  // joined
  pinned_by: string
  pinned_at: string
}

export interface ChatMention {
  id: string
  message_id: string
  conversation_id: string
  is_read: boolean
  created_at: string
  // joined
  message?: ChatMessage
  conversation?: { name: string }
}

export interface ConversationMember {
  user_id: string
  conversation_id: string
  role: 'member' | 'admin' | 'owner'
  joined_at: string
  // joined
  user?: {
    id: string
    full_name: string
    avatar_url?: string
  }
}

// Zod
export const CreateChannelSchema = z.object({
  name: z.string().min(2).max(80).regex(/^[a-z0-9-]+$/, 'Nur Kleinbuchstaben, Zahlen und Bindestriche'),
  topic: z.string().max(200).optional(),
  member_ids: z.array(z.string().uuid()).min(0).max(50),
})

export const InviteMembersSchema = z.object({
  conversation_id: z.string().uuid(),
  user_ids: z.array(z.string().uuid()).min(1).max(20),
})

export const UpdateMemberRoleSchema = z.object({
  conversation_id: z.string().uuid(),
  user_id: z.string().uuid(),
  role: z.enum(['member', 'admin']),
})
```

---

## Prompt 3 — Channel & Gruppen API Routes

**Abhängigkeit:** nach Prompt 1, 2

**Neue Dateien:**
- `app/api/chat/channels/route.ts`
- `app/api/chat/channels/[id]/members/route.ts`
- `app/api/chat/channels/[id]/members/[userId]/route.ts`
- `app/api/chat/conversations/[id]/route.ts`

```
Implementiere Channel- und Gruppen-Verwaltungs-API.

Stack-Regeln: requireTenant(), createServiceClient(), .eq('tenant_id', tenantId) überall.

─────────────────────────────────────────
1. app/api/chat/channels/route.ts
─────────────────────────────────────────
GET — Liste alle Channels des Tenants
- SELECT chat_conversations WHERE type='channel' AND tenant_id=?
- JOIN: member_count (COUNT aus chat_conversation_members)
- JOIN: eigene Mitgliedschaft (is_member boolean)
- Trenne in: my_channels (bin Mitglied) und browse_channels (bin nicht Mitglied)
- Return: { my_channels: ChatConversation[], browse_channels: ChatConversation[] }

POST body: CreateChannelSchema
- Validiere: name noch nicht vergeben im Tenant
  SELECT id FROM chat_conversations WHERE name=? AND type='channel' AND tenant_id=?
  → 409 "Kanalname bereits vergeben" wenn vorhanden
- Rufe RPC create_channel(name, topic, member_ids, tenant_id) auf
- Return: { conversation_id: string }

─────────────────────────────────────────
2. app/api/chat/channels/[id]/members/route.ts
─────────────────────────────────────────
GET — Mitgliederliste eines Channels/Gruppe
- requireTenant()
- Prüfe: currentUser ist Mitglied → sonst 403
- SELECT chat_conversation_members JOIN profiles ON user_id
  WHERE conversation_id=? AND tenant_id=?
- Return: ConversationMember[]

POST body: InviteMembersSchema — Mitglieder einladen
- Prüfe: currentUser hat role='admin' oder 'owner' → sonst 403
- INSERT chat_conversation_members für alle user_ids (IGNORE CONFLICT)
- Return: { added: number }

─────────────────────────────────────────
3. app/api/chat/channels/[id]/members/[userId]/route.ts
─────────────────────────────────────────
PATCH body: UpdateMemberRoleSchema — Rolle ändern
- Prüfe: currentUser hat role='owner' → sonst 403
- Prüfe: userId !== currentUser (sich selbst nicht degradieren)
- UPDATE chat_conversation_members SET role=? WHERE conversation_id=? AND user_id=? AND tenant_id=?
- Return: { ok: true }

DELETE — Mitglied entfernen / selbst austreten
- Wenn userId === currentUser: Austritt erlaubt (außer letzter Owner)
- Sonst: currentUser muss admin oder owner sein
- Wenn letzter Owner austritt: 400 "Zuerst anderen Owner ernennen"
- DELETE FROM chat_conversation_members WHERE conversation_id=? AND user_id=? AND tenant_id=?
- Return: { ok: true }

─────────────────────────────────────────
4. app/api/chat/conversations/[id]/route.ts
─────────────────────────────────────────
GET — Details einer Conversation
- JOIN: members (max 5 für Avatar-Stack), member_count, my_role
- Return: ChatConversation mit members[]

PATCH — Conversation bearbeiten (name, topic, avatar_url)
- Prüfe: currentUser hat role='admin' oder 'owner' → sonst 403
- UPDATE chat_conversations SET name=?, topic=?, avatar_url=? WHERE id=? AND tenant_id=?
- Return: aktualisierte Conversation

DELETE — Conversation archivieren (kein Hard-Delete)
- Prüfe: currentUser hat role='owner' → sonst 403
- UPDATE chat_conversations SET is_archived=true WHERE id=? AND tenant_id=?
- Return: { ok: true }
```

---

## Prompt 4 — @Mention System

**Abhängigkeit:** nach Prompt 1, 2, 3

**Neue Dateien:**
- `lib/chat/parse-mentions.ts`
- `app/api/chat/mentions/route.ts`
- `components/chat/MentionInput.tsx` (erweitert ChatInput)

```
Implementiere @Mention-System end-to-end.

─────────────────────────────────────────
1. lib/chat/parse-mentions.ts
─────────────────────────────────────────
Exportiere:

function parseMentions(content: string): string[]
  // Regex: /@\[([^\]]+)\]\(([a-f0-9-]{36})\)/g
  // Format in content: @[Bessam Baschar](uuid)
  // Return: Array der user_ids

function renderMentions(content: string, currentUserId: string): React.ReactNode
  // Ersetze @[Name](uuid) durch <span> Tags
  // currentUserId wird erwähnt → orange highlighted (#C9521A bg-10%)
  // Andere Mentions → bold blue
  // Normaler Text bleibt normaler Text

function formatMention(user: { id: string; full_name: string }): string
  // Return: "@[{full_name}]({id})"

─────────────────────────────────────────
2. app/api/chat/mentions/route.ts
─────────────────────────────────────────
GET — eigene ungelesene Mentions
- SELECT chat_mentions WHERE mentioned_user_id=currentUser AND is_read=false AND tenant_id=?
- JOIN: message (content, created_at, sender.full_name)
- JOIN: conversation (name, type)
- ORDER BY created_at DESC LIMIT 50
- Return: ChatMention[]

POST /api/chat/mentions/read-all
- UPDATE chat_mentions SET is_read=true
  WHERE mentioned_user_id=currentUser AND tenant_id=?
- Return: { ok: true }

Ergänze POST /api/chat/messages/route.ts:
Nach dem Insert der Nachricht:
- Rufe parseMentions(content) auf
- Wenn mentions.length > 0:
  INSERT chat_mentions (message_id, mentioned_user_id, conversation_id, tenant_id)
  für jede user_id in mentions
- Setze chat_messages.mentions = Array der user_ids via UPDATE

─────────────────────────────────────────
3. components/chat/MentionInput.tsx
─────────────────────────────────────────
Erweitere ChatInput um @-Mention-Autocomplete.
(Ersetze die textarea in ChatInput.tsx durch MentionInput)

Verhalten:
- User tippt "@" → öffne Dropdown mit Mitgliederliste
- User tippt weiter "@bes" → filtere nach full_name (case-insensitive)
- Mitglieder kommen aus GET /api/chat/channels/[id]/members
  (beim Mount laden, cachen)
- Dropdown: max 5 Einträge, Avatar-Initials + full_name
- Enter oder Klick → füge formatMention(user) in den Text ein
- Escape → schließe Dropdown
- Trigger: "@" gefolgt von mindestens 1 Buchstabe ohne Leerzeichen

Dropdown-Styling:
- Absolute positioniert über dem Input (bottom: 100%)
- border 0.5px, border-radius 8px, bg white
- Jedes Item: 36px hoch, px-3, hover-bg
- Highlighted Item: accent-bg (#C9521A 8%)

In MessageBubble: ersetze {message.content} durch renderMentions(content, currentUserId)
```

---

## Prompt 5 — Pin Messages

**Abhängigkeit:** nach Prompt 3

**Neue Dateien:**
- `app/api/chat/messages/[id]/pin/route.ts`
- `app/api/chat/conversations/[id]/pins/route.ts`
- `components/chat/PinnedMessagesBanner.tsx`

```
Implementiere Pin-System für Nachrichten.

─────────────────────────────────────────
1. app/api/chat/messages/[id]/pin/route.ts
─────────────────────────────────────────
POST — Nachricht pinnen
- requireTenant(), createServiceClient()
- Prüfe: currentUser hat role='admin' oder 'owner' in dieser Conversation → sonst 403
- Prüfe: max 3 gepinnte Nachrichten pro Conversation
  SELECT COUNT(*) FROM chat_pinned_messages WHERE conversation_id=? → 400 wenn >= 3
- INSERT chat_pinned_messages (conversation_id, message_id, pinned_by, tenant_id)
- UPDATE chat_messages SET is_pinned=true WHERE id=? AND tenant_id=?
- Return: { ok: true }

DELETE — Nachricht loslösen
- Prüfe: currentUser hat role='admin' oder 'owner' → sonst 403
- DELETE FROM chat_pinned_messages WHERE message_id=? AND tenant_id=?
- UPDATE chat_messages SET is_pinned=false WHERE id=? AND tenant_id=?
- Return: { ok: true }

─────────────────────────────────────────
2. app/api/chat/conversations/[id]/pins/route.ts
─────────────────────────────────────────
GET — alle gepinnten Nachrichten einer Conversation
- SELECT chat_pinned_messages WHERE conversation_id=? AND tenant_id=?
- JOIN: message (vollständig mit sender)
- ORDER BY pinned_at DESC
- Return: PinnedMessage[]

─────────────────────────────────────────
3. components/chat/PinnedMessagesBanner.tsx
─────────────────────────────────────────
'use client'
Props:
  conversationId: string
  myRole: 'member' | 'admin' | 'owner'

Verhalten:
- Lade GET /api/chat/conversations/[id]/pins beim Mount
- Wenn keine pins: render null
- Wenn 1 pin: zeige Banner direkt
- Wenn mehrere: zeige ersten pin, "▸ X weitere" öffnet Drawer

Banner (unter Chat-Header, über Message-Liste):
  - Höhe: 44px, border-bottom 0.5px, bg #FFF8F6 (leicht orange getönt)
  - Links: Pin-Icon (lucide PinIcon, 14px, accent-Farbe)
  - Mitte: Sender-Name + gekürzter Content (max 60 Zeichen)
  - Rechts: [wenn admin/owner] X-Button → unpin via DELETE
  - onClick auf Content: scroll zur Nachricht in der Liste
    (window.dispatchEvent(new CustomEvent('scroll-to-message', { detail: { id } })))

In MessageContextMenu: füge "Anpinnen" / "Loslösen" Item hinzu
  - Nur sichtbar wenn myRole='admin' oder 'owner'
  - Toggle basierend auf message.is_pinned
```

---

## Prompt 6 — Volltextsuche

**Abhängigkeit:** nach Prompt 1 (GIN Index)

**Neue Dateien:**
- `app/api/chat/search/route.ts`
- `components/chat/ChatSearch.tsx`

```
Implementiere Chat-Volltext-Suche.

─────────────────────────────────────────
1. app/api/chat/search/route.ts
─────────────────────────────────────────
GET ?q=string&conversation_id=UUID(optional)&limit=20&offset=0

- requireTenant(), createServiceClient()
- Validiere: q.length >= 2 → sonst 400
- Query:
  SELECT
    cm.*,
    p.full_name as sender_name,
    cc.name as conversation_name,
    cc.type as conversation_type,
    ts_headline('german', cm.content,
      plainto_tsquery('german', $query),
      'MaxWords=10, MinWords=5, ShortWord=2, HighlightAll=false'
    ) as highlight
  FROM chat_messages cm
  JOIN profiles p ON p.id = cm.sender_id
  JOIN chat_conversations cc ON cc.id = cm.conversation_id
  JOIN chat_conversation_members ccm ON ccm.conversation_id = cc.id
    AND ccm.user_id = {currentUserId}
  WHERE cm.tenant_id = {tenantId}
    AND cm.deleted_at IS NULL
    AND to_tsvector('german', cm.content) @@ plainto_tsquery('german', {q})
    AND (conversation_id = {conversation_id} OR {conversation_id} IS NULL)
  ORDER BY cm.created_at DESC
  LIMIT {limit} OFFSET {offset}

- Return: { results: Array<ChatMessage & { highlight: string; conversation_name: string }>, total: number }

─────────────────────────────────────────
2. components/chat/ChatSearch.tsx
─────────────────────────────────────────
'use client'
Props:
  conversationId?: string  // wenn gesetzt: nur in diesem Chat suchen
  onClose: () => void
  onResultClick: (messageId: string, conversationId: string) => void

State: query, results, isLoading, hasMore, page

UI (Fullscreen-Overlay oder Slide-in Panel, 480px breit):
  - Input oben mit Search-Icon, auto-focus beim Mount
  - Escape → onClose()
  - Debounce 400ms auf query → fetch wenn >= 2 Zeichen

  Ergebnisse:
  - Jedes Ergebnis als Card:
    - Conversation-Name + Type-Badge (#-Icon für Channel, Personen-Icon für DM)
    - Sender-Name + Datum (DD.MM.YYYY HH:mm)
    - highlight (HTML via dangerouslySetInnerHTML — Supabase ts_headline gibt <b>-Tags zurück)
      Ersetze <b> durch <mark style="background:#FFF0E8;color:#C9521A;border-radius:2px">
    - onClick: onResultClick(message_id, conversation_id) + onClose()
  - "Mehr laden" Button wenn hasMore
  - Leer-State: "Keine Ergebnisse für '{query}'"

Integration in Chat-Header:
  - Search-Icon Button im Header → setShowSearch(true)
  - onResultClick: router.push('/chat/' + conversationId)
    + CustomEvent 'scroll-to-message' dispatchen
```

---

## Prompt 7 — Admin-Rollen UI + Mute

**Abhängigkeit:** nach Prompt 3

**Neue Dateien:**
- `components/chat/ChannelMembersDrawer.tsx`
- `components/chat/MuteToggle.tsx`

```
Implementiere Mitglieder-Verwaltung UI und Mute-Funktion.

─────────────────────────────────────────
1. components/chat/ChannelMembersDrawer.tsx
─────────────────────────────────────────
'use client'
Props:
  conversationId: string
  myRole: 'member' | 'admin' | 'owner'
  isOpen: boolean
  onClose: () => void

Daten: GET /api/chat/channels/[id]/members

Layout (Slide-in von rechts, 320px, border-left 0.5px):
  HEADER:
  - Titel "Mitglieder ({count})"
  - X-Button

  EINLADEN (nur wenn myRole='admin'|'owner'):
  - "Mitglied einladen" Button → öffnet User-Picker Modal
  - User-Picker: Autocomplete auf GET /api/team/users (bestehende Route)
    Zeige nur User die noch nicht Mitglied sind
  - POST /api/chat/channels/[id]/members mit ausgewählten user_ids

  MITGLIEDER-LISTE:
  Sortierung: owner → admins → members

  Jedes Item (48px hoch):
  - Avatar-Initials (32px) + full_name + Rolle-Badge
  - Rolle-Badge: "Owner" (grau), "Admin" (amber), "Member" (nichts)
  - [wenn myRole='owner' UND item.role !== 'owner']:
    Dropdown-Button (⋮):
    - "Zum Admin machen" / "Admin entfernen"
      → PATCH /api/chat/channels/[id]/members/[userId] { role }
    - "Aus Gruppe entfernen"
      → DELETE /api/chat/channels/[id]/members/[userId]
      → Confirm-Dialog davor

  FOOTER (nur wenn myRole='member'|'admin'):
  - "Gruppe verlassen" Button (rot) → DELETE .../members/currentUserId → redirect /chat

─────────────────────────────────────────
2. Mute-Toggle — erweitere ConversationListItem
─────────────────────────────────────────
Füge in ConversationListItem einen Mute-Button hinzu (Rechtsklick-Kontextmenü oder Hover-Menü):

Neue API-Route app/api/chat/conversations/[id]/mute/route.ts:
POST body: { is_muted: boolean }
- UPDATE chat_conversation_members SET is_muted=?
  WHERE conversation_id=? AND user_id=currentUser AND tenant_id=?
- Return: { ok: true }

Im ConversationListItem:
- Rechtsklick (onContextMenu) → kleines Menü:
  - "Stummschalten" / "Stummschaltung aufheben" (je nach aktuellem Status)
  - "Archivieren" (TODO Phase 4)
- Nach Mute: Badge wird grau, keine Browser-Push-Notification mehr
  (In push-notifications.ts: prüfe is_muted vor showChatNotification)

─────────────────────────────────────────
3. Integration in Chat-Header
─────────────────────────────────────────
Ergänze den Chat-Header (app/(dashboard)/chat/[conversationId]/page.tsx):
- MoreVertical-Icon rechts → Dropdown:
  - "Mitglieder anzeigen" → setShowMembers(true) (öffnet ChannelMembersDrawer)
  - [wenn channel] "Kanal-Info bearbeiten" → Edit-Modal (name, topic)
  - "Stummschalten" / "Lautschalten"
  - [wenn owner] "Kanal archivieren"
```

---

## Prompt 8 — Haupt-Chat-Page: Phase 2 wiring + Smoke Test

**Abhängigkeit:** nach Prompt 3–7

```
Verbinde alle Phase-2-Komponenten in der Chat-Hauptseite und führe Smoke Test durch.

─────────────────────────────────────────
1. app/(dashboard)/chat/[conversationId]/page.tsx anpassen
─────────────────────────────────────────
Ergänze folgende State und Komponenten:

State:
  const [showSearch, setShowSearch] = useState(false)
  const [showMembers, setShowMembers] = useState(false)
  const [myRole, setMyRole] = useState<'member'|'admin'|'owner'>('member')

Beim Mount: lade myRole aus GET /api/chat/conversations/[id] → setMyRole(data.my_role)

In der MessageList:
  - Vor der Message-Liste: <PinnedMessagesBanner conversationId={id} myRole={myRole} />
  - In MessageBubble: übergebe myRole für Pin-Option im ContextMenu
  - Lausche auf CustomEvent 'scroll-to-message':
    useEffect(() => {
      const handler = (e: CustomEvent) => {
        document.getElementById('msg-'+e.detail.id)?.scrollIntoView({ behavior: 'smooth' })
      }
      window.addEventListener('scroll-to-message', handler as EventListener)
      return () => window.removeEventListener('scroll-to-message', handler as EventListener)
    }, [])
  - Jede MessageBubble bekommt id={'msg-'+message.id}

Im Header:
  - Search-Icon onClick: setShowSearch(true)
  - MoreVertical onClick: setShowHeaderMenu(true)

Portale (absolute über alles):
  {showSearch && <ChatSearch conversationId={id} onClose={() => setShowSearch(false)}
    onResultClick={(msgId, convId) => { router.push('/chat/'+convId) }} />}
  {showMembers && <ChannelMembersDrawer conversationId={id} myRole={myRole}
    isOpen={showMembers} onClose={() => setShowMembers(false)} />}

ChatInput: Ersetze textarea durch <MentionInput> (aus Prompt 4)

─────────────────────────────────────────
2. ConversationList: Channels-Sektion
─────────────────────────────────────────
Erweitere components/chat/ConversationList.tsx:

Oben im Sidebar:
  - "+" Button rechts vom Titel → öffnet "Neuen Kanal erstellen" Modal
  - Modal: Input für name (Regex-Hint: nur a-z, 0-9, -), textarea für topic,
    User-Picker für Mitglieder, Submit → POST /api/chat/channels

Gruppierung in der Liste:
  - "DIREKTNACHRICHTEN" Sektion-Header (11px, uppercase, grau, non-clickable)
  - Direct + Group Conversations
  - Trennlinie
  - "KANÄLE" Sektion-Header
  - Channel Conversations (mit #-Prefix im Namen)
  - "Kanal beitreten" Link → öffnet Channel-Browser (GET /api/chat/channels browse_channels)

─────────────────────────────────────────
3. Smoke Test
─────────────────────────────────────────
npx tsc --noEmit

Häufige Fehler Phase 2:
- ConversationMember in types/chat.ts doppelt definiert (Phase 1 hatte es auch) → zusammenführen
- parseMentions return type: stelle sicher dass es string[] ist, nicht RegExpMatchArray
- dangerouslySetInnerHTML in ChatSearch: TypeScript braucht { __html: string }
- myRole prop fehlt in MessageContextMenu wenn aus Page weitergegeben

Nach 0 Fehlern: docker compose up --build -d in /opt/comentra-amazon
```
