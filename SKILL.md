---
name: feat-pig
description: Autonomously implement every unbuilt feature in a markdown feature-list file, one per fresh-context iteration, until none remain. Takes the markdown file as its argument (tag it with @).
argument-hint: "@feature-list.md"
allowed-tools: Bash, Read, Grep, Glob
---

# feat-pig — headless feature-implementation loop

Drive a loop that implements features from `$ARGUMENTS` one at a time. Each iteration spawns a fresh `claude -p` so the working context stays small across the whole run.

The argument is expected to be an `@`-tagged file reference (e.g. `/feat-pig @FEATURES.md`). The leading `@` is stripped before resolving the path — a bare path still works for backward compat.

## Canonical feature-file format

The loop's counters and per-iteration prompt both key off this shape — no heuristics, no tolerance:

- Every feature is a `### N. Title` heading (numbered, unique across the file).
- A feature is **done** iff its `### ` heading contains the literal string `DONE` (e.g. `### 3. Engagement tab — DONE`).
- Priority groups are `## High priority`, `## Medium priority`, `## Low priority` — exactly these three strings. The inner prompt walks them in order.
- No other `### ` headings exist in the file. Section labels, commentary, architecture notes, and summaries live at `##` or below-`###` levels only.

Non-conforming files are normalised in step 2 before the loop runs. The counters can't tell a stray `### Phase 1` section label from a real feature heading, so the skill fixes the file instead of hardening the counters.

## What you (the model running this skill) do

1. **Validate the argument.** If `$ARGUMENTS` is empty, stop and tell the user to tag the feature file with `@` (e.g. `/feat-pig @FEATURES.md`). Strip any leading `@` from the argument before treating it as a path. If the resolved file doesn't exist, stop and tell the user.
2. **Normalise the feature-file format.** Read the file. If it already matches the canonical format above exactly, skip this step. Otherwise rewrite it in place so the loop can address every feature:
   - **Bullet-item features** (`- **Title** — body…` under a heading) → promote each bullet to a `### N. Title` heading with the bullet body verbatim as the entry body. Preserve any existing `— DONE` marker.
   - **Sub-priority `### ` labels** (e.g. `### Must-have`, `### Nice-to-have`, `### P0`, `### Critical` used as section headers, not as numbered features) → delete the label and treat its children as features under the mapped canonical priority section. Mapping:
     - Critical / Must-have / P0 / Blocker → **High**
     - Should-have / P1 / Important → **Medium**
     - Nice-to-have / P2 / Optional / Wishlist → **Low**
   - **Phase-based `## ` sections** (e.g. `## Phase 1`, `## Phase 2 — Manual engagement`, `## MVP`, `## Later`) → keep the prose/context as a paragraph, but move the features inside them into the canonical priority sections (earlier phases → higher priority by default; if the phase labels specify priority explicitly, honour it).
   - **Non-canonical `## ` sections** (e.g. `## Scope & non-goals`, `## Architecture`, `## Roadmap`) → keep the prose/context as a paragraph, but move the features under them into one of the three canonical priority sections. Pick the priority from the section name's intent; if genuinely ambiguous, ask the user which priority to use before rewriting.
   - **Out-of-scope / non-goal items** → keep them under a `## Non-goals` or `## Out of scope` section as prose, but do NOT give them `### ` headings — the loop would otherwise try to implement them.
   - **Non-feature `### ` sub-headings inside reference sections** (e.g. `### Rate limits`, `### 3.1 Query construction`, `### x_engagements_xbot`, `### Parameters`, `### Endpoints`) → these are document-structural sub-sections, not features. Demote every one to `#### ` (or deeper) so the counter doesn't see them. This applies to ANY `### ` heading that is NOT a numbered `### N. Title` feature entry living under a `## <priority> priority` section. No exceptions — even a single stray `### ` inflates the loop's counts and can waste iterations after the real work is done.
   - **Renumber** `### N.` entries sequentially across the file once restructuring is done, so no two features collide on a number.
   - **Lossless on bodies**: do not rephrase, summarise, or drop existing feature descriptions, acceptance criteria, or DONE postmortems. Only restructure headings and priority grouping.
   - **Verify before kicking off the loop.** After rewriting, run `grep -nE '^### ' <file>` and confirm that (a) every result sits under a `## High priority` / `## Medium priority` / `## Low priority` section, and (b) every result matches the shape `### N. Title` (a number, a dot, a title). If any line fails either check, it's a stray sub-heading — demote it to `#### ` and re-verify. Do this check even when you believe the rewrite is clean; a single miss silently costs iterations later.
   - Tell the user in one sentence what you changed (e.g. "Normalised 4 phase-grouped features under High/Medium priority; preserved all acceptance criteria.") before kicking off the loop. Do not ask for confirmation — the user reviews the diff before committing.
3. **Pick model + effort for the inner `claude -p` sessions.** Subprocesses do NOT inherit the parent session's `/model` or `/effort` overrides — those live only in the parent's runtime. You must pass them explicitly. Defaults if the user hasn't said otherwise: `opus` + `xhigh`. If the user recently mentioned a different model/effort for this work (e.g. "I'm running sonnet today"), use that. If genuinely unsure, ask in one sentence. Export `CLAUDE_FEAT_MODEL` and `CLAUDE_FEAT_EFFORT` before kicking off the loop — the script reads them.
4. **Kick off the loop script below via Bash with `run_in_background: true`.** It writes progress to `.feat-pig.log` in the same directory as the feature file.
5. **Monitor** the log (tail it periodically, or use `Monitor` on the bash process). Do NOT sleep-poll tightly — the Monitor tool notifies on new lines.
6. When the loop exits, **read the final feature file** and report to the user: how many features were shipped this run, what remains unbuilt, and any newly-discovered features that were appended.

Do not commit anything. The user reviews and commits themselves.

## The loop

Run this exactly — do not inline-edit the prompt body unless the user asks. `FEATURES_FILE` must be absolute.

```bash
set -u
RAW_ARG="${ARGUMENTS_PATH#@}"   # strip leading @ if the user tagged the file
FEATURES_FILE="$(cd "$(dirname "$RAW_ARG")" && pwd)/$(basename "$RAW_ARG")"
LOG="$(dirname "$FEATURES_FILE")/.feat-pig.log"
MODEL="${CLAUDE_FEAT_MODEL:-opus}"
EFFORT="${CLAUDE_FEAT_EFFORT:-xhigh}"
: > "$LOG"

count_unbuilt() { grep -E '^### ' "$FEATURES_FILE" 2>/dev/null | grep -cv 'DONE' | tr -d '\n'; }
count_total()   { grep -cE '^### ' "$FEATURES_FILE" 2>/dev/null | tr -d '\n'; }
state_line()    { printf '%s total, %s unbuilt' "$(count_total)" "$(count_unbuilt)"; }

# Cap scales with actual work: 2× the initial unbuilt count, + buffer for
# sub-features rule 4 may append. Minimum 5 to keep trivial runs sane.
UNBUILT_START=$(count_unbuilt)
MAX_ITER=$(( UNBUILT_START * 2 + 5 ))
if [ "$MAX_ITER" -lt 5 ]; then MAX_ITER=5; fi
echo "[$(date +%H:%M:%S)] Start: $(state_line). MAX_ITER=$MAX_ITER, model=$MODEL, effort=$EFFORT" >> "$LOG"

prev_unbuilt=$UNBUILT_START
prev_total=$(count_total)
stuck_streak=0

for i in $(seq 1 $MAX_ITER); do
  if [ "$(count_unbuilt)" -eq 0 ]; then
    echo "[$(date +%H:%M:%S)] All features marked DONE after $((i-1)) iteration(s). Stopping." >> "$LOG"
    break
  fi
  {
    echo ""
    echo "=========================================="
    echo "[$(date +%H:%M:%S)] iteration $i — $(state_line)"
    echo "=========================================="
  } >> "$LOG"

  claude -p "You are implementing exactly ONE feature from the markdown file at $FEATURES_FILE.

Rules:
1. Read $FEATURES_FILE. The highest-priority unbuilt feature is the first '### ' heading under '## High priority' (then Medium, then Low) that does NOT contain the word DONE.
2. Read the feature entry in full AND any code it references. Understand the full scope — read neighbouring code, existing patterns, and anything marked as 'done' in earlier entries that this feature may depend on. Do not pattern-match a shallow implementation.
3. Implement the feature. Create new files when the feature calls for it; edit existing files when the feature extends them. Follow the codebase's existing conventions (naming, module shape, auth patterns, error handling). Do NOT introduce abstractions, refactors, or additional features beyond what the entry specifies.
4. Scope discipline: if you discover that this feature depends on work that wasn't listed, or you spot a genuine gap/sub-feature while implementing, append it as a new '### N. <title>' entry under the appropriate '## <priority> priority' section of $FEATURES_FILE — do NOT build it this iteration. If the dependency is a hard blocker, append it and stop (still mark nothing DONE); the outer loop will pick it up next iteration.
5. Self-review pass: re-read your diff. Does it match what the entry described? Does it break any existing feature (especially ones already marked DONE)? If yes, fix in the same iteration.
6. Update the feature's entry in $FEATURES_FILE: append ' — DONE' to its '### ' heading, and rewrite the body to describe (a) what was built, (b) which files/functions/routes changed or were added, and (c) anything a reviewer should pay attention to. Keep it concise — match the style of existing DONE entries.
7. Do NOT commit. Do NOT touch any other unbuilt feature. Do NOT run the app or start servers. Type-checking and linting commands are fine.
8. End with a one-line summary: 'Built: <feature title>'." \
    --model "$MODEL" \
    --effort "$EFFORT" \
    --permission-mode bypassPermissions \
    --max-turns 80 \
    >> "$LOG" 2>&1 || {
      echo "[$(date +%H:%M:%S)] iteration $i FAILED (claude exit $?). Stopping." >> "$LOG"
      break
    }

  # Stuck detection: iteration ran but neither flipped a feature to DONE nor
  # appended a new one. One retry allowed; bail on the second consecutive stall.
  cur_unbuilt=$(count_unbuilt)
  cur_total=$(count_total)
  if [ "$cur_unbuilt" -ge "$prev_unbuilt" ] && [ "$cur_total" -le "$prev_total" ]; then
    stuck_streak=$((stuck_streak + 1))
    echo "[$(date +%H:%M:%S)] iteration $i made no progress (stuck_streak=$stuck_streak)" >> "$LOG"
    if [ "$stuck_streak" -ge 2 ]; then
      echo "[$(date +%H:%M:%S)] No progress for 2 iterations. Stopping." >> "$LOG"
      break
    fi
  else
    stuck_streak=0
  fi
  prev_unbuilt=$cur_unbuilt
  prev_total=$cur_total
done

echo "" >> "$LOG"
echo "[$(date +%H:%M:%S)] Loop done. Final state: $(state_line)" >> "$LOG"
```

Before running it, in the same bash invocation:
- `export ARGUMENTS_PATH="$ARGUMENTS"` — path to the feature file.
- Optionally `export CLAUDE_FEAT_MODEL=...` and `export CLAUDE_FEAT_EFFORT=...` if the user wants a non-default model/effort for the inner sessions. Defaults are `opus` + `xhigh`. Valid effort levels: `low, medium, high, xhigh, max`.

## Safety rails

- `MAX_ITER` auto-sizes to `unbuilt_at_start * 2 + 5` (min 5) — enough headroom for one retry per feature plus a few rule-4 appends, without blind over-spinning.
- **Stuck detection**: if an iteration neither flipped a feature to `DONE` nor appended a new `### ` entry, it counts as no-progress. One retry is allowed; two consecutive stalls stop the loop. Prevents burning the cap on a single unimplementable feature.
- `--max-turns 80` — feature work typically involves more edits than a bug fix, so the per-iteration turn budget is higher than fix-bugs' 60.
- `--permission-mode bypassPermissions` is required for autonomy. Every inner iteration is sandboxed to one feature and cannot commit.
- If `claude -p` returns non-zero, the loop stops — the user investigates via the log.
- The log lives next to the feature file as `.feat-pig.log`. Gitignore it.

## Differences from fix-bugs

Keep these in mind when normalising or triaging:

- Features usually ship in **phases** (e.g. "Phase 1 — Discovery only", "Phase 2 — Manual engagement"). Phase sections must be flattened into priority sections during normalisation, or the loop will skip them.
- Feature entries often have embedded sub-lists of acceptance criteria or endpoints. Keep those as prose/sub-bullets in the body — they are NOT separate features.
- **Out-of-scope** items in feature docs are common (explicit non-goals). Never promote them to `### ` headings; keep them under a `## Non-goals` or `## Out of scope` section.
- Feature bodies can be long (design rationale, API contracts, schema tables). The inner session needs the full body, so never truncate during normalisation.
