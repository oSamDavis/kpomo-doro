# Kpomo-doro — Product Spec (v0.1)
*One-pager · March 2026*

---

## What We're Building

A Zoom app that takes how much time you have, and automatically divides it into a structured session of focused work and intentional rest — no manual timer-juggling, no screen-sharing a Chrome tab. Built for people who want to work deeply, alone or together, inside Zoom.

---

## The Core Concept

**1 KPOMO = 1 complete time block** — the total time a user has available (e.g., 60 minutes). The app divides that block into alternating **Work Sessions** and **Break Sessions**. Together, those sessions fill the KPOMO.

```
Work | Break | Work | Break | Work | Long Break
```

The formula for how to split that time is grounded in Pomodoro research: cognitive fatigue sets in after ~20–30 minutes of sustained focus, and structured breaks consistently outperform self-regulated ones for maintaining attention and reducing mental fatigue.

---

## The Auto-Split Formula

When a user inputs their total time, the app applies the following logic:

| Total Time | Pattern Applied |
|---|---|
| < 30 min | 1 Work Session, no break |
| 30–60 min | Short work blocks (~20 min), short breaks (~5 min) |
| 60–90 min | Full KPOMO: 3 work sessions, 2 short breaks, 1 long break |
| 90+ min | Multiple KPOMOs auto-chained |

**Default ratios (research-backed):**
- Work session: ~20–25 min
- Short break: 5 min
- Long break: 15–20 min (after every 3 work sessions)

The system calculates the closest clean fit to the user's total time, rounds gracefully, and always **ends on a break** — rest is not optional.

---

## Preset Templates

Rather than surfacing raw numbers, users choose a mode. Each mode reflects a different work/break philosophy:

| Template | Vibe | Work Block | Break | Best For |
|---|---|---|---|---|
| **Just Work** | Relaxed cadence | 20 min | 7 min | Light tasks, warm-up days |
| **Lock In** | Classic Pomodoro | 25 min | 5 min | Standard deep work |
| **Ultra Lock In** | High intensity | 45 min | 12 min | Flow state, long projects |

Each template auto-fills when you input total time, so you always see exactly what your session looks like before you start.

---

## Custom Mode

Users who want full control can open a **Custom window** and set:
- Work session duration
- Short break duration
- Long break duration
- When the long break triggers (e.g., after every N work sessions)

The app still enforces a minimum break duration (3 min) to stay true to the rest-first ethos.

---

## Session Flow (End to End)

1. **Input** — User enters total available time
2. **Preview** — App shows the auto-generated KPOMO breakdown (sessions, breaks, template label)
3. **Choose or Customize** — Accept the preset or switch to custom
4. **Start** — Session begins inside Zoom; timer visible to all participants
5. **Session Boundary** — At end of each work session, optional prompts appear:
   - *Before work session:* "What's your goal for this block?" (optional)
   - *After work session:* Quick checkbox-style check-in ("How'd it go?") — minimal, low-pressure
6. **Break** — Soothing visual displayed; rest is encouraged, not filled
7. **Recap** — At end of the full KPOMO, a summary of check-ins is saved to Zoom meeting notes (future: exportable as a nicely formatted doc)

---

## Design Principles

- **Rest is a feature, not a gap.** Breaks get their own visual treatment — calm, warm tones (yellows, rust). Not a countdown to get back to work.
- **Low friction.** Every prompt is optional. The check-ins are checkboxes and short text, never a form.
- **Social by default.** All participants in a Zoom call share one KPOMO — same timer, same state, same breaks. Shared accountability without surveillance. Late joiners sync into the active KPOMO in progress.

---

## MVP Scope

**In:**
- Total time input → auto-split into KPOMO
- 3 preset templates (Just Work, Lock In, Ultra Lock In)
- Custom mode
- In-session timer with work/break display
- Optional goal + check-in prompts at session boundaries
- Basic recap saved at end of KPOMO

**Out (V2):**
- Task → Work Session mapping
- Recurring KPOMO schedules
- Analytics / historical session data

---

## Decisions Made

- **Recap saves to:** Zoom meeting notes (MVP). Future V2 consideration: export as a formatted doc, since Zoom notes have low visibility.
- **Multi-user model:** All participants share one KPOMO. One session, one timer, everyone in sync.
- **Mid-session joins:** Late joiners sync into the active KPOMO already in progress — no separate session spawned.
