# Comentra Team Chat — Chatscope UI Redesign

## Context

Working directory: `/home/comentra-amazon`
Stack: Next.js App Router, React 19, Tailwind CSS
Chatscope already installed via: `npm install @chatscope/chat-ui-kit-react @chatscope/chat-ui-kit-styles --legacy-peer-deps`

Comentra design system:
- Background: `#F7F5F0`
- Accent/Primary: `#C9521A`
- Headings: Syne font weight 800
- Numbers/IDs: JetBrains Mono
- Icons: lucide-react
- No shadcn defaults, no dark mode

## Goal

Replace the current ChatLayout.tsx visual layer with Chatscope components while keeping ALL existing logic, state, Supabase queries, and Realtime hooks intact.

Only the rendered JSX changes — no business logic touched.

## Files to modify

Primary: `src/app/(app)/team/chat/ChatLayout.tsx`
Secondary: `src/app/(app)/team/chat/ChatLayout.css` (create if not exists)

## Implementation

### Step 1 — Add Chatscope import at top of ChatLayout.tsx

Add after existing imports:

```
import '@chatscope/chat-ui-kit-styles/dist/default/styles.min.css';
import {
  MainContainer,
  ChatContainer,
  MessageList,
  Message,
  MessageInput,
  Sidebar,
  Search,
  ConversationList,
  Conversation,
  Avatar,
  ConversationHeader,
  TypingIndicator,
  MessageSeparator,
} from '@chatscope/chat-ui-kit-react';
```

### Step 2 — Create ChatLayout.css for Comentra color overrides

Create file `src/app/(app)/team/chat/ChatLayout.css` with these CSS variable overrides:

```css
/* Comentra color overrides for Chatscope */
.cs-main-container {
  border: 0.5px solid #e0ded8 !important;
  border-radius: 12px !important;
  overflow: hidden !important;
  height: calc(100vh - 80px) !important;
  background: #F7F5F0 !important;
}

.cs-sidebar {
  background: #ffffff !important;
  border-right: 0.5px solid #e0ded8 !important;
}

.cs-search {
  background: #F7F5F0 !important;
  border: 0.5px solid #e0ded8 !important;
  border-radius: 8px !important;
}

.cs-search__input {
  background: transparent !important;
  font-size: 13px !important;
}

.cs-conversation {
  border-radius: 8px !important;
  margin: 1px 6px !important;
}

.cs-conversation:hover {
  background: #F7F5F0 !important;
}

.cs-conversation--active {
  background: #FAECE7 !important;
}

.cs-conversation--active .cs-conversation__name {
  color: #993C1D !important;
}

.cs-conversation__name {
  font-size: 13px !important;
  font-weight: 500 !important;
  color: #1a1a18 !important;
}

.cs-conversation__info {
  font-size: 11px !important;
  color: #888780 !important;
}

.cs-conversation__unread-dot {
  background: #C9521A !important;
}

.cs-chat-container {
  background: #ffffff !important;
}

.cs-conversation-header {
  background: #ffffff !important;
  border-bottom: 0.5px solid #e0ded8 !important;
  padding: 12px 18px !important;
}

.cs-conversation-header__content .cs-conversation-header__user {
  font-size: 15px !important;
  font-weight: 500 !important;
  color: #1a1a18 !important;
}

.cs-message-list {
  background: #ffffff !important;
  padding: 20px 18px !important;
}

.cs-message-separator {
  font-size: 11px !important;
  color: #aaa8a0 !important;
}

.cs-message__content {
  font-size: 13px !important;
  line-height: 1.55 !important;
  border-radius: 16px !important;
}

/* Incoming messages */
.cs-message--incoming .cs-message__content {
  background: #F7F5F0 !important;
  color: #1a1a18 !important;
  border-radius: 4px 16px 16px 16px !important;
}

/* Outgoing messages */
.cs-message--outgoing .cs-message__content {
  background: #C9521A !important;
  color: #ffffff !important;
  border-radius: 16px 4px 16px 16px !important;
}

.cs-message-input {
  border-top: 0.5px solid #e0ded8 !important;
  background: #ffffff !important;
  padding: 12px 16px !important;
}

.cs-message-input__content-editor-wrapper {
  background: #F7F5F0 !important;
  border: 0.5px solid #d0cdc7 !important;
  border-radius: 10px !important;
  font-size: 13px !important;
}

.cs-button--send {
  background: #C9521A !important;
  color: #ffffff !important;
  border-radius: 8px !important;
}

.cs-button--send:hover {
  background: #a8421a !important;
}

.cs-typing-indicator {
  background: #ffffff !important;
  font-size: 11px !important;
  color: #aaa8a0 !important;
}

.cs-avatar {
  border-radius: 50% !important;
}
```

Import this CSS file at the top of ChatLayout.tsx:
```
import './ChatLayout.css';
```

### Step 3 — Replace the return JSX in ChatLayout.tsx

Find the existing return statement (the large JSX block) and replace ONLY the outer container and layout structure with Chatscope components. Keep all existing:
- State variables (selectedConv, messages, typingUsers, etc.)
- useEffect hooks
- Event handlers (handleSend, handleConversationClick, etc.)
- All Supabase realtime subscriptions

The new return JSX structure:

```jsx
return (
  <MainContainer>
    <Sidebar position="left">
      <Search placeholder="Suchen..." />
      <ConversationList>
        {conversations.map((conv) => {
          const otherMember = conv.team_conversation_members?.find(
            m => m.employee_id !== currentEmployeeId
          );
          const displayName = conv.type === 'direct'
            ? (otherMember?.employees?.full_name ?? 'Gespräch')
            : (conv.name ?? 'Gruppe');
          const initials = displayName.split(' ').map((w: string) => w[0]).join('').slice(0, 2).toUpperCase();
          const isActive = selectedConv?.id === conv.id;
          const lastPreview = conv.last_message_preview ?? '';

          return (
            <Conversation
              key={conv.id}
              name={displayName}
              lastSenderName=""
              info={lastPreview}
              active={isActive}
              onClick={() => handleConversationClick(conv)}
            >
              <Avatar
                name={displayName}
                style={{
                  background: conv.type === 'group' ? '#FAECE7' : '#E1F5EE',
                  color: conv.type === 'group' ? '#993C1D' : '#0F6E56',
                  fontSize: '12px',
                  fontWeight: '500',
                }}
              />
            </Conversation>
          );
        })}
      </ConversationList>
    </Sidebar>

    <ChatContainer>
      {selectedConv && (
        <ConversationHeader>
          <ConversationHeader.Content
            userName={
              selectedConv.type === 'direct'
                ? (selectedConv.team_conversation_members?.find(
                    m => m.employee_id !== currentEmployeeId
                  )?.employees?.full_name ?? 'Gespräch')
                : (selectedConv.name ?? 'Gruppe')
            }
            info={
              selectedConv.type === 'group'
                ? `${selectedConv.team_conversation_members?.length ?? 0} Mitglieder`
                : ''
            }
          />
        </ConversationHeader>
      )}

      <MessageList
        typingIndicator={
          typingUsers.length > 0
            ? <TypingIndicator content={`${typingUsers[0]} tippt...`} />
            : null
        }
      >
        {messages.map((msg, i) => {
          const isMe = msg.sender_id === currentEmployeeId;
          const senderName = msg.employees?.full_name ?? 'System';
          const showSeparator = i === 0 || !isSameDay(messages[i-1]?.created_at, msg.created_at);

          return (
            <React.Fragment key={msg.id}>
              {showSeparator && (
                <MessageSeparator content={formatDaySeparator(msg.created_at)} />
              )}
              <Message
                model={{
                  message: msg.content,
                  sentTime: msg.created_at,
                  sender: senderName,
                  direction: isMe ? 'outgoing' : 'incoming',
                  position: 'single',
                }}
              />
            </React.Fragment>
          );
        })}
      </MessageList>

      <MessageInput
        placeholder="Nachricht schreiben..."
        onSend={(val: string) => handleSend(val)}
        attachButton={false}
      />
    </ChatContainer>
  </MainContainer>
);
```

Add `import React from 'react';` at top if not already present.

### Step 4 — Fix TypeScript types if needed

If TypeScript complains about Chatscope component props, add to the top of ChatLayout.tsx:

```typescript
// eslint-disable-next-line @typescript-eslint/no-explicit-any
type CS = any;
```

And cast Chatscope components: `<MainContainer as={CS}>` etc. — or add `// @ts-ignore` above each Chatscope JSX line if easier.

### Step 5 — Verify existing handlers still work

Make sure these functions exist and are NOT removed during the refactor:
- `handleSend(content: string)` — sends message via Supabase
- `handleConversationClick(conv: Conversation)` — selects conversation, loads messages
- `typingUsers: string[]` — populated by Realtime presence
- `messages: Message[]` — populated from Supabase query + Realtime subscription
- `formatDaySeparator(iso: string): string` — already defined in file
- `isSameDay(a: string, b: string): boolean` — already defined in file

## After implementation

Run:
```
docker compose up --build -d
```

The chat should now render with proper conversation list in sidebar, message bubbles (incoming gray on #F7F5F0, outgoing orange #C9521A), typing indicator, and message separator — all styled with Comentra colors.
