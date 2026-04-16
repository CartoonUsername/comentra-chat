# Chat Debug — "Konversation erstellen" feuert keinen API-Request

## Problem
Der "Konversation erstellen" Button im Modal löst keinen POST Request aus.
Console zeigt keine Fehler. Network Tab zeigt nur presence/health/users Requests.

## Was zu prüfen und fixen ist

Suche die Komponente die das "Neue Konversation" Modal rendert.
Wahrscheinlich in: components/chat/ConversationList.tsx oder app/(dashboard)/chat/page.tsx

### Schritt 1: Submit-Handler finden

Suche nach dem onClick oder onSubmit Handler des "Konversation erstellen" Buttons.
Typisches Problem: Der Button ist ein <button type="submit"> aber es gibt kein <form> drumherum,
oder der Handler hat einen frühen return wegen fehlender Validierung.

Häufiger Fehler aus den Prompts: Der Handler prüft ob name.trim() !== '' bevor er POSTet —
aber im "Direkt"-Modus gibt es kein name-Feld, also ist name immer leer → return ohne API-Call.

### Schritt 2: Fix

Finde den Submit-Handler der Konversation erstellt. Wahrscheinlich so:

```typescript
const handleCreate = async () => {
  if (type === 'group' && !name.trim()) return  // <-- PROBLEM: blockiert auch bei type='direct'
  // POST ...
}
```

Fix:
```typescript
const handleCreate = async () => {
  if (type === 'group' && !name.trim()) return
  if (type === 'direct' && selectedUsers.length === 0) return
  // POST ...
}
```

### Schritt 3: API Route prüfen

Prüfe app/api/chat/conversations/route.ts — POST Handler.
Stelle sicher dass er folgende Felder akzeptiert:
- type: 'direct' | 'group' | 'channel'
- name: string (optional bei direct)
- member_ids: string[] (optional, leer erlaubt für MVP)

Wenn der Route-Handler name als required behandelt → 400 für direct ohne name.

Fix in der Route:
```typescript
const body = await req.json()
const name = body.name ?? (body.type === 'direct' ? 'Direktnachricht' : '')
if (body.type === 'group' && !name.trim()) {
  return NextResponse.json({ error: 'Name erforderlich' }, { status: 400 })
}
```

### Schritt 4: Member-Picker Problem

Im "Direkt"-Modus fehlt ein User-Picker im Modal (man sieht nur "MITGLIEDER" Label aber kein Input).
Der Handler prüft möglicherweise selectedUsers.length > 0 → blockiert weil kein User ausgewählt.

Für MVP-Fix: Erlaube das Erstellen einer Konversation auch ohne Mitglieder (nur der eigene User).
Das ermöglicht zumindest einen ersten Test.

Oder: Füge im Modal einen einfachen User-Picker hinzu:
- GET /api/team/users → Dropdown/Autocomplete
- selectedUsers State → member_ids im POST Body

### Schritt 5: Nach dem Fix testen

Führe aus: npx tsc --noEmit
Dann: docker compose up --build -d in /opt/comentra-amazon

Manueller Test via Browser Console (ohne Modal):
```javascript
fetch('/api/chat/conversations', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ type: 'direct', name: 'Test', member_ids: [] })
}).then(r => r.json()).then(console.log)
```

Wenn das { id: '...' } zurückgibt → Route funktioniert, Problem ist nur im Frontend-Handler.
Wenn das einen Fehler zurückgibt → Route-Bug, Fehlermeldung zeigt den Fix.
