---
name: capture-todo
description: >
  Use when the user expresses a task, idea, reminder, or note in any form — dictated, typed, mid-sentence, or as an explicit "add to my todo". Triggers: "remind me to", "I need to", "don't forget", "add to my list", "todo", "note ça", "j'ai pensé à", "il faudrait", "rappelle-moi", "j'oublie pas", "faut que je", "une idée à creuser", any imperative or future-action phrasing not tied to the current task. Routes the captured item into Todoist with the right project, section, labels (duration/energy/context), date, and priority. Also exposes daily-pull and digest primitives for morning briefings using the GTD-allégé methodology (configurable).
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

## Prerequisites

Before running `setup`:

- **Claude Code (or claude.ai) with MCP support** — needed for the Todoist integration to be reachable.
- **Todoist MCP server connected** — install and authenticate the official Todoist MCP from your Claude settings. The skill will refuse to operate if it cannot reach the MCP.
- **Telegram bot (for the digest only)** — create a bot via [@BotFather](https://t.me/BotFather), grab the token, send `/start` to your new bot, and find your `chat_id` from `https://api.telegram.org/bot<TOKEN>/getUpdates`. Store both in `~/.config/capture-todo/secrets.env` with mode `0600`.
- **Cron / schedule** — for the daily digest to fire automatically, the user creates a routine (Claude `/schedule` or system cron) that calls `daily-pull` then `digest`. The skill itself does not run on a timer; it exposes the primitives.

If any of these are missing, the `capture` primitive still works for ad-hoc capture; only the morning digest depends on Telegram + cron.

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

**Tiebreaker — todo dropped mid-task:** if the user is in the middle of another task (coding, debugging, writing) AND drops a self-directed action that is *distinct from* the current task ("while I'm at it, remind me to email Pierre", "il faut aussi que je relance le client"), the strong-signal rule wins — capture immediately. Do NOT activate if the imperative is *part of* the current task ("I need to add error handling here").

If unsure, default to NOT activating and let the user say "add it to todoist" explicitly.

## 2. Primitives overview

The skill exposes 4 primitives. Each primitive is a procedure Claude executes by reading the corresponding section.

| Primitive | When | Section |
|---|---|---|
| `setup` | First run, or when convention drifted | §3 |
| `capture` | Whenever an actionable signal is detected (auto) or user requests "add to todoist" | §4 |
| `daily-pull` | Called by the morning routine — selects today's `must` (label, see §8), applies `stale` (label, see §8) flag, builds candidate list | §5 |
| `digest` | Called by the morning routine after `daily-pull` — formats and delivers the briefing | §6 |

Primitives are stateless — all state lives in Todoist + the user's config file (§9).

## 3. Setup (first run)

Run this primitive when the skill is invoked for the first time, or when `~/.config/capture-todo/config.yaml` is missing.

### Step 3.1 — Discover existing structure

**Preflight:** if the Todoist MCP server is not connected (any of the calls below returns a connection error), STOP. Tell the user: "Todoist MCP is not reachable. Connect the integration in your Claude settings (claude.ai → Settings → MCP), then re-run setup." Do not proceed.

If MCP is reachable, call (in parallel):
- `mcp__claude_ai_Todoist__user-info` — confirm timezone, plan
- `mcp__claude_ai_Todoist__get-overview` — list projects + sections
- `mcp__claude_ai_Todoist__find-labels` (limit 200) — list labels

### Step 3.2 — Compare against v2 convention

The v2 label set is the labels listed in §8. For each missing label, plan an `add-labels` call. For each label whose name matches an old convention (T5/T15/T30/T45/E1/E2/E3/Frog/Important/IA generated), plan a rename or delete.

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

Print: "Setup complete. The skill is now active. Try saying 'remind me to send the report Friday morning'."

## 4. Capture primitive

Pipeline executed when activation conditions (§1) are met.

### Step 4.1 — Extract intent

Parse the user's message into a structured intent:

```yaml
content: <imperative form, ≤80 chars, action verb first>
context_hints:
  topic: <work | personal | finance | hobby | unknown>
  where: <@ordi | @tel | @dehors | @maison | @courses | unknown>
  duration_estimate: <5min | 15min | 30min | 1h | unknown>
  energy_estimate: <easy | focus | deep | unknown>
  due_natural: <"tomorrow" | "friday" | "tonight" | null>
priority: <p1 | p2 | p3 | p4>  # default p4 unless explicit signals
description: <full original phrasing if longer than content>
```

Heuristics:
- Topic: keyword match against config's project mapping (§9). If no match → `unknown`.
- Where: keyword match against the user's contexts list. Always include the `@` prefix to match the Todoist label name (e.g. `@ordi`, not `ordi`).
- Duration: explicit ("ça prend 5 min") or estimate from verb (call → 15min, write doc → 1h, send email → 5min).
- Energy: `deep` if requires focus/creative work, `focus` if structured but not heavy, `easy` for admin/errands.
- Due: parse natural language. If user says "demain" and current time > 18h, interpret as next morning.
- Priority: "urgent" / "ASAP" → p1, "important" → p2, default p4. (Top-level field, not part of `context_hints`, since it maps directly to the Todoist payload.)

### Step 4.2 — Decide routing

| Confidence | Routing |
|---|---|
| Topic + where both known | Direct to project's section |
| Topic known, where unknown | Project's default section, no `@` label |
| Topic unknown, where known | Inbox, keep the `@` label |
| Topic + where both unknown | Inbox, no `@` label |

Always tag `ai-captured`. Never tag `classified` (that label is reserved for cron-triaged items).

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

---

## 5. Daily-pull primitive

Run by the morning routine. Selects today's commitments and updates lifecycle labels.

### Step 5.1 — Apply stale-flag

Find tasks where:
- Created or last-modified more than 3 days ago
- Not completed
- Not already labeled `stale` or `someday`

Tool: `mcp__claude_ai_Todoist__find-tasks` with filter:

```
!@must & !@next & !@stale & !@someday & (created before: -3 days | no date & created before: -3 days)
```

This catches any task without a lifecycle label that has been sitting more than 3 days. Tasks already in the lifecycle (`must`/`next`/`stale`/`someday`) are excluded — `stale` already has the flag, the others have an active intent.

For each result, add label `stale` via `update-tasks`.

### Step 5.2 — Select today's `must` (max 3)

Pick candidates by score:

| Signal | Weight |
|---|---|
| Already labeled `must` from yesterday | +5 |
| Labeled `next` | +4 |
| Priority p1 | +3 |
| Due today or overdue | +3 |
| Labeled `stale` | +2 (forces decision) |
| Energy matches current slot (`deep` if `now < config.morning_slot_until`, else `easy`) | +1 |

Top 3 by score → keep `must`. Drop `must` from anything that didn't make the cut.

**Tie-break:** if multiple tasks share the same score, prefer the one with the oldest creation date. Stable, deterministic, rewards old commitments.

### Step 5.3 — Build candidate buckets

Compute lists for the digest:
- `today_must`: the 3 selected
- `stale_to_decide`: items with `stale` label
- `quick_wins`: any task with both `5min` and `easy` that is NOT already in `today_must` and NOT labeled `someday` (prevents double-counting in the digest)
- `overdue`: due date < today

Return as a structured object — does NOT mutate Todoist beyond Step 5.1 stale-flag and 5.2 must-flag changes.

### Step 5.4 — Methodology variants

If config (`§9.methodology`) is one of:
- `eat-the-frog` — pick 1 highest-scored task, force as the day's headline; ignore `must` cap
- `ivy-lee` — pick 6 by score, present in strict order, no `stale` logic
- `mit-pure` — select 3 `must`, no `stale` logic at all

Default: `gtd-daily-pull-stale` (the algorithm above).

---

## 6. Digest primitive

Format and deliver the morning briefing. Called after `daily-pull`.

### Step 6.1 — Format

Markdown template (Telegram supports a subset — use plain text + emoji, no headers > H2):

```text
🌅 *Daily briefing — {date}*

🎯 *Today's 3 must* ({weekday})
1. {task1.content}  · _{task1.duration} · {task1.energy}_
2. {task2.content}  · _{task2.duration} · {task2.energy}_
3. {task3.content}  · _{task3.duration} · {task3.energy}_

⏳ *Stale (>3 days, decide)*
• {stale1.content} — do / downgrade / kill?
• ...

⚡ *Quick wins available* (5min + easy)
• {qw1.content}
• ...

🔴 *Overdue*
• {od1.content}  · was due {od1.due}
```

If a section is empty, omit it entirely (don't print "0 stale items").

### Step 6.2 — Deliver to Telegram

POST to `https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage` with `parse_mode=MarkdownV2`, `chat_id=${TELEGRAM_CHAT_ID}`.

**Escape user content before interpolation.** Telegram MarkdownV2 requires escaping these characters in any non-markup text: `_`, `*`, `[`, `]`, `(`, `)`, `~`, `` ` ``, `>`, `#`, `+`, `-`, `=`, `|`, `{`, `}`, `.`, `!`. Apply a backslash escape (e.g. `it_works` → `it\_works`) to every task `content`, `due`, and label string before substituting into the template. Failing to escape will produce a 400 error from the Telegram API.

Use `Bash` tool with `curl --data-urlencode text=...`. Read token + chat_id from `~/.config/capture-todo/secrets.env` (sourced via `set -a; source secrets.env; set +a`).

If the formatted message would exceed 4000 characters (Telegram cap is 4096), truncate the `stale_to_decide` and `quick_wins` sections to the top 5 entries and append `[… N more — see Todoist]` to each truncated list.

### Step 6.3 — Archive in Todoist

Locate the task `Daily Briefing` for archive. Resolution order:
1. If `config.archive_task_id` is set, use it directly.
2. Else search by name in `projects.work` → `projects.personal` → Inbox (first hit wins).
3. If not found, create it once in the same project (work > personal > Inbox), with label `someday` so it doesn't appear in the daily-pull, and **no recurrence** (it's a permanent anchor, not a recurring chore). Cache the new ID in `config.archive_task_id`.

Append today's digest as a new comment via `mcp__claude_ai_Todoist__add-comments`.

### Step 6.4 — Failure modes

- Telegram fails (network, bad token) → still archive in Todoist; report the failure to the routine output (visible in `/schedule` logs)
- No tasks at all in any bucket → silent by default (no Telegram message sent). To opt-in to a confirmation ping, set `notify_on_empty: true` in config (see §9). Either way, still archive a comment in Todoist saying "🌅 Nothing pulled — clear day."

---

## 7. Methodology (configurable)

The skill ships with 4 methodologies. Pick one in `~/.config/capture-todo/config.yaml`:

```yaml
methodology: gtd-daily-pull-stale  # default
```

The default (`gtd-daily-pull-stale`) automates triage via the `stale` flag. `mit-pure` is for users who triage manually — same daily cap, no automation. The other two re-shape what counts as a "day's commitment."

| Key | Best for | Daily cap | Stale logic |
|---|---|---|---|
| `gtd-daily-pull-stale` (default) | People whose tasks naturally drift over multiple days | 3 `must` | Yes (3-day threshold) |
| `eat-the-frog` | Mornings reserved for one hard thing | 1 task + N quick wins | No |
| `ivy-lee` | Strict-order execution, no replanning mid-day | 6 ordered | No |
| `mit-pure` | Light commitment, focus on 3 things | 3 `must` | No |

Switch methodology by editing the config file — no skill code changes needed.

---

## 8. Label convention v2

| Group | Labels | Meaning |
|---|---|---|
| Duration | `5min` `15min` `30min` `1h` | Estimated time-on-task |
| Energy | `easy` `focus` `deep` | Cognitive load required |
| Lifecycle | `must` `next` `stale` `someday` | Where in the funnel — `must` = today, `next` = tomorrow candidate, `stale` = drift signal, `someday` = parking lot |
| Context | `@ordi` `@tel` `@dehors` `@maison` `@courses` | Where/how to do it (GTD context — extensible per user) |
| Provenance | `ai-captured` | Came in via this skill (vs. manual) |
| Marker | `classified` | Triaged by `daily-pull` (do not apply manually) |

**Rules:**
- A task should ideally carry: 1 duration + 1 energy + ≥1 context + (lifecycle or none). The capture pipeline (§4.3) skips any field it could not infer — best-effort, not enforced.
- Multiple contexts allowed (`@ordi @maison` for "from home on laptop").
- `must` is mutually exclusive with `someday`. `stale` overrides nothing — it sits next to other labels as a flag.
- Never apply `classified` from `capture` — only `daily-pull` may set it.
- All label names are lowercase. Context labels keep the leading `@`.

---

## 9. Configuration file

Path: `~/.config/capture-todo/config.yaml`

Schema:

```yaml
methodology: gtd-daily-pull-stale  # see §7

projects:
  work:
    id: "<todoist project id>"
    default_section: "<id or null>"
  personal:
    id: "..."
    default_section: null
  finance:                # optional
    id: "..."
    default_section: null
  hobby:                  # optional
    id: "..."
    default_section: null
  inbox_fallback: true    # send unknowns to Inbox if no topic match

# Map free-text topic keywords to project keys above.
# Defaults are placeholders — replace with terms relevant to the user.
topic_routing:
  work:     ["client", "meeting", "deck", "code", "ship", "deploy"]
  personal: ["maison", "rdv", "famille", "santé", "admin"]
  finance:  ["impôt", "facture", "compta", "investis", "tax"]
  hobby:    ["sport", "lecture", "jeu", "loisir"]

contexts: ["@ordi", "@tel", "@dehors", "@maison", "@courses"]

morning_slot_until: "12:00"  # before this time, prefer `deep` energy tasks

notify_on_empty: false  # if true, send Telegram ping even when no tasks pulled (see §6.4)

archive_task_id: null  # filled by digest primitive on first run
```

The skill writes back to this file in only one case: `archive_task_id` is populated by the `digest` primitive the first time it creates the "Daily Briefing" anchor task (§6.3). All other keys are user-managed.

Secrets (Telegram token + chat_id) live in `~/.config/capture-todo/secrets.env`, never in this YAML.

Edit by hand — the skill will not overwrite user edits, only append missing keys when new versions of the skill add them.

---

## 10. Failure modes & fallbacks

| Failure | Behavior |
|---|---|
| MCP Todoist down | Queue capture intents to `~/.config/capture-todo/pending.jsonl`. Flush on next successful capture. |
| Telegram delivery fails | Still archive digest in Todoist comment. Surface error in routine output. |
| Config file missing | Trigger `setup` primitive automatically. |
| Config file malformed YAML | Refuse to operate. Print exact line+error. Suggest `cp config.yaml config.yaml.bak && rm config.yaml` to re-run setup. |
| Plan limit hit on Todoist (Free plan, 5 projects) | Detect during setup. Recommend creating one section per logical project inside an existing parent project, then setting all `topic_routing` buckets to that parent project's `id` and using distinct `default_section` IDs to differentiate routing destinations. |
| Ambiguous capture | Ask 1 question max. If user doesn't answer, file to Inbox + `someday`. |
| User says "actually no" right after capture | Offer one-shot undo: "Want me to delete the task I just created?" |
| Label not found in Todoist (e.g. user skipped setup) | Skip the missing label silently — never block the capture. Surface a one-line warning. |
| Todoist API rate limit (HTTP 429) | Back off 1s and retry once. If still 429, queue capture intents to `pending.jsonl` and surface a warning in the routine output. |
| Telegram message exceeds 4096 chars | Truncate `stale_to_decide` and `quick_wins` to top 5 each, append `[… N more — see Todoist]`. See §6.2. |
| Todoist `find-tasks` returns more than one page | Iterate through pagination cursors before computing buckets. Don't trust a single-page response when more than 50 results are possible. |
