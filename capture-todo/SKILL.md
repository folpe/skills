---
name: capture-todo
description: >
  Use when the user expresses a task, idea, reminder, or note in any form — dictated, typed, mid-sentence, or as an explicit "add to my todo". Triggers: "remind me to", "I need to", "don't forget", "add to my list", "todo", "note ça", "j'ai pensé à", "il faudrait", "rappelle-moi", any imperative or future-action phrasing not tied to the current task. Routes the captured item into Todoist with the right project, section, labels (duration/energy/context), date, and priority. Also exposes daily-pull and digest primitives for morning briefings using the GTD-allégé methodology (configurable).
---

# capture-todo

Generic Todoist capture + daily-triage skill. Detects actionable thoughts, classifies them with consistent conventions, and produces a daily digest.

## Sections

1. When to activate
2. Primitives overview
3. Setup (first run)
4. Capture primitive
5. Daily-pull primitive
6. Digest primitive
7. Methodology (configurable)
8. Label convention v2
9. Configuration file
10. Failure modes & fallbacks

## 1. When to activate

Activate **automatically** (no explicit user request needed) when the user's message contains a clear actionable signal:

**Strong signals (activate immediately):**
- Imperatives directed at self: "I need to X", "I should X", "remind me to X", "rappelle-moi de X", "il faudrait X", "j'ai pensé à X"
- Explicit list verbs: "add to my todo", "note ça", "ajoute X à ma todo"
- Future-tense self-commitments: "tomorrow I'll X", "this week I have to X"

**Weak signals (activate but ask 1 clarifying question max):**
- Bare nouns dropped without context: "appel banque", "renouveler assurance moto"
- Mixed messages where the actionable part is buried in conversation

**Do NOT activate when:**
- The user is asking a question, debugging, coding, or in the middle of another task
- The "task" is something the user is doing right now (e.g. "I'm going to read this PR" — that's narration, not a todo)
- The user explicitly says "don't add this" or similar opt-out

If unsure, default to NOT activating and let the user say "add it to todoist" explicitly.

## 2. Primitives overview

The skill exposes 4 primitives. Each primitive is a procedure Claude executes by reading the corresponding section.

| Primitive | When | Section |
|---|---|---|
| `setup` | First run, or when convention drifted | §3 |
| `capture` | Whenever an actionable signal is detected (auto) or user requests "add to todoist" | §4 |
| `daily-pull` | Called by the morning routine — selects today's `must`, applies `stale` flag, builds candidate list | §5 |
| `digest` | Called by the morning routine after `daily-pull` — formats and delivers the briefing | §6 |

Primitives are stateless — all state lives in Todoist + the user's config file (§9).

## 3. Setup (first run)

Run this primitive when the skill is invoked for the first time, or when `~/.config/capture-todo/config.yaml` is missing.

### Step 3.1 — Discover existing structure

Call (in parallel):
- `mcp__claude_ai_Todoist__user-info` — confirm timezone, plan
- `mcp__claude_ai_Todoist__get-overview` — list projects + sections
- `mcp__claude_ai_Todoist__find-labels` (limit 200) — list labels

### Step 3.2 — Compare against v2 convention

The v2 label set is the 18 labels in §8. For each missing label, plan an `add-labels` call. For each label whose name matches an old convention (T5/T15/T30/T45/E1/E2/E3/Frog/Important/IA generated), plan a rename or delete.

**Show the diff to the user before applying.** Wait for explicit "go" before mutating anything.

### Step 3.3 — Create labels (after user approval)

Use `mcp__claude_ai_Todoist__add-labels` (batch) and `update-labels` (batch).

### Step 3.4 — Detect projects/sections mapping

Ask the user 5 questions, in this order, accepting "skip" for any:

1. Which project holds your **work / pro** tasks? (offer existing project names + "create new")
2. Which project holds your **personal** tasks?
3. Which project holds your **finance / investments**? (optional)
4. Which project holds your **hobbies / side**? (optional)
5. List your physical contexts in use (defaults: `@ordi @tel @dehors @maison @courses`).

### Step 3.5 — Write config

Create `~/.config/capture-todo/config.yaml` with the gathered mapping (see §9 schema). Confirm written, show contents, ask user to validate.

### Step 3.6 — Done

Print: "Setup complete. The skill is now active. Try saying 'remind me to test capture-todo tomorrow morning'."

## 4. Capture primitive

Pipeline executed when activation conditions (§1) are met.

### Step 4.1 — Extract intent

Parse the user's message into a structured intent:

```yaml
content: <imperative form, ≤80 chars, action verb first>
context_hints:
  topic: <work | personal | finance | hobby | unknown>
  where: <ordi | tel | dehors | maison | courses | unknown>
  duration_estimate: <5min | 15min | 30min | 1h | unknown>
  energy_estimate: <easy | focus | deep | unknown>
  due_natural: <"tomorrow" | "friday" | "tonight" | null>
  priority_signal: <p1 | p2 | p3 | p4>  # default p4 unless signals
description: <full original phrasing if longer than content>
```

Heuristics:
- Topic: keyword match against config's project mapping (§9). If no match → `unknown`.
- Where: keyword match against the user's contexts list.
- Duration: explicit ("ça prend 5 min") or estimate from verb (call → 15min, write doc → 1h, send email → 5min).
- Energy: `deep` if requires focus/creative work, `focus` if structured but not heavy, `easy` for admin/errands.
- Due: parse natural language. If user says "demain" and current time > 18h, interpret as next morning.
- Priority signals: "urgent" / "ASAP" → p1, "important" → p2, default p4.

### Step 4.2 — Decide routing

| Confidence | Routing |
|---|---|
| Topic + where both known | Direct to project's section |
| Topic known, where unknown | Project's default section, `@unknown` skipped |
| Topic unknown | Inbox |

Always tag `ai-captured`. Never tag `Classified` (that label is reserved for cron-triaged items).

### Step 4.3 — Build the Todoist call

Use `mcp__claude_ai_Todoist__add-tasks` with:

```json
{
  "tasks": [{
    "content": "<extracted content>",
    "description": "<extracted description, if any>",
    "projectId": "<resolved>",
    "sectionId": "<resolved or null>",
    "labels": ["<duration>", "<energy>", "<@where>", "ai-captured"],
    "dueString": "<natural>",
    "priority": "<p1-p4>"
  }]
}
```

Skip any label whose value is `unknown`.

### Step 4.4 — Confirm to user

One line, terse:

```
✓ Captured "<content>" → <project>/<section> · <labels> · <due>
```

If routing went to Inbox, suggest a project: "Inbox for now — want me to file under <best-guess>?"

### Step 4.5 — Failure modes

- MCP Todoist unavailable → write to `~/.config/capture-todo/pending.jsonl` (one JSON intent per line). On next successful capture, flush pending queue first.
- Ambiguous intent → ask 1 question max, then proceed with best guess + tag `someday` if still unsure.
