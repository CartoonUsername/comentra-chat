# Comentra Chat — Phase 3: Comentra-Integrationen

> Voraussetzung: Phase 1 + 2 deployed, beide SQL-Migrationen auf Supabase applied.

---

## Prompt 1 — DB Migration Phase 3

**Datei:** `supabase/migrations/20260419_000000_chat_phase3.sql`

```
Erstelle supabase/migrations/20260419_000000_chat_phase3.sql

Alle Änderungen mit IF NOT EXISTS absichern.

1. chat_messages erweitern:
   - message_type TEXT NOT NULL DEFAULT 'text'
     CHECK (message_type IN ('text','system','order_link','product_link','alert','task'))
   - metadata JSONB NOT NULL DEFAULT '{}'
     Wird je nach message_type unterschiedlich befüllt:
     - order_link:   { order_id, order_number, status, total, marketplace }
     - product_link: { product_id, asin, title, image_url, price }
     - alert:        { level: 'info'|'warning'|'error', source, job_id? }
     - task:         { task_id, title, assignee_id, due_date, status }
     - system:       { action: 'member_joined'|'member_left'|'channel_created'|'pinned' }

2. Neue Tabelle: chat_alert_rules
   - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
   - tenant_id UUID NOT NULL
   - conversation_id UUID REFERENCES chat_conversations(id) ON DELETE CASCADE
   - trigger_type TEXT NOT NULL
     CHECK (trigger_type IN (
       'order_new','order_cancelled','order_shipped',
       'return_new','return_approved',
       'bullmq_failure','bullmq_dead_letter',
       'marketplace_error','low_stock',
       'daily_kpi'
     ))
   - is_active BOOLEAN DEFAULT true
   - config JSONB DEFAULT '{}'
     Beispiel für daily_kpi: { "send_at": "08:00", "timezone": "Europe/Berlin" }
   - created_at TIMESTAMPTZ DEFAULT NOW()
   - UNIQUE(tenant_id, conversation_id, trigger_type)

3. Neue Tabelle: chat_tasks
   - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
   - tenant_id UUID NOT NULL
   - message_id UUID REFERENCES chat_messages(id) ON DELETE SET NULL
   - conversation_id UUID REFERENCES chat_conversations(id) ON DELETE CASCADE
   - title TEXT NOT NULL
   - description TEXT
   - assignee_id UUID
   - due_date DATE
   - status TEXT CHECK (status IN ('open','in_progress','done')) DEFAULT 'open'
   - created_by UUID NOT NULL
   - created_at TIMESTAMPTZ DEFAULT NOW()

4. Indexes:
   - chat_messages(conversation_id, message_type) WHERE message_type != 'text'
   - chat_alert_rules(tenant_id, trigger_type) WHERE is_active = true
   - chat_tasks(tenant_id, status, assignee_id)

5. RLS:
   - chat_alert_rules: tenant_id = auth.jwt()->>'tenant_id', nur admin/owner darf INSERT/UPDATE
   - chat_tasks: tenant_id = auth.jwt()->>'tenant_id'

6. Neue RPC: post_system_message(conversation_id UUID, content TEXT, metadata JSONB, tenant_id UUID)
   - INSERT chat_messages (conversation_id, sender_id=NULL, content, message_type='system',
     metadata, tenant_id, status='read')
   - SECURITY DEFINER (damit Server-Side ohne Auth aufrufen kann)
   - Return: message_id UUID
```

---

## Prompt 2 — Types Phase 3

**Datei:** `types/chat.ts` (erweitern)

```
Erweitere types/chat.ts am Ende. Nichts bestehender anfassen.

export type MessageType = 'text' | 'system' | 'order_link' | 'product_link' | 'alert' | 'task'

// Metadata je nach message_type
export interface OrderLinkMetadata {
  order_id: string
  order_number: string
  status: string
  total: number
  currency: string
  marketplace: 'amazon' | 'ebay' | 'otto' | 'shopify' | 'kaufland'
  customer_name?: string
}

export interface ProductLinkMetadata {
  product_id: string
  asin?: string
  ean?: string
  title: string
  image_url?: string
  price?: number
  currency?: string
  marketplace?: string
}

export interface AlertMetadata {
  level: 'info' | 'warning' | 'error'
  source: string        // z.B. 'bullmq', 'amazon-sp-api', 'ebay-webhook'
  job_id?: string
  error_message?: string
  retries?: number
}

export interface TaskMetadata {
  task_id: string
  title: string
  assignee_id?: string
  assignee_name?: string
  due_date?: string
  status: 'open' | 'in_progress' | 'done'
}

export interface SystemMetadata {
  action: 'member_joined' | 'member_left' | 'channel_created' | 'pinned' | 'name_changed'
  actor_name?: string
  target_name?: string
}

// Erweitertes ChatMessage (message_type + metadata)
// Füge zu bestehendem ChatMessage Interface hinzu:
// message_type: MessageType
// metadata: OrderLinkMetadata | ProductLinkMetadata | AlertMetadata | TaskMetadata | SystemMetadata | {}

export interface ChatTask {
  id: string
  tenant_id: string
  message_id: string | null
  conversation_id: string
  title: string
  description?: string
  assignee_id?: string
  due_date?: string
  status: 'open' | 'in_progress' | 'done'
  created_by: string
  created_at: string
}

export interface AlertRule {
  id: string
  conversation_id: string
  trigger_type: string
  is_active: boolean
  config: Record<string, unknown>
}

// Zod
export const CreateTaskSchema = z.object({
  message_id: z.string().uuid().optional(),
  conversation_id: z.string().uuid(),
  title: z.string().min(2).max(200),
  description: z.string().max(1000).optional(),
  assignee_id: z.string().uuid().optional(),
  due_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
})

export const UpsertAlertRuleSchema = z.object({
  conversation_id: z.string().uuid(),
  trigger_type: z.enum([
    'order_new','order_cancelled','order_shipped',
    'return_new','return_approved',
    'bullmq_failure','bullmq_dead_letter',
    'marketplace_error','low_stock','daily_kpi'
  ]),
  is_active: z.boolean(),
  config: z.record(z.unknown()).optional().default({}),
})
```

---

## Prompt 3 — Chat-Alert Service (Server-Side)

**Abhängigkeit:** nach Prompt 1, 2

**Neue Datei:** `lib/chat/alert-service.ts`

```
Erstelle lib/chat/alert-service.ts — zentraler Service der Chat-Nachrichten
aus anderen Teilen der Codebase auslöst.

Verwende createServiceClient() (server-only, nie im Browser aufrufen).
Importiere Types aus types/chat.ts.

─────────────────────────────────────────
Exportiere folgende Funktionen:
─────────────────────────────────────────

async function findAlertConversation(
  tenantId: string,
  triggerType: string
): Promise<string | null>
  - SELECT conversation_id FROM chat_alert_rules
    WHERE tenant_id=? AND trigger_type=? AND is_active=true
    LIMIT 1
  - Return conversation_id oder null

async function postSystemMessage(params: {
  tenantId: string
  conversationId: string
  content: string
  messageType?: MessageType
  metadata?: Record<string, unknown>
}): Promise<void>
  - Rufe RPC post_system_message auf
  - Fehler loggen aber nie werfen (fire-and-forget safe)

─────────────────────────────────────────
Exportiere spezialisierte Alert-Funktionen:
─────────────────────────────────────────

export async function alertNewOrder(tenantId: string, order: {
  id: string; order_number: string; status: string; total: number;
  currency: string; marketplace: string; customer_name?: string
}): Promise<void>
  - conversationId = await findAlertConversation(tenantId, 'order_new')
  - Wenn null: return (kein Alert konfiguriert)
  - content: "🛒 Neue Bestellung #{order_number} von {customer_name ?? 'Kunde'} — {total} {currency}"
  - postSystemMessage({ messageType: 'order_link', metadata: order, ... })

export async function alertOrderCancelled(tenantId, order): Promise<void>
  - triggerType: 'order_cancelled'
  - content: "❌ Bestellung #{order_number} wurde storniert"

export async function alertOrderShipped(tenantId, order): Promise<void>
  - triggerType: 'order_shipped'
  - content: "📦 Bestellung #{order_number} wurde versandt"

export async function alertNewReturn(tenantId: string, ret: {
  id: string; order_number: string; reason: string; marketplace: string
}): Promise<void>
  - triggerType: 'return_new'
  - content: "↩ Neue Retoure zu #{order_number}: {reason}"
  - metadata: { order_id, order_number, marketplace }

export async function alertBullMQFailure(tenantId: string, job: {
  id: string; name: string; queue: string; error: string; retries: number
}): Promise<void>
  - triggerType: 'bullmq_failure'
  - content: "⚠️ Job '{job.name}' in Queue '{job.queue}' fehlgeschlagen (Versuch {job.retries})"
  - metadata: { level: 'warning', source: 'bullmq', job_id: job.id, error_message: job.error }

export async function alertBullMQDeadLetter(tenantId: string, job): Promise<void>
  - triggerType: 'bullmq_dead_letter'
  - content: "🔴 Job '{job.name}' in Dead Letter Queue — manuelle Intervention nötig"
  - metadata: { level: 'error', source: 'bullmq', ... }

export async function postDailyKPI(tenantId: string, kpi: {
  orders_today: number; revenue_today: number; returns_today: number;
  open_orders: number; currency: string
}): Promise<void>
  - triggerType: 'daily_kpi'
  - content: mehrzeiliger KPI-Report:
    "📊 Tagesbericht {DD.MM.YYYY}\n\n✅ Bestellungen: {orders_today}\n💶 Umsatz: {revenue_today} {currency}\n↩ Retouren: {returns_today}\n🕐 Offene Bestellungen: {open_orders}"
  - messageType: 'system'
```

---

## Prompt 4 — BullMQ Integration (Alert-Hooks einbauen)

**Abhängigkeit:** nach Prompt 3

```
Integriere alertBullMQFailure und alertBullMQDeadLetter aus lib/chat/alert-service.ts
in das bestehende BullMQ Worker-System.

Suche alle Worker-Dateien im Projekt (Pattern: **/workers/*.ts, **/jobs/*.ts).
Importiere dort die Alert-Funktionen.

Für jeden Worker füge einen globalen Error-Handler hinzu:

worker.on('failed', async (job, error) => {
  if (!job) return
  const tenantId = job.data?.tenantId ?? job.data?.tenant_id
  if (!tenantId) return

  await alertBullMQFailure(tenantId, {
    id: job.id ?? 'unknown',
    name: job.name,
    queue: worker.name,
    error: error.message,
    retries: job.attemptsMade,
  })
})

Für Dead Letter / completed-with-failure:
worker.on('error', async (error) => {
  // Global worker error — kein Job-Kontext verfügbar
  // Logge nur, kein Alert (kein tenantId bekannt)
  console.error('[Worker Error]', worker.name, error)
})

Ergänze in instrumentation.ts den bestehenden Daily-KPI-Cron
(oder erstelle einen neuen falls nicht vorhanden):

Cron: täglich 07:00 Europe/Berlin
Name: 'daily-kpi-chat-report'
Queue: 'system-jobs' (oder bestehende Cron-Queue)

Job-Handler (neue Datei workers/daily-kpi-report.ts oder in bestehenden System-Worker einbauen):
- Lade alle tenant_ids mit aktivem 'daily_kpi' Alert-Rule
- Für jeden Tenant: lade KPI-Daten aus DB (orders, revenue, returns für heute)
- Rufe postDailyKPI(tenantId, kpi) auf

Keine neuen Queues erstellen falls bereits eine Cron-Queue existiert —
in die bestehende einbauen.
```

---

## Prompt 5 — Marketplace-Alert-Hooks

**Abhängigkeit:** nach Prompt 3

```
Integriere Chat-Alerts in bestehende Marketplace-Webhooks und Order-Sync-Routen.

Suche folgende bestehende Dateien/Routes und ergänze sie:

─────────────────────────────────────────
1. eBay Webhook Handler (app/api/ebay/webhook/route.ts oder ähnlich)
─────────────────────────────────────────
Nach erfolgreichem Order-Processing:
- Bei MARKETPLACE_ORDER_CREATED:
  await alertNewOrder(tenantId, { order_number, status, total, currency: 'EUR', marketplace: 'ebay' })
- Bei MARKETPLACE_ORDER_CANCELLED:
  await alertOrderCancelled(tenantId, { order_number, ... })

─────────────────────────────────────────
2. Amazon SP-API Order Sync (Worker oder Route)
─────────────────────────────────────────
Gleiche Pattern wie eBay.
Suche nach dem Code-Pfad der neue Amazon-Orders in die DB schreibt.
Dort einfügen:
  await alertNewOrder(tenantId, { ..., marketplace: 'amazon' })

─────────────────────────────────────────
3. Otto Order Sync
─────────────────────────────────────────
Gleich. marketplace: 'otto'

─────────────────────────────────────────
4. Retouren-Route (app/api/returns/... oder ähnlich)
─────────────────────────────────────────
Nach erfolgreichem Retouren-Insert:
  await alertNewReturn(tenantId, { order_number, reason, marketplace })

─────────────────────────────────────────
Wichtig:
─────────────────────────────────────────
- Alle alertXxx() Aufrufe sind fire-and-forget: await aber niemals try/catch den gesamten
  Request-Handler davon abhängig machen. Wrap in:
  alertNewOrder(...).catch(err => console.error('[Chat Alert]', err))
- Kein Alert wenn kein Rule konfiguriert (findAlertConversation gibt null zurück → return)
- Keine neuen DB-Tabellen, keine neuen Queues — nur Hooks in bestehende Flows
```

---

## Prompt 6 — Order-Link-Card + Produkt-Karte UI

**Abhängigkeit:** nach Prompt 2

**Neue Datei:** `components/chat/MessageCard.tsx`

```
Erstelle components/chat/MessageCard.tsx — Rich-Card-Rendering für
order_link, product_link, alert und system Nachrichten.

Design: bg #F7F5F0, accent #C9521A, Syne für Überschriften, JetBrains Mono für Zahlen/IDs.
Keine shadcn-Defaults, lucide-react Icons.

─────────────────────────────────────────
Props:
  message: ChatMessage  (message_type + metadata befüllt)
─────────────────────────────────────────

Rendere je nach message_type:

── ORDER_LINK ──
Card (border-left 3px, Farbe je Marketplace):
  amazon → #FF9900, ebay → #0064D2, otto → #F00020, shopify → #96BF48
  Header: Marketplace-Badge + "Bestellung #{order_number}"
  Body:
    Status-Badge (Farbe je Status: neu=blau, versandt=grün, storniert=rot)
    Kunde (wenn vorhanden)
    Betrag (JetBrains Mono, 16px, bold)
  Footer: "Bestellung öffnen →" Link zu /orders/{order_id}

── PRODUCT_LINK ──
Card (border-left 3px, accent-Farbe):
  Links: Produktbild (48x48px, border-radius 4px) oder Placeholder-Box
  Rechts:
    Titel (2 Zeilen max, overflow ellipsis)
    ASIN oder EAN (JetBrains Mono, 12px, grau)
    Preis (wenn vorhanden, JetBrains Mono)
  Footer: "Produkt öffnen →" Link zu /products/{product_id}

── ALERT ──
Card je level:
  info: border-left 3px #3B82F6, bg #EFF6FF
  warning: border-left 3px #EAB308, bg #FEFCE8
  error: border-left 3px #EF4444, bg #FEF2F2
  Icon: Info / AlertTriangle / AlertOctagon (lucide, 16px)
  Source-Badge (grau, 10px): metadata.source
  Content: message.content (normaler Text)
  Job-ID wenn vorhanden (JetBrains Mono, 11px, grau, klickbar → /jobs/{job_id} TODO)

── SYSTEM ──
Kein Card — zentrierte graue Zeile:
  <div style="text-align:center; font-size:12px; color:grau; padding:8px 0">
    {message.content}
  </div>
  Kein Avatar, kein Sender-Name, kein Timestamp rechts

── TASK ──
Card (border-left 3px, je status: open=grau, in_progress=amber, done=grün):
  Checkbox-Icon links (checked wenn done)
  Titel (durchgestrichen wenn done)
  Assignee-Name (wenn vorhanden, 12px grau)
  Due-Date (wenn vorhanden, 12px, rot wenn überfällig)
  onClick auf Card: öffnet Task-Detail-Panel (TODO Phase 4)

─────────────────────────────────────────
Integration in MessageBubble.tsx:
─────────────────────────────────────────
Ersetze den reinen Text-Render durch:

if (message.message_type === 'system') return <MessageCard message={message} />
if (['order_link','product_link','alert','task'].includes(message.message_type)) {
  return (
    <div>
      <MessageCard message={message} />
      {message.content && <p>{message.content}</p>}  // optionaler Text unter der Card
    </div>
  )
}
// Fallback: normaler Text-Render (bestehender Code)
```

---

## Prompt 7 — Task-aus-Nachricht + Task API

**Abhängigkeit:** nach Prompt 1, 2, 6

**Neue Dateien:**
- `app/api/chat/tasks/route.ts`
- `app/api/chat/tasks/[id]/route.ts`
- `components/chat/CreateTaskModal.tsx`

```
Implementiere "In Aufgabe umwandeln" Feature.

─────────────────────────────────────────
1. app/api/chat/tasks/route.ts
─────────────────────────────────────────
GET ?conversation_id=UUID&status=open|in_progress|done
- requireTenant(), createServiceClient()
- SELECT chat_tasks WHERE tenant_id=? AND conversation_id=? (optional)
- JOIN: assignee (profiles: full_name, avatar_url)
- ORDER BY due_date ASC NULLS LAST, created_at DESC
- Return: ChatTask[]

POST body: CreateTaskSchema
- INSERT chat_tasks (title, description, assignee_id, due_date, message_id, conversation_id,
  created_by=currentUser, tenant_id)
- Wenn message_id gesetzt:
  UPDATE chat_messages SET metadata = metadata || '{"task_id": "{new_task_id}"}'::jsonb,
    message_type='task'
  WHERE id=message_id AND tenant_id=?
- Return: ChatTask

─────────────────────────────────────────
2. app/api/chat/tasks/[id]/route.ts
─────────────────────────────────────────
PATCH body: { status?, assignee_id?, due_date?, title? }
- Prüfe: created_by=currentUser ODER assignee_id=currentUser → sonst 403
- UPDATE chat_tasks SET ... WHERE id=? AND tenant_id=?
- Wenn status geändert: UPDATE chat_messages SET metadata = metadata || '{"status":...}'
  WHERE metadata->>'task_id' = id (synchron halten)
- Return: aktualisierter ChatTask

DELETE
- Nur created_by=currentUser → sonst 403
- DELETE chat_tasks WHERE id=? AND tenant_id=?
- UPDATE chat_messages SET message_type='text', metadata='{}' WHERE metadata->>'task_id'=id
- Return: { ok: true }

─────────────────────────────────────────
3. components/chat/CreateTaskModal.tsx
─────────────────────────────────────────
'use client'
Props:
  message: ChatMessage  // die Nachricht aus der die Aufgabe wird
  conversationId: string
  onClose: () => void
  onCreated: (task: ChatTask) => void

Modal (340px breit, zentriert):
  Titel: "Aufgabe erstellen"
  Felder:
    - title: Input, vorbelegt mit message.content (max 200 Zeichen)
    - description: Textarea (optional), vorbelegt mit leerem String
    - assignee: User-Picker Dropdown (GET /api/team/users, Autocomplete)
    - due_date: date Input (type="date", min=heute)
  Submit: POST /api/chat/tasks → onCreated(task) → onClose()
  Cancel: onClose()

Integration in MessageContextMenu.tsx:
- Füge Item "In Aufgabe umwandeln" hinzu (nur für message_type='text')
- onClick: setShowCreateTask(true)
- Render <CreateTaskModal> wenn showCreateTask
```

---

## Prompt 8 — Alert-Regeln UI + Smoke Test

**Abhängigkeit:** nach Prompt 3–7

**Neue Datei:** `components/chat/AlertRulesPanel.tsx`

```
Implementiere Alert-Konfiguration UI und abschließenden Smoke Test.

─────────────────────────────────────────
1. app/api/chat/alert-rules/route.ts (neue Route)
─────────────────────────────────────────
GET ?conversation_id=UUID
- SELECT chat_alert_rules WHERE tenant_id=? AND conversation_id=? ORDER BY trigger_type
- Return: AlertRule[]

POST body: UpsertAlertRuleSchema
- UPSERT chat_alert_rules ON CONFLICT (tenant_id, conversation_id, trigger_type)
  DO UPDATE SET is_active=EXCLUDED.is_active, config=EXCLUDED.config
- Nur admin/owner der Conversation darf das → prüfe chat_conversation_members
- Return: AlertRule

─────────────────────────────────────────
2. components/chat/AlertRulesPanel.tsx
─────────────────────────────────────────
'use client'
Props:
  conversationId: string
  myRole: 'member' | 'admin' | 'owner'

Nur rendern wenn myRole='admin'|'owner'. Sonst null.

Layout (Abschnitt in ChannelMembersDrawer oder eigener Tab im Channel-Header-Menü):

Titel: "Alert-Benachrichtigungen"
Untertitel: "Automatische Nachrichten in diesem Kanal"

Liste aller trigger_types als Toggle-Rows:

  order_new         "Neue Bestellungen"          🛒
  order_cancelled   "Stornierungen"              ❌
  order_shipped     "Versandbestätigungen"        📦
  return_new        "Neue Retouren"              ↩
  return_approved   "Genehmigte Retouren"         ✅
  bullmq_failure    "Job-Fehler (Versuche)"       ⚠️
  bullmq_dead_letter "Jobs in Dead Letter Queue"  🔴
  marketplace_error  "Marktplatz-API Fehler"      🌐
  low_stock         "Niedriger Lagerbestand"      📉
  daily_kpi         "Täglicher KPI-Bericht"       📊

Jede Row: Icon + Label + Beschreibung (12px grau) + Toggle-Switch rechts
Toggle-State: is_active aus geladenen AlertRules
Kein Rule vorhanden → is_active=false (default)

onChange: POST /api/chat/alert-rules { conversation_id, trigger_type, is_active: !current }

Für daily_kpi: zeige zusätzlich Time-Input (HH:MM) wenn aktiviert
  → config: { send_at: "07:00", timezone: "Europe/Berlin" }

─────────────────────────────────────────
3. Integration
─────────────────────────────────────────
Füge "Alert-Regeln" als eigenen Menüpunkt im Channel-Header-Dropdown hinzu
(neben "Mitglieder anzeigen").
Nur sichtbar wenn conversation.type='channel'.
onClick: setShowAlertRules(true)
Render als Drawer (rechts, 360px) oder Modal.

─────────────────────────────────────────
4. Smoke Test
─────────────────────────────────────────
npx tsc --noEmit

Häufige Fehler Phase 3:
- alertXxx Funktionen importiert aber nicht awaited → TypeScript warnt nicht,
  aber Runtime-Fehler möglich. Alle Calls mit .catch() absichern.
- metadata JSONB union type: ChatMessage.metadata kann verschiedene Shapes haben.
  Lösung: (message.metadata as OrderLinkMetadata) mit Type Guard oder satisfies
- message_type fehlt im bestehenden ChatMessage Interface aus Phase 1 →
  muss nachträglich ergänzt werden: message_type: MessageType DEFAULT 'text'
- CreateTaskModal: assignee User-Picker braucht den bestehenden /api/team/users Endpoint.
  Falls nicht vorhanden: einfaches Text-Input als Fallback.

Nach 0 Fehlern: docker compose up --build -d in /opt/comentra-amazon
```
