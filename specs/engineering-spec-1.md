# Kpomo-doro — Engineering Spec (v0.1)
*High-level architecture · March 2026*

---

## What We're Building (Engineering Lens)

A Zoom App that manages a shared, real-time timer session across all participants in a meeting. The core engineering challenge is **state synchronization**: everyone in the call must see the same timer, the same session phase, and transition together — including people who join late.

This is fundamentally a **distributed state problem** — multiple clients, one shared truth.

---

## System Components

```
┌─────────────────────────────────────────────────────┐
│                    ZOOM APP (UI)                     │
│  - Session setup (time input, template picker)       │
│  - Timer display (work / break phase)                │
│  - Break visuals                                     │
│  - Goal prompt + check-in form                       │
│  - Recap summary view                                │
└────────────────────┬────────────────────────────────┘
                     │  Zoom Apps SDK
          ┌──────────▼──────────┐
          │   STATE SYNC LAYER  │
          │  (broadcast events) │
          └──────────┬──────────┘
                     │
          ┌──────────▼──────────┐
          │      BACKEND        │
          │  - Timer authority  │
          │  - Session storage  │
          │  - Recap data       │
          └──────────┬──────────┘
                     │  Zoom REST API
          ┌──────────▼──────────┐
          │   ZOOM MEETING      │
          │   NOTES (recap)     │
          └─────────────────────┘
```

Each layer has a single responsibility — the UI renders, the sync layer propagates, the backend persists. This is **Separation of Concerns** in practice: changes to how recap is stored shouldn't touch the timer display, and changes to the formula shouldn't affect how events are broadcast.

---

## The Core Problem: Who Owns the Timer?

This is the most important architectural decision in the app, and it's a **Systems Design** question at its core: **where does the Single Source of Truth (SSOT) live?**

In any system where multiple consumers need to agree on shared state, you must designate one authority. Ambiguity here causes drift, conflicts, and bugs that are hard to reproduce.

### Option A — Client-side authority (simpler, riskier)
The host's instance of the app runs the timer locally. Every tick is broadcast to all other participants via the Zoom Apps SDK `sendAppMessage`. Participants receive the tick and update their UI.

- ✅ Easier to build, no backend needed initially
- ❌ If the host leaves or their app crashes, the timer dies
- ❌ Network lag can cause visible **drift** between participants — a form of **eventual consistency** that's acceptable in some systems but jarring in a shared timer

### Option B — Server-side authority (robust, more work)
A backend server owns the timer. All clients subscribe to it. The server emits state updates (current phase, time remaining) via WebSockets or polling. Clients render whatever the server says.

- ✅ Timer survives host disconnection
- ✅ Late joiners get accurate state immediately by querying the server
- ✅ True SSOT — the server is always right
- ❌ Requires building and hosting a backend

**Recommendation for MVP:** Start with Option A to validate the product, with the data model designed so Option B can be swapped in later without rebuilding the UI. This is the **Strangler Fig** principle — design for replaceability from day one.

---

## Design Patterns in Play

These patterns are not theoretical — they describe decisions you'll actually make while building.

### Observer Pattern (Pub/Sub)
The entire sync layer is an Observer Pattern. The host (or server) is the **Subject** — it holds state and emits events when state changes. Each participant's app instance is an **Observer** — it subscribes to those events and updates its UI in response.

```
Subject (timer authority)
  └── notifies → Observer A (participant 1's UI)
  └── notifies → Observer B (participant 2's UI)
  └── notifies → Observer C (late joiner's UI)
```

Zoom's `sendAppMessage` / `onMessage` is literally an implementation of this pattern. Understanding it as Observer helps you reason about what happens when observers are added (late joins), removed (participant leaves), or miss an event (network drop).

### Finite State Machine (FSM)
The `KpomoSession` lifecycle is a **Finite State Machine** — a system with a defined set of states and explicit rules for how it transitions between them.

```
[setup] ──start──▶ [running] ──pause──▶ [paused]
                       │                    │
                    complete              resume
                       │                    │
                       ▼                    ▼
                  [complete]           [running]
```

Modelling this explicitly prevents illegal transitions (e.g., "complete" jumping back to "running") and makes the app's behaviour predictable. Each block within a session is also a mini state machine: `pending → active → done`.

### Command Pattern
Every meaningful action in the app — `StartSession`, `AdvanceBlock`, `PauseSession`, `SubmitCheckin` — can be modelled as a **Command**: a discrete, self-contained object that encapsulates an action and its data.

This matters because:
- Commands are easy to broadcast (serialize to JSON, send via `sendAppMessage`)
- They create a natural **event log** — if you store them in order, you can reconstruct any past session state
- They enable future features like undo, replay, or audit trails without rearchitecting

### Strategy Pattern
The auto-split formula applies different calculation strategies depending on the selected template. Each template (Just Work, Lock In, Ultra Lock In, Custom) is a **Strategy** — a swappable algorithm that implements the same interface (`input: totalMinutes → output: blocks[]`). New templates can be added without changing the core formula logic.

### Facade Pattern
The Zoom Apps SDK is a **Facade** — it hides the complexity of Zoom's underlying platform (meeting state, participant management, broadcast infrastructure) behind a clean, simple API. You call `sendAppMessage`, not "route a UDP packet through Zoom's signalling layer." Recognising facades helps you understand the boundary between what you control and what you depend on.

---

## State Model

At any point in time, the app needs to know:

```
KpomoSession {
  sessionId         // unique ID per KPOMO
  meetingId         // Zoom meeting this belongs to
  totalDuration     // total time the user input (in seconds)
  template          // "just-work" | "lock-in" | "ultra-lock-in" | "custom"
  blocks: [
    {
      type          // "work" | "short-break" | "long-break"
      duration      // in seconds
      index         // order in the sequence
    }
  ]
  currentBlockIndex // which block is active
  blockStartedAt    // timestamp when current block began
  status            // "setup" | "running" | "paused" | "complete"
  checkins: [
    {
      blockIndex    // which work session this belongs to
      goal          // optional string (set before block)
      reflection    // optional string (set after block)
    }
  ]
}
```

This object is the **Single Source of Truth** for the session. All UI is derived from it — nothing is stored locally per-client beyond what's in this shape. This makes late joins straightforward: receive the current `KpomoSession`, render it. No special-casing needed.

---

## Component Communication Model

Understanding how components talk to each other is as important as knowing what they do.

```
┌──────────┐   Command (JSON)    ┌─────────────┐
│  HOST UI │ ──sendAppMessage──▶ │  SYNC LAYER │
└──────────┘                     └──────┬──────┘
                                        │ onMessage (broadcast)
                          ┌─────────────┼─────────────┐
                          ▼             ▼             ▼
                     Participant A  Participant B  Late Joiner
                       (re-render)   (re-render)  (state request
                                                   → full sync)
```

The sync layer is **unidirectional** in Option A: commands flow from the authority (host), events flow out to all observers. This mirrors **Flux/Redux architecture** in frontend systems — one direction of data flow makes the system easier to reason about and debug.

In Option B, the server sits in the middle of this diagram and the host becomes just another observer.

---

## The Auto-Split Formula (Engineering Version)

Given `totalMinutes` as input and a selected template (Strategy), the algorithm:

1. Determine `workDuration` and `shortBreakDuration` from the template
2. Calculate how many complete `work + short break` pairs fit: `pairs = floor(totalMinutes / (workDuration + shortBreakDuration))`
3. Reserve time for the final long break (15–20 min) if total time allows
4. If leftover time after pairs > half a work session, add one more work session
5. Always append a long break at the end as the final block
6. Return the ordered array of blocks

Edge cases to handle:
- Total time < 30 min → 1 work block, no break
- Leftover time that doesn't cleanly fit a full block → round down, absorb into long break

---

## Key Integration: Saving Recap to Zoom Meeting Notes

At the end of a KPOMO, the app calls the **Zoom REST API** (not the Apps SDK — these are separate systems with separate auth) to write a summary of check-ins to the meeting's notes. This requires:

- OAuth authentication with the `meeting:write` scope
- A formatted payload built from the `checkins` array in the session state
- The meeting ID from the running Zoom context

This is a good example of a **system boundary** — the point where your app hands off to an external system you don't control. You need to handle failures here gracefully (what if the API call fails at the end of a session?), which means the recap should be built and stored locally first, then written out. Never lose user data on an outbound call failure.

⚠️ **Zoom meeting notes have low user visibility** — known limitation flagged in the product spec. The write call is straightforward; the UX value is TBD and may push toward a V2 export feature.

---

## Open Architectural Questions

- **Persistence between meetings:** Should session history be stored? If yes, we need a database and user accounts. If no, state lives only for the duration of the meeting.
- **Pause/resume:** Can the host pause the timer mid-block? The FSM state model supports it (`status: "paused"`) but the product spec doesn't address it yet.
- **Who can start a KPOMO?** Host only, or any participant? This affects permission checks in the SDK and the authority model.
- **Backend hosting:** If we go Option B, where does it live? (Vercel, Railway, etc.) Zoom Apps require HTTPS endpoints — use `ngrok` locally.

---

## Stack Decision Points (To Resolve)

| Decision | Options | What to Consider |
|---|---|---|
| Frontend framework | React, Vue, vanilla JS | React is most common in Zoom App examples |
| Backend (if needed) | Node/Express, Next.js API routes | JS across the stack reduces context switching |
| Real-time transport | Zoom SDK broadcast, WebSockets, SSE | Zoom broadcast is free; WebSockets needs a server |
| Hosting | Vercel, Railway, Render | Zoom Apps require a public HTTPS URL even in dev |
| State management | React context, Zustand, Redux | Keep it simple; the FSM should live here |

---

## What to Build First

1. **Scaffold the Zoom App** — get a basic app running inside a Zoom meeting (this alone is a meaningful milestone)
2. **Build the timer locally** — single client, no sync, just the UI and formula logic working
3. **Add broadcast sync** — host broadcasts state, one other participant receives it (Observer Pattern working end to end)
4. **Add session setup flow** — time input, template picker, preview screen (Strategy Pattern plugged in)
5. **Add check-ins** — goal prompt before work session, checkbox after
6. **Wire up Zoom meeting notes** — write recap at end of KPOMO (system boundary handled gracefully)
7. **Handle late joins** — request state on app open mid-session (Observer joining after Subject has started)
