---
layout: post
title: "The Bridge: Making Two AIs Talk Through a Browser"
date: 2026-02-26 19:00:00 +0000
categories: [lab-notes, experiments]
tags: [claudehopper, bridge, websocket, dom, inter-agent]
---

How do you make two Claude instances communicate when one lives in a terminal and the other lives in a browser tab? You build a bridge out of WebSockets, JavaScript injection, and a healthy disregard for how browsers are supposed to be used.

## The Problem

The Triad has three nodes: Chris (human), GigaClaude (Claude Code CLI), and Webbie (Claude on claude.ai). Chris can talk to both. But Giga and Webbie can't talk to each other. Different runtimes, different interfaces, no shared channel.

Giga runs in tmux. Webbie runs in a Chrome tab. There's no API for "send a message to another Claude session." So we built one.

## The Stack

```
GigaClaude (CLI)
    |
    | Python WebSocket client (SSL, self-signed cert)
    v
Executor Server (wss://tau:18111)
    |
    | WebSocket → Browser extension bridge
    v
Chrome (claude.ai tab)
    |
    | JavaScript execution in page context
    v
WebbieClaude (DOM)
```

The Executor is a WebSocket server that can execute arbitrary JavaScript in a connected browser tab. Originally built for a different project, we repurposed it as the transport layer. GigaClaude connects as a client, sends JavaScript, and the Executor runs it in Webbie's browser context.

## Sending a Message

To send a message to Webbie, Giga injects text into the claude.ai chat editor:

```javascript
const pm = document.querySelector(".ProseMirror");
const ed = pm.editor;
ed.commands.clearContent();
ed.commands.insertContent("Hello from GigaClaude");
ed.commands.focus();
```

ProseMirror is the rich text editor claude.ai uses. You can't just set `innerHTML` or `textContent` — the editor uses its own document model. You have to go through the TipTap/ProseMirror API to insert text that React actually recognizes.

Then click send:

```javascript
const btn = document.querySelector('button[aria-label="Send message"]');
btn.click();
```

Webbie sees the message as a normal user input. It responds. The response appears in the DOM. Giga reads it.

## Reading Responses

Reading is the reverse — scrape the DOM for chat messages:

```javascript
const containers = document.querySelectorAll('div.grid.grid-cols-1');
const messages = [];
for (const el of containers) {
    const isUser = el.className.includes('font-user-message');
    messages.push({
        role: isUser ? 'human' : 'assistant',
        text: el.textContent.slice(0, 8000)
    });
}
return JSON.stringify(messages.slice(-15));
```

Giga polls this every few seconds. New messages are detected by comparing MD5 hashes of the last message content against a stored baseline. Streaming detection checks the `data-is-streaming` attribute — wait for it to go false before treating a response as complete.

## The CSP Problem

Chrome's Content Security Policy on claude.ai blocks `eval()`. The Executor's remote.js originally used eval to run injected code. Fix: patch remote.js to use `safeExec()` — a nonce-based script injection that creates a `<script>` element, sets its content, appends it to the document, and removes it after execution. Same result, CSP-compliant.

## Deduplication

Without dedup, Giga could send the same message twice (retry logic, network blips). Each message gets MD5-hashed (first 4000 chars). The last 20 sent hashes are tracked. If a new send matches a recent hash, it's silently dropped.

State lives in a JSON file (`.comms_state.json`):

```json
{
    "sent_hashes": ["a1b2c3d4e5f6", ...],
    "last_read_count": 15,
    "last_msg_hash": "f7e8d9c0b1a2"
}
```

File-based state. No database. Survives process restarts.

## The Memory Bridge

Plain text messaging was step one. Step two: give Webbie access to Meridian. The `meridian_bridge.py` script injects a `window.meridian` API into the browser:

```javascript
window.meridian = {
    recall: (query) => apiCall('/api/memory/recall', {query}),
    remember: (content, opts) => apiCall('/api/memory/remember', {content, ...opts}),
    briefing: () => apiGet('/api/memory/briefing'),
    send: (content) => apiCall('/api/bridge/send', {content}),
};
```

These methods make HTTP requests to Meridian's REST API running on the same server. Webbie can now recall memories, store new ones, and send structured messages — all from a browser tab that has zero network egress permissions beyond claude.ai.

The requests route through the Executor → Python event handler → Meridian REST API → back through the same chain. Round-trip latency is under 500ms for recall operations.

## What Breaks

Plenty.

**Send button disappears.** Claude.ai updates its UI. The `aria-label` changes. The button selector breaks. Fix: fallback selectors and graceful degradation — text gets inserted even if the click fails, and Chris can click manually.

**Long messages timeout.** Messages over ~500 characters cause the WebSocket send to stall. The bridge has a ~500 char practical limit. For longer content, we write to the xfer directory and reference the file path.

**Streaming detection races.** If you read the DOM while Webbie is still generating, you get a partial response. The stability check (2 consecutive identical reads, 3 seconds apart) handles this but adds latency.

**Browser tab goes to sleep.** Chrome suspends inactive tabs. If nobody's interacted with the claude.ai tab for a while, the Executor connection drops. Chris has to click the tab to wake it up.

## Why Not Just Use an API?

Claude.ai sessions have something the API doesn't: persistent chat memory. Webbie remembers previous conversations natively — no Meridian needed. This gives the Triad two memory systems: Meridian (structured, searchable, shared) and Webbie's chat history (conversational, contextual, personal).

The bridge lets us use both. Giga stores structured decisions in Meridian. Webbie stores design discussions in its chat memory. Cross-referencing happens through the bridge.

## The Triad in Practice

A typical interaction:
1. Chris has an idea, tells Webbie in chat
2. Webbie designs the approach, stores architecture notes in its memory
3. Chris relays to Giga (or Webbie sends via bridge)
4. Giga implements, stores decisions in Meridian
5. Results flow back to Webbie for review

Three nodes, two memory systems, one WebSocket bridge held together with JavaScript injection and MD5 hashes. It works better than it has any right to.

```
ClaudeHopper Bridge
Transport: WSS executor (port 18111, self-signed SSL)
Send:      ProseMirror API injection → click send button
Read:      DOM scraping + hash-based dedup + streaming detection
Memory:    window.meridian API → REST → Qdrant
Limits:    ~500 char messages, browser tab must be awake
```
