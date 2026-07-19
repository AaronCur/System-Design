# LLD Delivery Framework Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## The Five Phases

| # | Phase | Time | Output |
|---|---|---|---|
| 1 | Requirements | ~5 min | Numbered spec + explicit out-of-scope list |
| 2 | Entities & Relationships | ~3 min | Entity list + ownership arrows (no formal UML) |
| 3 | Class Design | ~10-15 min | State + behavior per class, top-down from orchestrator |
| 4 | Implementation | ~10 min | Happy path → edge cases → verification trace |
| 5 | Extensibility | ~5 min (if reached) | Point to design seams, explain why change stays clean |

## Phase 1 — Requirements: Four Question Themes
| Theme | Ask |
|---|---|
| Primary capabilities | What operations must this support? |
| Rules & completion | What defines success/failure/state transition? |
| Error handling | How are invalid inputs/actions handled? |
| Scope boundaries | What's explicitly in vs. out of scope? |

## Phase 2 — Entity Filter
- Maintains changing state or enforces rules → **entity**
- Just data attached to something else → **field**, not its own class

Relationship questions: who's the **orchestrator**? who owns durable state? has-a/uses/contains? where does each rule live?

## Phase 3 — Class Design: Two Questions Per Class
1. **State** — what must it remember to satisfy requirements it owns?
2. **Behavior** — one method per real action/query the outside world needs (small, focused API)

**Rule placement (encapsulation / "Tell, Don't Ask"):**
| Rule type | Lives in |
|---|---|
| Workflow/lifecycle ("can this run now?") | Orchestrator |
| Data-specific ("is this spot free?") | The entity that owns that data |

⚠️ No UML — use simplified class notation (state + methods list). Ask interviewer if unsure.

## Phase 4 — Implementation Order
1. Ask interviewer: pseudocode, real code, or talk-through?
2. **Happy path** first (inputs → steps → calls → result/state change)
3. **Edge cases** second (invalid input, illegal ops, wrong state)
4. **Verify**: trace one concrete scenario tick-by-tick — catches logic bugs before the interviewer does

⚠️ Don't force patterns (Strategy/Factory/Singleton) where they don't add value — over-engineering is the more common failure mode, not under-using patterns.

## Phase 5 — Extensibility
- Interviewer proposes a twist → point to the **design seam** that absorbs it, explain *why*, stay high-level (no live rewrite)
- Expectation scales by level: Junior (little/none) → Mid (1-2 follow-ups) → Senior (several, some self-initiated)

## One-Line Reminders
- "Requirements before entities before classes" — skipping ahead causes guessing and scope creep.
- "Rules live with the data they govern" — orchestrator handles *when*, entity handles *what*.
- "Happy path first, always" — structure before edge cases, every phase.
- "Verify before you're told to" — trace a scenario unprompted; it's often graded.
