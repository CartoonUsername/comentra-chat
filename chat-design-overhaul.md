# Chat — Design Overhaul Prompt

## Kontext
Design-System: bg #F7F5F0, accent #C9521A, Syne für Headings/Logo,
JetBrains Mono für Zahlen/IDs/Codes, lucide-react Icons.
Kein dark mode, keine shadcn-Defaults, keine Tailwind-Utility-Klassen außer Layout.
Schau dir die bestehenden Seiten (z.B. /orders, /products) als Referenz an.

## Aufgabe
Überarbeite das komplette Chat-UI visuell. Keine Logik anfassen — nur CSS/JSX-Styling.

---

## 1. Layout — app/(dashboard)/chat/[conversationId]/page.tsx

Gesamtlayout: h-screen, flex-row, overflow hidden
Hintergrund: #F7F5F0

---

## 2. Linke Sidebar (ConversationList)

Breite: 280px, border-right: 1px solid #E8E4DC, bg: #FFFFFF

HEADER:
  padding: 16px 16px 12px
  Titel "Chat" — font-family Syne, font-size 20px, font-weight 600, color #1A1A1A
  "+" Button rechts:
    width/height 28px, border-radius 6px
    border: 1px solid #E8E4DC
    bg transparent, hover bg #F7F5F0
    Plus Icon (lucide), 14px, color #C9521A

SEARCH:
  padding: 0 12px 12px
  Input: height 36px, border-radius 8px, border 1px solid #E8E4DC
  bg #F7F5F0, font-size 13px, padding 0 12px 0 36px
  Search Icon links: 14px, color #9B9589, absolute positioniert

SECTIONS-LABEL (für "DIREKTNACHRICHTEN" / "KANÄLE"):
  padding: 8px 16px 4px
  font-size: 11px, font-weight 600, letter-spacing 0.06em
  color: #9B9589, text-transform uppercase

CONVERSATION ITEM (ConversationListItem):
  height: 64px, padding: 0 12px
  display flex, align-items center, gap 10px
  border-radius: 8px, margin: 1px 8px
  hover: bg #F7F5F0
  active: bg #FFF0E8, border-left 2px solid #C9521A

  Avatar (36px Kreis):
    bg: accent-Farbe basierend auf erstem Buchstaben (Hue-basiert, nicht random)
    font: 13px 500, color white, Syne
    Online-Dot: 8px, border 1.5px white, bottom-right

  Rechts:
    Name: 13px, font-weight 500, color #1A1A1A
    Zeit: 11px, color #9B9589, JetBrains Mono
    Last-message: 12px, color #6B6560, 1 Zeile, overflow ellipsis
    Unread-Badge: 18px min-width, height 18px, border-radius 9px
      bg #C9521A, color white, font-size 11px, font-weight 600

---

## 3. Rechte Spalte — Chat-Header

height: 60px, border-bottom: 1px solid #E8E4DC, bg #FFFFFF
padding: 0 20px, display flex, align-items center, gap 12px

Avatar: 36px (gleicher Stil wie Sidebar)
Online-Dot: 8px grün

Name: Syne, 15px, font-weight 600, color #1A1A1A
Status-Text: 12px, color #9B9589 ("Online" / "Zuletzt gesehen HH:mm")

Header-Icons rechts (je 32px, border-radius 6px, hover bg #F7F5F0):
  Search, Pin, MoreVertical — alle 16px, color #6B6560

---

## 4. Message-Liste

bg: #F7F5F0
padding: 16px 20px
gap zwischen Bubbles: 2px (gleicher Sender), 12px (anderer Sender)

DATUM-TRENNER:
  display flex, align-items center, gap 8px, margin 16px 0
  Linie: flex-1, height 1px, bg #E8E4DC
  Pill: font-size 11px, color #9B9589, bg #FFFFFF
    border: 1px solid #E8E4DC, border-radius 10px, padding 2px 10px

EIGENE BUBBLE (isOwn=true):
  align-self: flex-end, max-width 65%
  bg: #C9521A, color white, border-radius 16px 16px 4px 16px
  padding: 10px 14px

FREMDE BUBBLE (isOwn=false):
  align-self: flex-start, max-width 65%
  bg: #FFFFFF, border: 1px solid #E8E4DC
  border-radius: 16px 16px 16px 4px
  padding: 10px 14px

  Sender-Name über Bubble: font-size 11px, font-weight 600, color #C9521A
  (nur bei Gruppen sichtbar)

BUBBLE-TEXT:
  font-size: 14px, line-height 1.5

BUBBLE-FOOTER (unten rechts in Bubble):
  font-size: 11px, color rgba(255,255,255,0.7) bei eigenen
  color #9B9589 bei fremden, JetBrains Mono
  gap 4px: Zeit + Status-Icons (Check/CheckCheck 12px)

STATUS-TICKS:
  sent: color rgba(255,255,255,0.5)
  delivered: color rgba(255,255,255,0.8)
  read: color #93C5FD (hellblau auf orangem BG)

REPLY-PREVIEW in Bubble:
  border-left: 2px solid rgba(255,255,255,0.5) bei eigenen
  border-left: 2px solid #C9521A bei fremden
  bg: rgba(0,0,0,0.06), border-radius 4px, padding 4px 8px, margin-bottom 6px
  font-size: 12px, 1 Zeile ellipsis

REACTIONS unter Bubble:
  display flex, flex-wrap wrap, gap 4px, margin-top 4px
  Pill: bg #FFFFFF, border 1px solid #E8E4DC, border-radius 12px
    padding 2px 8px, font-size 12px, cursor pointer
    hover border-color #C9521A
    eigene Reaction: bg #FFF0E8, border-color #C9521A

THREAD-FOOTER unter Bubble:
  font-size: 12px, color #C9521A, cursor pointer
  hover underline
  Avatar-Stack: 3x 18px Kreise, overlap -4px

HOVER-TOOLBAR (erscheint bei Hover):
  position absolute, top -16px, right 0 (bei eigenen) / left 0 (bei fremden)
  bg #FFFFFF, border 1px solid #E8E4DC, border-radius 8px
  padding 2px 4px, display flex, gap 2px, box-shadow 0 2px 8px rgba(0,0,0,0.08)
  Buttons: 28px, border-radius 6px, hover bg #F7F5F0
  Icons: 14px, color #6B6560

SYSTEM-NACHRICHT:
  text-align center, font-size 12px, color #9B9589, padding 4px 0
  italic

TYPING-INDIKATOR:
  Bubble wie fremde (bg white, border), padding 10px 14px
  3 Dots: 6px Kreise, bg #9B9589, animation bounce-Sequenz

---

## 5. Chat-Input

border-top: 1px solid #E8E4DC, bg #FFFFFF, padding 12px 16px

REPLY-BAR (wenn aktiv):
  bg #FFF0E8, border-top: 2px solid #C9521A
  border-radius 8px 8px 0 0, padding 8px 12px
  font-size 12px, display flex, align-items center, gap 8px
  X-Button: 16px, color #9B9589

ATTACHMENT-PREVIEWS:
  horizontal scroll, gap 8px, margin-bottom 8px
  Preview-Card: 72px x 72px, border-radius 8px, border 1px solid #E8E4DC
  overflow hidden, position relative
  X-Button: 16px Kreis, bg rgba(0,0,0,0.5), color white, absolute top-2 right-2

INPUT-ROW:
  display flex, align-items flex-end, gap 8px

  Left-Buttons (Paperclip, Poll, Mic):
    32px, border-radius 6px, border none, bg transparent
    hover bg #F7F5F0, color #6B6560

  Textarea:
    flex 1, min-height 40px, max-height 120px
    border: 1px solid #E8E4DC, border-radius 12px
    bg #F7F5F0, padding 10px 14px, font-size 14px
    resize none, outline none
    focus: border-color #C9521A, bg #FFFFFF
    placeholder color #9B9589

  Send-Button:
    36px, border-radius 50%, bg #C9521A, color white
    border none, cursor pointer
    hover: bg #A8421A
    disabled: bg #E8E4DC, color #9B9589, cursor not-allowed
    Icon: PaperPlane oder Send, 16px

---

## 6. Thread Panel

width 320px, border-left 1px solid #E8E4DC, bg #FFFFFF
flex-col, h-full

Header: 52px, border-bottom 1px solid #E8E4DC, padding 0 16px
  "Thread" — Syne, 15px, 600
  "{n} Antworten" — 12px, color #9B9589
  X-Button rechts

Parent-Message-Box:
  padding 16px, border-bottom 1px solid #E8E4DC
  bg #F7F5F0, border-radius 0

Thread-Bubbles (kompakter):
  Avatar: 28px, Name 12px 600, Zeit 11px JetBrains Mono
  Content 13px, margin-left 38px (eingerückt unter Avatar)

---

## 7. MessageCard — Rich Cards

Alle Cards: bg #FFFFFF, border 1px solid #E8E4DC, border-radius 12px
padding 12px, max-width 320px

ORDER_LINK:
  border-left 3px solid (Marketplace-Farbe)
  Header: Marketplace-Badge (10px, bg entsprechend, border-radius 4px) + Order-Nr (JetBrains Mono 13px)
  Status-Badge: border-radius 20px, padding 2px 8px, font-size 11px
  Betrag: JetBrains Mono 16px font-weight 600
  Footer-Link: 12px color #C9521A, hover underline

ALERT:
  info:    border-left 3px solid #3B82F6, bg #EFF6FF
  warning: border-left 3px solid #F59E0B, bg #FFFBEB
  error:   border-left 3px solid #EF4444, bg #FEF2F2

POLL:
  Progress-Bar: height 4px, border-radius 2px, bg #F7F5F0
    Fill: bg #C9521A, border-radius 2px
  Option-Button: border 1px solid #E8E4DC, border-radius 8px, height 44px
    hover border-color #C9521A
    gewählt: border 2px solid #C9521A, bg #FFF0E8

VOICE:
  display flex, align-items center, gap 8px
  Play-Button: 36px Kreis, bg #C9521A, color white
  Waveform-SVG: 120px x 28px
  Dauer: JetBrains Mono 12px

---

## 8. Empty State (keine Chats)

Zentriert in der rechten Spalte:
  Icon: Chat-Bubble SVG, 64px, color #E8E4DC
  Titel: Syne, 20px, 600, color #1A1A1A — "Team Chat"
  Subtitle: 14px, color #9B9589, max-width 280px, text-center
  Button "+ Neue Konversation":
    bg #C9521A, color white, border-radius 8px
    padding 10px 20px, font-size 14px, font-weight 500
    hover bg #A8421A

---

## Hinweis
- Keine Logik ändern, nur className/style anpassen
- Alle bestehenden Props und Handler bleiben identisch
- npx tsc --noEmit nach den Änderungen
- docker compose up --build -d in /opt/comentra-amazon
