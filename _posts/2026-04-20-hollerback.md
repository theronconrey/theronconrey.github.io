---
layout: post
title: "hollerback: Signal for your AI agents"
date: 2026-04-20
---

A couple days ago I posted about [goose-signal-gateway](https://theron.wtf/2026/04/18/goose-signal-gateway.html), a rough proof-of-concept that let you text your local Goose instance from Signal. It worked. The core loop was implemented, I was using it, and I wanted to share it while the code was still honest about what it was.

Since then I've done a proper pass on it. The project is now called **hollerback**, it's on [PyPI](https://pypi.org/project/hollerback/), and it's meaningfully more capable than what I described two days ago. Here's what changed and why.

---

## Why "hollerback"

The original name reflected the implementation: a gateway between Signal and Goose. As I kept building it became clear the thing wasn't really a Goose-specific gateway. It was a Signal number that any AI agent could use. Goose is still the primary target but a rename felt right.

---

## Two use cases, one phone

The clearest way I've found to describe hollerback is a dedicated Signal number to share with your agents. There are two primary use cases today.

**Use case 1: Goose is live and waiting.** Someone messages the number, Goose picks up and replies: unattended, session-aware, in real time. This is the use case from the April 18 post.

**Use case 2: An authorized agent can use the number to send messages.** Any MCP client (Claude CLI, Goose Desktop, Cursor, Claude Desktop) connects to hollerback's MCP endpoint via a standard HTTP Bearer token and gets tools to send Signal messages, list paired contacts, and read inbound traffic. If you're in a Claude session and want to send someone a Signal message, you can. If you want Goose to notify you on Signal when a long task finishes, it can.

They share one process, one contact list, one phone number. You can run either use case alone or both together.

---

## What's working today

- Signal to Goose: inbound messages create or resume goosed sessions, Goose replies stream back to Signal
- Typing indicators while Goose is processing; read receipts on delivery
- Pairing flow for unknown senders: an unknown number gets a code; you approve it via CLI before the bot responds
- MCP server on port 7322 with four tools: `get_signal_identity`, `list_signal_contacts`, `send_signal_message`, `get_messages`
- Per-agent Bearer auth: multiple agents can share one Signal number, each with its own key
- Graceful operation without Goose Desktop: messages buffer when goosed is unreachable, auto-replies resume on reconnect
- systemd user service, `hollerback.service`
- Running on Fedora, smoke-tested in production

---

## Install

```bash
uv tool install hollerback
hollerback setup
hollerback start --detach
```

Or from source at [github.com/theronconrey/hollerback](https://github.com/theronconrey/hollerback). Full setup instructions in the README.

---

The project is really early. I'm settling in where I can help, and I'm looking forward to engaging with the Goose team!

<p style="text-align: center; margin-top: 1em;"><a href="https://github.com/theronconrey/hollerback">hollerback on GitHub</a> | <a href="https://pypi.org/project/hollerback/">hollerback on PyPI</a> | <a href="https://github.com/aaif-goose/goose">Goose on GitHub</a></p>
