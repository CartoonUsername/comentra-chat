# Comentra Chat — Phase 4: Advanced Features

> Voraussetzung: Phase 1–3 deployed, alle 3 SQL-Migrationen auf Supabase applied.

---

## Prompt 1 — DB Migration Phase 4

**Datei:** `supabase/migrations/20260420_000000_chat_phase4.sql`

```
Erstelle supabase/migrations/20260420_000000_chat_phase4.sql

Alle Änderungen mit IF NOT EXISTS / DO $$ absichern.

1. chat_messages erweitern:
   - thread_id UUID REFERENCES chat_messages(id) ON DELETE CASCADE
     (NULL = Hauptnachricht, gesetzt = Antwort in Thread)
   - thread_reply_count INT NOT NULL DEFAULT 0
     (denormalisiert: wie viele Antworten hat diese Nachricht)
   - thread_last_reply_at TIMESTAMPTZ
   - thread_participants UUID[] DEFAULT '{}'
     (Array der user_ids die im Thread geantwortet haben, max 3 für Avatars)
   - is_forwarded BOOLEAN DEFAULT false
   - forwarded_from_id UUID REFERENCES chat_messages(id) ON DELETE SET NULL
   - is_starred BOOLEAN DEFAULT false
   - scheduled_at TIMESTAMPTZ
     (NULL = sofort, gesetzt = geplante Nachricht)
   - message_type erweitern: füge 'voice' und 'poll' zum CHECK hinzu
     ALTER TABLE chat_messages DROP CONSTRAINT IF EXISTS chat_messages_message_type_check;
     ALTER TABLE chat_messages ADD CONSTRAINT chat_messages_message_type_check
       CHECK (message_type IN ('text','system','order_link','product_link','alert','task','voice','poll'));

2. Neue Tabelle: chat_starred_messages
   - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
   - user_id UUID NOT NULL
   - message_id UUID REFERENCES chat_messages(id) ON DELETE CASCADE
   - conversation_id UUID REFERENCES chat_conversations(id) ON DELETE CASCADE
   - starred_at TIMESTAMPTZ DEFAULT NOW()
   - tenant_id UUID NOT NULL
   - UNIQUE(user_id, message_id)

3. Neue Tabelle: chat_polls
   - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
   - message_id UUID REFERENCES chat_messages(id) ON DELETE CASCADE
   - tenant_id UUID NOT NULL
   - question TEXT NOT NULL (max 300 Zeichen)
   - options JSONB NOT NULL
     Format: [{ id: uuid, text: string, voter_ids: string[] }]
   - is_anonymous BOOLEAN DEFAULT false
   - closes_at TIMESTAMPTZ (optional)
   - created_at TIMESTAMPTZ DEFAULT NOW()

4. Neue Tabelle: chat_scheduled_messages
   - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
   - tenant_id UUID NOT NULL
   - conversation_id UUID REFERENCES chat_conversations(id) ON DELETE CASCADE
   - sender_id UUID NOT NULL
   - content TEXT NOT NULL
   - attachments JSONB DEFAULT '[]'
   - reply_to_id UUID REFERENCES chat_messages(id) ON DELETE SET NULL
   - scheduled_at TIMESTAMPTZ NOT NULL
   - sent_at TIMESTAMPTZ (NULL = noch nicht gesendet)
   - cancelled_at TIMESTAMPTZ

5. Indexes:
   - chat_messages(thread_id) WHERE thread_id IS NOT NULL
   - chat_messages(scheduled_at) WHERE scheduled_at IS NOT NULL AND deleted_at IS NULL
   - chat_starred_messages(user_id, tenant_id)
   - chat_scheduled_messages(scheduled_at) WHERE sent_at IS NULL AND cancelled_at IS NULL

6. RPC: increment_thread_reply_count(parent_id UUID, replier_id UUID)
   - UPDATE chat_messages SET
       thread_reply_count = thread_reply_count + 1,
       thread_last_reply_at = NOW(),
       thread_participants = CASE
         WHEN replier_id = ANY(thread_participants) THEN thread_participants
         WHEN array_length(thread_participants, 1) >= 3 THEN thread_participants
         ELSE thread_participants || replier_id
       END
     WHERE id = parent_id
   - SECURITY DEFINER

7. RLS:
   - chat_starred_messages: user_id = auth.uid()
   - chat_polls: tenant_id = auth.jwt()->>'tenant_id'
   - chat_scheduled_messages: tenant_id = auth.jwt()->>'tenant_id'
     + sender_id = auth.uid() für INSERT/DELETE
```

---

## Prompt 2 — Types Phase 4

**Datei:** `types/chat.ts` (erweitern)

```
Erweitere types/chat.ts am Ende. Nichts Bestehendes anfassen.

// Thread-Erweiterung für ChatMessage (ergänze zum bestehenden Interface):
// thread_id: string | null
// thread_reply_count: number
// thread_last_reply_at: string | null
// thread_participants: string[]
// is_forwarded: boolean
// forwarded_from_id: string | null
// is_starred: boolean
// scheduled_at: string | null

export interface VoiceMetadata {
  url: string
  duration_seconds: number
  waveform: number[]   // ~50 Werte zwischen 0-1 für Waveform-Visualisierung
  mime_type: 'audio/webm' | 'audio/ogg' | 'audio/mp4'
}

export interface PollOption {
  id: string
  text: string
  voter_ids: string[]
}

export interface PollMetadata {
  poll_id: string
  question: string
  options: PollOption[]
  is_anonymous: boolean
  closes_at?: string
  total_votes: number
}

export interface ChatPoll {
  id: string
  message_id: string
  question: string
  options: PollOption[]
  is_anonymous: boolean
  closes_at?: string
  created_at: string
}

export interface StarredMessage {
  id: string
  user_id: string
  message_id: string
  conversation_id: string
  starred_at: string
  message?: ChatMessage
  conversation?: { name: string; type: string }
}

export interface ScheduledMessage {
  id: string
  conversation_id: string
  sender_id: string
  content: string
  attachments: MessageAttachment[]
  reply_to_id?: string
  scheduled_at: string
  sent_at?: string
  cancelled_at?: string
}

// Zod
export const CreatePollSchema = z.object({
  conversation_id: z.string().uuid(),
  question: z.string().min(3).max(300),
  options: z.array(z.string().min(1).max(100)).min(2).max(6),
  is_anonymous: z.boolean().default(false),
  closes_at: z.string().datetime().optional(),
})

export const VotePollSchema = z.object({
  poll_id: z.string().uuid(),
  option_id: z.string().uuid(),
})

export const ScheduleMessageSchema = z.object({
  conversation_id: z.string().uuid(),
  content: z.string().min(1).max(4000),
  scheduled_at: z.string().datetime(),
  reply_to_id: z.string().uuid().optional(),
})
```

---

## Prompt 3 — Thread API + Hook

**Abhängigkeit:** nach Prompt 1, 2

**Neue Dateien:**
- `app/api/chat/messages/[id]/thread/route.ts`
- `hooks/useThread.ts`

```
Implementiere Threaded Replies (Slack-Style).

─────────────────────────────────────────
1. app/api/chat/messages/[id]/thread/route.ts
─────────────────────────────────────────
GET — Lade alle Thread-Antworten einer Nachricht
- requireTenant(), createServiceClient()
- Prüfe: message existiert und tenant_id stimmt
- SELECT chat_messages WHERE thread_id=? AND tenant_id=?
  ORDER BY created_at ASC
- JOIN: sender (full_name, avatar_url)
- Return: { parent: ChatMessage, replies: ChatMessage[], total: number }

POST body: SendMessageSchema (mit thread_id gesetzt)
- Validiere: thread_id existiert, ist Hauptnachricht (thread_id IS NULL)
  → 400 wenn thread_id selbst auf eine Thread-Antwort zeigt (kein Thread-in-Thread)
- INSERT chat_messages mit thread_id=parent_id
- Rufe RPC increment_thread_reply_count(parent_id, currentUserId) auf
- Return: neue ChatMessage

─────────────────────────────────────────
2. hooks/useThread.ts
─────────────────────────────────────────
Hook: useThread(parentMessageId: string | null, currentUserId: string)

Return:
{
  parent: ChatMessage | null
  replies: ChatMessage[]
  isLoading: boolean
  sendReply: (content: string) => Promise<void>
  isOpen: boolean
  open: (message: ChatMessage) => void
  close: () => void
}

Implementierung:
- isOpen: true wenn parentMessageId !== null
- Lade GET /api/chat/messages/[id]/thread beim open()
- Supabase Realtime auf INSERT WHERE thread_id=parentMessageId
  (gleiche Pattern wie useChat für messages)
- sendReply: POST /api/chat/messages mit { conversation_id, content, thread_id: parentMessageId }
- Optimistic update für replies

Integriere useThread in die Chat-Page:
- State: const [threadMessageId, setThreadMessageId] = useState<string|null>(null)
- Wenn threadMessageId gesetzt: zeige Thread-Panel rechts (320px, border-left)
- MessageBubble bekommt Prop: onOpenThread: (messageId: string) => void
- Thread-Panel hat eigenen ChatInput (nur für Replies, kein Attachment nötig MVP)
```

---

## Prompt 4 — Thread UI Panel

**Abhängigkeit:** nach Prompt 3

**Neue Datei:** `components/chat/ThreadPanel.tsx`

```
Erstelle components/chat/ThreadPanel.tsx — Slack-style Thread-Panel.

'use client'
Props:
  parentMessage: ChatMessage
  replies: ChatMessage[]
  currentUserId: string
  onSendReply: (content: string) => void
  onClose: () => void

Layout (volle Höhe, 320px breit, border-left 0.5px, flex-col):

  HEADER (52px, border-bottom):
    - "Thread" (Syne, 15px, 500)
    - Anzahl Antworten (13px, grau): "{n} Antworten"
    - X-Button rechts → onClose()

  PARENT MESSAGE (padding 16px, border-bottom, bg leicht getönt):
    - Vollständige Darstellung der Ursprungs-Nachricht
    - Sender + Zeit + Content (kein Context-Menu, kein Reply-Button hier)

  REPLIES (flex-1, overflow-y-auto, padding 12px 16px):
    - replies.map() → Thread-Bubble:
      Kompaktere Version von MessageBubble:
      - Avatar (24px) + Name (12px bold) + Zeit (11px grau) inline
      - Content darunter (14px)
      - Reactions (kleiner, 11px)
      - Kein Status-Indikator (kein ✓✓ in Threads)
    - "Heute", "Gestern" Datum-Trenner (gleich wie Haupt-Chat)

  REPLY INPUT (border-top, padding 12px):
    - Vereinfachter ChatInput (kein Attachment-Button, kein Reply-Preview)
    - Placeholder: "Antworten..."
    - Enter → onSendReply()

Integration in MessageBubble:
  Füge Thread-Footer unter der Bubble hinzu (wenn thread_reply_count > 0):
  - Avatar-Stack (max 3 Avatars der thread_participants, 20px, überlappend)
  - "{n} Antwort/Antworten · Letzte vor {timeago}"
  - Gesamte Zeile klickbar → onOpenThread(message.id)
  - Hover: underline auf dem Text

Im Context-Menu: füge "In Thread antworten" hinzu (für alle message_types außer system)
```

---

## Prompt 5 — Voice Notes

**Abhängigkeit:** nach Prompt 1, 2

**Neue Dateien:**
- `components/chat/VoiceRecorder.tsx`
- `app/api/chat/upload/voice/route.ts`

```
Implementiere Voice-Note Aufnahme und Wiedergabe.

─────────────────────────────────────────
1. app/api/chat/upload/voice/route.ts
─────────────────────────────────────────
POST multipart/form-data mit field 'audio'
- requireTenant(), createServiceClient()
- Akzeptiere: audio/webm, audio/ogg, audio/mp4, audio/wav
- Max 10MB, max 120 Sekunden (aus einem duration Header-Wert den der Client mitschickt)
- SHA256 → upload zu R2:
  Key: voice/{tenantId}/{sha256}.{ext}
  Public URL: https://pub-5afd0726b6344560903091f23ad3707d.r2.dev/voice/{tenantId}/{sha256}.{ext}
- Waveform generieren: Empfange vom Client auch ein 'waveform' JSON-Field
  (Float32Array der Amplitude-Werte, vom Browser via AnalyserNode berechnet)
  Normalisiere auf 50 Werte zwischen 0.0 und 1.0
- Return: VoiceMetadata

─────────────────────────────────────────
2. components/chat/VoiceRecorder.tsx
─────────────────────────────────────────
'use client'
Props:
  onSend: (metadata: VoiceMetadata) => void
  onCancel: () => void

State Machine: 'idle' → 'recording' → 'preview' → 'uploading'

IDLE: Mikrofon-Button in ChatInput (ersetzt vorübergehend den Send-Button)

RECORDING (nach getUserMedia + MediaRecorder.start()):
  - Rotes Puls-Icon + Dauer-Counter (MM:SS, JetBrains Mono)
  - Waveform-Visualisierung: Canvas 160x32px, Echtzeit via AnalyserNode
    (requestAnimationFrame, Balken in accent-Farbe)
  - Stop-Button (roter Kreis) → state='preview'
  - Cancel-Button (X) → onCancel(), tracks stoppen

PREVIEW:
  - Statische Waveform-Anzeige + Dauer
  - Play-Button → Audio abspielen via new Audio(objectURL)
  - Nochmal-Button (Trash + Mikrofon) → state='recording'
  - Senden-Button → state='uploading'
    POST /api/chat/upload/voice (multipart mit audio blob + waveform JSON)
    → onSend(metadata)

Fehlerbehandlung:
  - getUserMedia denied → Toast "Mikrofonzugriff verweigert"
  - Kein MediaRecorder Support → Button gar nicht rendern

Integration in ChatInput.tsx:
- Wenn content leer UND keine Attachments: zeige Mikrofon-Icon statt Send-Button
- onClick Mikrofon → setShowVoiceRecorder(true)
- Render <VoiceRecorder onSend={(meta) => {
    onSend({ content: '', attachments: [], voiceMetadata: meta })
    setShowVoiceRecorder(false)
  }} onCancel={() => setShowVoiceRecorder(false)} />

─────────────────────────────────────────
3. Voice Playback in MessageCard.tsx
─────────────────────────────────────────
Füge Voice-Case hinzu (message_type='voice'):

Card:
- Mikrofon-Icon (accent)
- Play/Pause Button (16px)
- Waveform: SVG, 120x24px
  50 vertikale Balken, Breite 2px, Abstand 1px
  Bereits abgespielt: accent-Farbe (#C9521A)
  Noch nicht: grau
  Fortschritt: via Audio timeupdate Event (currentTime / duration)
- Dauer (JetBrains Mono, 12px): "0:23"

State: isPlaying, progress (0-1)
Audio-Instanz: useRef<HTMLAudioElement>(null)
```

---

## Prompt 6 — Polls

**Abhängigkeit:** nach Prompt 1, 2

**Neue Dateien:**
- `app/api/chat/polls/route.ts`
- `app/api/chat/polls/[id]/vote/route.ts`
- `components/chat/CreatePollModal.tsx`

```
Implementiere Umfragen (Polls) im Chat.

─────────────────────────────────────────
1. app/api/chat/polls/route.ts
─────────────────────────────────────────
POST body: CreatePollSchema
- requireTenant(), createServiceClient()
- Generiere option IDs: options.map(text => ({ id: crypto.randomUUID(), text, voter_ids: [] }))
- INSERT chat_polls (message_id=null erstmal, question, options JSONB, is_anonymous, closes_at, tenant_id)
- INSERT chat_messages (conversation_id, content=question, message_type='poll',
    metadata={ poll_id, question, options, is_anonymous, total_votes: 0 }, tenant_id)
- UPDATE chat_polls SET message_id = new_message_id
- Return: { message: ChatMessage, poll: ChatPoll }

─────────────────────────────────────────
2. app/api/chat/polls/[id]/vote/route.ts
─────────────────────────────────────────
POST body: VotePollSchema
- Lade Poll, prüfe tenant_id
- Prüfe: closes_at noch nicht überschritten → 400 "Umfrage geschlossen"
- Finde option_id in options array
- Toggle-Logik (wie Reactions):
  - Wenn currentUser bereits in voter_ids → entfernen (Stimme zurückziehen)
  - Sonst → hinzufügen
- UPDATE chat_polls SET options=updated_options WHERE id=?
- total_votes = Summe aller voter_ids.length
- UPDATE chat_messages SET metadata = metadata || updated_metadata
  WHERE id = poll.message_id
- Return: { options: PollOption[], total_votes: number }

─────────────────────────────────────────
3. components/chat/CreatePollModal.tsx
─────────────────────────────────────────
Props: conversationId, onClose, onCreated

Modal (400px):
  - Question Input (Textarea, max 300 Zeichen)
  - Options: dynamische Liste
    - 2 Options vorausgefüllt ("Option 1", "Option 2") — leer, user tippt rein
    - "+ Option hinzufügen" Button (max 6)
    - Jede Option hat X zum Entfernen (min 2 müssen bleiben)
    - Drag-to-reorder (TODO, vorerst nur feste Reihenfolge)
  - Toggle: "Anonyme Abstimmung" (Checkbox)
  - Optional: "Enddatum" Date+Time Input
  - Submit: POST /api/chat/polls → onCreated()

Integration in ChatInput.tsx:
- Füge Poll-Icon (BarChart2 aus lucide) in der Toolbar hinzu
- onClick → setShowCreatePoll(true)

─────────────────────────────────────────
4. Poll Rendering in MessageCard.tsx
─────────────────────────────────────────
Poll-Case (message_type='poll'):

Header: BarChart2-Icon + "Umfrage" Badge + Frage (16px, 500)
Wenn closes_at: "Endet {relative time}" (12px, grau) — rot wenn < 1h

Options als Vote-Buttons:
  Jede Option (48px hoch, border 0.5px, border-radius 8px, margin 4px 0):
  - Linke Seite: Optionstext
  - Rechte Seite: Stimmenanzahl + Prozent (JetBrains Mono)
  - Progress-Bar: Breite = (voter_count / total_votes * 100)%, bg accent 20%
  - Wenn currentUser hat gestimmt:
    - Eigene Option: accent-border (2px), accent bg (5%)
    - Alle Optionen zeigen Prozent (vorher versteckt wenn noch nicht gestimmt, außer is_anonymous=false)
  - onClick → POST /api/chat/polls/[id]/vote

Footer:
  - "{total_votes} Stimme(n)"
  - Wenn is_anonymous: "Anonyme Abstimmung"
  - Wenn geschlossen: "Umfrage beendet"

Optimistic Update: Stimme sofort lokal togglen, dann API aufrufen
```

---

## Prompt 7 — Starred Messages + Message Forwarding + Scheduled Messages

**Abhängigkeit:** nach Prompt 1, 2

**Neue Dateien:**
- `app/api/chat/starred/route.ts`
- `app/api/chat/scheduled/route.ts`
- `components/chat/StarredMessagesDrawer.tsx`
- `components/chat/ScheduleMessageModal.tsx`

```
Implementiere Starring, Forwarding und Message Scheduling.

─────────────────────────────────────────
1. Starring API
─────────────────────────────────────────
app/api/chat/starred/route.ts:

GET — eigene gesternten Nachrichten
- SELECT chat_starred_messages WHERE user_id=currentUser AND tenant_id=?
- JOIN: message (vollständig mit sender)
- JOIN: conversation (name, type)
- ORDER BY starred_at DESC LIMIT 50
- Return: StarredMessage[]

POST body: { message_id: string }
- UPSERT chat_starred_messages (user_id, message_id, conversation_id, tenant_id)
  ON CONFLICT DO NOTHING
- UPDATE chat_messages SET is_starred=true WHERE id=? AND tenant_id=?
- Return: { ok: true, starred: true }

DELETE ?message_id=UUID
- DELETE chat_starred_messages WHERE user_id=currentUser AND message_id=?
- Prüfe: noch andere User haben diese Message gestarrt? Wenn nein:
  UPDATE chat_messages SET is_starred=false WHERE id=?
- Return: { ok: true, starred: false }

components/chat/StarredMessagesDrawer.tsx:
  Slide-in (rechts, 360px) mit Sternchen-Header
  Liste der gesternten Nachrichten:
  - Conversation-Name Badge
  - Sender + Datum
  - Content Preview (2 Zeilen) oder MessageCard für non-text
  - Klick → navigiert zum Chat + scroll-to-message Event
  - Unstar Button (⭐→☆) rechts

Integration in ConversationList Header:
  Star-Icon Button → setShowStarred(true)

Im MessageContextMenu: "Markieren" / "Markierung entfernen" Toggle
  (Basis: message.is_starred)

─────────────────────────────────────────
2. Message Forwarding
─────────────────────────────────────────
Füge in MessageContextMenu "Weiterleiten" hinzu:

ForwardMessageModal (inline, keine neue Datei nötig):
  Modal (320px):
  - "Weiterleiten an..." Titel
  - Suchfeld + Liste aller eigenen Conversations (aus bestehendem State)
  - Multi-Select: max 5 Conversations
  - Submit: für jede ausgewählte Conversation:
    POST /api/chat/messages {
      conversation_id,
      content: original.content,
      attachments: original.attachments,
      is_forwarded: true,
      forwarded_from_id: original.id
    }
  - Return: forwarded count

In MessageBubble: wenn is_forwarded=true, zeige kleinen Header:
  ↗ "Weitergeleitet" (12px, grau, italic) über dem Content

Ergänze POST /api/chat/messages/route.ts:
  Akzeptiere is_forwarded: boolean und forwarded_from_id: string optional im Body
  (Zod-Schema um diese Felder erweitern)

─────────────────────────────────────────
3. Scheduled Messages
─────────────────────────────────────────
app/api/chat/scheduled/route.ts:

GET ?conversation_id=UUID — eigene geplante Nachrichten
- SELECT chat_scheduled_messages WHERE sender_id=currentUser AND tenant_id=?
  AND conversation_id=? AND sent_at IS NULL AND cancelled_at IS NULL
- ORDER BY scheduled_at ASC
- Return: ScheduledMessage[]

POST body: ScheduleMessageSchema
- Validiere: scheduled_at > NOW() + 1 Minute → sonst 400
- INSERT chat_scheduled_messages
- Return: ScheduledMessage

DELETE [id] — Geplante Nachricht abbrechen
- Prüfe: sender_id=currentUser
- Prüfe: sent_at IS NULL → sonst 400 "Bereits gesendet"
- UPDATE chat_scheduled_messages SET cancelled_at=NOW() WHERE id=?
- Return: { ok: true }

BullMQ Job für Scheduled Messages (in instrumentation.ts):
  Repeatable Job: alle 60 Sekunden, Queue 'system-jobs'
  Job-Handler (workers/send-scheduled-messages.ts):
  - SELECT chat_scheduled_messages WHERE scheduled_at <= NOW()
    AND sent_at IS NULL AND cancelled_at IS NULL
  - Für jede: POST /api/chat/messages (intern via createServiceClient direkt)
  - UPDATE chat_scheduled_messages SET sent_at=NOW() WHERE id=?

components/chat/ScheduleMessageModal.tsx:
  Props: conversationId, content, replyToId, onScheduled, onClose
  
  Modal (340px):
  - Zeigt Content-Preview (readonly, grau)
  - DateTime-Picker: date + time Inputs
    Vorbelegung: morgen 09:00 Uhr
    Min: jetzt + 2 Minuten
  - Shortcut-Buttons:
    "Morgen 9:00" / "Nächste Woche" / "Datum wählen"
  - Submit: POST /api/chat/scheduled → onScheduled()

Integration in ChatInput:
  Long-Press auf Send-Button (500ms) oder kleiner Dropdown-Pfeil neben Send:
  - "Jetzt senden" (default)
  - "Senden planen" → öffnet ScheduleMessageModal

Zeige geplante Nachrichten-Indikator in ConversationListItem:
  Kleines Uhr-Icon wenn scheduled_count > 0 für diese Conversation
```

---

## Prompt 8 — Final Wiring + E2E Badge + Smoke Test

**Abhängigkeit:** nach Prompt 3–7

```
Verbinde alle Phase-4-Features und führe Smoke Test durch.

─────────────────────────────────────────
1. Chat-Page: Thread Panel einbauen
─────────────────────────────────────────
In app/(dashboard)/chat/[conversationId]/page.tsx:

State: const [threadMessageId, setThreadMessageId] = useState<string|null>(null)
const { parent, replies, sendReply, close } = useThread(threadMessageId, currentUserId)

Layout anpassen wenn Thread offen:
  - Normale Message-Liste: flex-1 (wird schmaler wenn Thread offen)
  - Thread-Panel rechts: 320px (nur wenn threadMessageId !== null)
  - Smooth transition via CSS: width transition 200ms

MessageBubble erhält:
  onOpenThread={(id) => setThreadMessageId(id)}

─────────────────────────────────────────
2. Chat-Header: neue Icons
─────────────────────────────────────────
Header-Dropdown (MoreVertical) ergänzen:
  - "⭐ Markierte Nachrichten" → setShowStarred(true)
  - "🗓 Geplante Nachrichten" → setShowScheduled(true) (neue Drawer-Komponente)

Geplante Nachrichten Drawer (inline im Header-Dropdown als Panel):
  - Liste aus GET /api/chat/scheduled?conversation_id=X
  - Jeder Eintrag: Zeit (JetBrains Mono) + Content-Preview + Löschen-Button
  - "Keine geplanten Nachrichten" Empty-State

─────────────────────────────────────────
3. E2E-Verschlüsselung Indicator
─────────────────────────────────────────
Im Chat-Header für Direct Messages (type='direct'):
  Zeige kleines Lock-Icon (12px, grau) mit Tooltip "Ende-zu-Ende verschlüsselt"
  (Nur ein visueller Indikator — keine echte E2E-Implementierung in diesem MVP)

Für Groups und Channels: kein Lock-Icon (zu komplex für MVP)

Implementierung: simples Icon neben dem Presence-Status-Text
  {conversation.type === 'direct' && (
    <span title="Ende-zu-Ende verschlüsselt" style={{fontSize:12,color:'gray'}}>🔒</span>
  )}

─────────────────────────────────────────
4. Smoke Test
─────────────────────────────────────────
npx tsc --noEmit

Häufige Fehler Phase 4:

- useThread Hook: parentMessageId kann null sein →
  alle Supabase-Channel-Subscribes mit if(!parentMessageId) return absichern

- VoiceRecorder: MediaRecorder + AnalyserNode nur im Browser verfügbar →
  'use client' prüfen, typeof window !== 'undefined' Guards

- ScheduleMessageModal: datetime-local Input gibt lokale Zeit zurück,
  nicht UTC. Konvertierung: new Date(localValue).toISOString() vor dem API-Call

- Poll Rendering: total_votes kann 0 sein → Division durch 0 bei Prozentrechnung
  Schutz: (voter_ids.length / Math.max(total_votes, 1)) * 100

- Thread Panel: wenn thread_reply_count=0 und onOpenThread geklickt →
  zeige leeres Panel mit Input. Kein Crash.

- ForwardMessageModal: conversations State kommt aus ConversationList —
  muss als prop oder Context nach unten weitergegeben werden.
  Einfachste Lösung: eigener fetch in der Modal-Komponente.

Nach 0 Fehlern:
  docker compose up --build -d in /opt/comentra-amazon

Dann manuell testen:
  1. Sprachnachricht aufnehmen + abspielen
  2. Poll erstellen + abstimmen
  3. Nachricht für morgen 09:00 planen
  4. Thread öffnen + Antwort schreiben
  5. Nachricht weiterleiten
  6. Nachricht markieren + in Starred Drawer sehen
```
