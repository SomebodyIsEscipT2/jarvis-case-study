# Case Study: E.V — A Local-First AI Assistant (71 tools, 293 modules)

*A technical write-up of a private project, describing architecture and design decisions without exposing source code.*

## Overview

E.V ("JARVIS") is a local-first desktop AI assistant I've built and used as a **daily driver** — not a demo. It runs entirely on my own machine: a Python/FastAPI backend with a browser-based HUD frontend, WebSocket state sync, and ~71 callable tools across calendar, email, messaging, finance, development activity, tasks, and memory.

**Key stats:**
- ~293 Python modules, single-user production use
- Runs at `127.0.0.1:7777` — no cloud dependency for the core loop
- Integrations: Google Calendar/Email (OAuth), iMessage (read-only, macOS Full Disk Access), banking (Plaid, read-only, per-institution consent), GitHub activity
- Voice + text interaction, with listening/thinking/speaking HUD states

## Architecture

```
Browser HUD (frontend)
   │  WebSocket — state sync (idle / listening / thinking / speaking)
   ▼
FastAPI backend (127.0.0.1:7777)
   │
   ├── brain.py ── request router
   │      ├── deterministic fast-path (simple actions, no LLM call)
   │      └── tool-augmented LLM (streamed) for everything else
   │
   ├── Tool layer (~71 tools)
   │      calendar · email · sms · finance · github · tasks
   │      memory · awareness · briefings · app control · …
   │
   └── Integrations — each one a separate, explicit consent step
          OAuth (Google) · Plaid Link (banking) · OS permissions (camera/mic/sms)
```

### Design decisions worth explaining

**1. Deterministic fast-path before the LLM.**
Most assistant requests are simple ("what's on my calendar today"). Sending every request to an LLM adds latency, cost, and failure modes. The brain routes simple, well-shaped actions to deterministic code paths and reserves the LLM (with tool access) for everything else. Result: instant answers for routine requests, streamed LLM replies for complex ones.

**2. Consent is per-integration, never bundled.**
Connecting your calendar does not grant messaging. Every integration is a separate, explicit connection: an OS permission dialog, an OAuth screen, or a Plaid Link flow. This was a deliberate choice — trust in a personal assistant is earned per-capability, and a "grant everything" dialog would have been the easy-but-wrong design.

**3. Privacy boundaries are explicit, not implied.**
Some data (like email content) stays private even from the assistant's own spoken summaries — it surfaces counts, not content. Financial anomaly detection (flagging transactions ≥3× the category median) runs server-side locally and notifies once per flagged transaction. The foreground-app awareness observer is the one deliberate exception — it runs continuously, is local-only, sends nothing anywhere, and has a single explicit off switch. That trade-off is documented in the repo's own README rather than hidden.

**4. Auth boots locked.**
The assistant starts locked and requires a face scan or passcode; auto-relocks after 15 idle minutes. A machine that knows your finances, texts, and calendar should not be trivially readable at the lock screen.

**5. Proactive, not reactive.**
Scheduled morning/evening briefings (calendar, tasks, unread mail, new texts — counts only) and a spend watcher make the assistant surface information when it's useful, instead of waiting to be asked. Priority notifications degrade to quiet banners when you're mid-meeting — context-awareness applied to interruptions.

## What building it taught me

- **Latency is a feature.** Routing around the LLM for simple requests changed how usable the assistant feels day to day.
- **Consent architecture is product design.** Deciding what each integration can and cannot see — and making those boundaries visible to the user — mattered as much as any feature.
- **A daily driver forces honesty.** Software you use every day exposes every rough edge. The README's "status: daily driver, not a demo" section exists because the docs had drifted from what the code actually did; fixing that drift was a real engineering task.
- **Local-first is viable.** With streaming LLM APIs and local inference as fallback, a fully local assistant with cloud-only model calls (never data) is practical on consumer hardware.

## Stack

Python · FastAPI · WebSockets · OpenRouter (streamed LLM, tool-augmented) · Google OAuth · Plaid · macOS integrations (Full Disk Access, AppKit) · browser HUD frontend

---
*The source is private because it contains personal integrations and configuration. This write-up describes the architecture; I'm happy to walk through any part of it in conversation.*
