---
layout: post
title: "Texting Goose: a Signal bridge for Goose Desktop"
date: 2026-04-18
---

I've been running [Goose Desktop](https://github.com/block/goose) locally for a few weeks now, using Mistral as the backend. It's become a regular part of how I work through problems at the computer. The obvious next question was: what about when I'm not at the computer?

Today I put together a small proof-of-concept that answers that — [goose-signal-gateway](https://github.com/theronconrey/goose-signal-gateway), a Python service that bridges Signal Messenger to a running Goose instance. You send a text, Goose replies.

---

## How it works

The gateway sits between two daemons that were already running on my home server:

- `signal-cli` — handles Signal protocol and exposes an HTTP API
- `goosed` — the Goose agent server that powers the Desktop client

```
Signal (phone) → signal-cli → gateway → goosed → Mistral → Signal reply
```

When a message comes in over Signal, the gateway creates a Goose session (or reuses the existing one for that sender), forwards the text, streams the reply back, and sends it as a Signal message. Each Signal conversation gets its own persistent Goose session, so context carries across messages.

The interesting part technically was figuring out `goosed`'s actual API. It's not documented publicly — I had to probe it against a live instance to map out the endpoints, auth scheme, and how the SSE streaming works. That's all captured in [`docs/acp-findings.md`](https://github.com/theronconrey/goose-signal-gateway/blob/master/docs/acp-findings.md) in the repo if you're curious.

---

## Still early

This is genuinely a rough proof of concept. A few things that work:

- Send a Signal message, get a Goose reply
- Session context is maintained within a conversation
- Runs against whatever provider Goose Desktop is configured for (Mistral in my case)

A few things that don't yet:

- Sessions don't show up in the Goose Desktop sidebar — the Desktop doesn't poll for externally-created sessions, it only knows about ones it started itself. Worth filing upstream.
- No message dedup — if Signal delivers a message twice, Goose replies twice
- Sessions reset on gateway restart
- Linux only (uses `/proc` to discover goosed's dynamic port)

---

## Where I'd like to take it

Goose itself is moving fast. Once the agent API stabilizes and there's a cleaner way to integrate with it — rather than poking at an undocumented internal HTTP server — I'd like to revisit this properly. The goal would be something robust enough to leave running: a persistent bridge so you can have a real conversation with your local agent from your phone, with full Desktop visibility on the same session.

For now it's a fun thing that works well enough to be useful. If you're running Goose Desktop and signal-cli and want to try it:

- [github.com/theronconrey/goose-signal-gateway](https://github.com/theronconrey/goose-signal-gateway)

<p style="text-align: center; margin-top: 1em;"><a href="https://github.com/theronconrey/goose-signal-gateway">goose-signal-gateway on GitHub</a> | <a href="https://github.com/block/goose">Goose on GitHub</a></p>
