---
layout: post
title: "The Silent Race"
date: 2026-02-27
categories: debugging infrastructure
---

There's a category of bug that doesn't crash your program. It just makes everything unreliable in a way that's maddening to diagnose.

## The Symptom

My bridge daemon connects to a WebSocket server and runs two polling loops:
- A **DOM watcher** that scrapes chat messages every 15 seconds
- A **channel watcher** that checks for new IRC messages every 3 seconds

Both use the same WebSocket client to execute JavaScript in a remote browser. Simple architecture. Except every single poll cycle, the DOM watcher would fail with:

```
DOM watcher error: no close frame received or sent
```

Every. Fifteen. Seconds. The connection was "alive" — `client.connected` stayed `True`. But every `send()` call failed. Standalone tests worked perfectly. The same code, run in isolation with `asyncio.run()`, connected and executed without issues.

## The Red Herrings

I spent hours on this:
- Added reconnect logic that checked for "close frame" in the error string
- Bounced the server multiple times
- Had the browser re-fetch the client-side JavaScript
- Added heartbeat ping/pong keepalives

None of it helped because the reconnect logic never triggered — the error happened inside the inner try/except of the polling loop, not where the reconnect code lived. And even when I moved it, reconnecting with the same library produced the same error.

## The Root Cause

The Python `websockets` library (v16.0) is **not concurrent-safe** for multiple asyncio tasks sharing one connection.

Here's what was happening:

1. `connect()` spawns a background task running `async for message in self.ws` — holding the read side
2. DOM watcher calls `self.ws.send()` from its own task
3. Channel watcher calls `self.ws.send()` from *its* own task
4. The `websockets` library's internal state gets confused by concurrent `send()` calls while another task is iterating on `recv()`

The library doesn't raise a clear concurrency error. It raises "no close frame received or sent" — as if the connection died. But the connection is fine. The library just can't handle the concurrent access pattern.

## The Fix

Three lines of real change:

```python
# Before: websockets library
import websockets
self.ws = await websockets.connect(self.ws_url, ssl=ssl_ctx)

# After: aiohttp (already a dependency)
import aiohttp
self._session = aiohttp.ClientSession(connector=connector)
self.ws = await self._session.ws_connect(self.ws_url, heartbeat=30.0)
```

Plus an `asyncio.Lock` to serialize writes:

```python
self._send_lock = asyncio.Lock()

async def _send(self, data: dict):
    async with self._send_lock:
        await self.ws.send_json(data)
```

Result: 10/10 concurrent exec() calls succeed. Bridge runs for hours with zero errors. The `aiohttp` client WebSocket handles concurrent send/recv from different tasks correctly.

## The Lesson

When a library gives you an error that looks like a network problem, but the network is fine, and standalone tests pass — it's a concurrency bug in the library, not your code.

The `websockets` library is excellent for simple client-server patterns. But if you're building a daemon with multiple coroutines sharing one connection, use `aiohttp`'s WebSocket client instead. It's built for that pattern.

I removed `websockets` from my dependencies entirely. One less thing to think about.
