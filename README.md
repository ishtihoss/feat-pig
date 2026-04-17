# feat-pig

A [Claude Code](https://claude.com/claude-code) skill that autonomously implements every feature in a markdown feature-list, one per fresh-context iteration, until none remain.

You write a plan. feat-pig builds it — feature by feature, top to bottom.

## What it does

Drop a markdown file listing your features. Run `/feat-pig @FEATURES.md`. The skill:

1. **Normalises** your file into a canonical format (flattens phases, demotes stray sub-headings, groups features under `## High / Medium / Low priority`).
2. **Loops**, spawning a fresh `claude -p` subprocess for every iteration — so each feature gets a clean 200k-token context instead of a bloated one.
3. Each iteration picks the highest-priority unbuilt feature, implements it end-to-end, marks it `— DONE` in the file, and exits.
4. Stops when every feature is done, or after a sensible cap, or if two iterations in a row make zero progress.

Fresh context per iteration is the whole point. Long-running feature work in a single session drifts, forgets, and hallucinates; feat-pig sidesteps that by design.

## Why "feat-pig"?

It eats features. One at a time. Keeps eating until the trough is empty. 🐖

## Install

Drop `SKILL.md` into your personal Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/feat-pig
curl -fsSL https://raw.githubusercontent.com/ishtihoss/feat-pig/main/SKILL.md \
  > ~/.claude/skills/feat-pig/SKILL.md
```

Restart Claude Code. You should see `feat-pig` in your skills list.

## Usage

```
/feat-pig @FEATURES.md
```

The `@` tag is how Claude Code resolves file references — same syntax you use to attach any file to a prompt.

If your file is already in canonical format, the loop starts immediately. Otherwise, the skill rewrites it in place first (you review the diff before committing).

## Canonical format

```markdown
# My Project — Feature Plan

<any prose, architecture notes, reference tables live at ## level>

## Non-goals

- Things we are deliberately NOT building
- These stay as prose; no ### headings

## High priority

### 1. First feature title

Full description. Acceptance criteria as sub-bullets. API contracts,
schema tables, whatever the implementer needs.

### 2. Second feature title — DONE

Entry gets rewritten by the loop to describe what shipped.

## Medium priority

### 3. ...

## Low priority

### 4. ...
```

Rules the counters rely on:

- Every feature is `### N. Title` — numbered, unique across the file.
- Feature is done iff its `### ` heading contains the literal string `DONE`.
- Priority groups are exactly `## High priority`, `## Medium priority`, `## Low priority`.
- No other `### ` headings exist. Document-structural sub-sections must be `####` or deeper.

The skill's normalisation step handles all common non-canonical shapes (phased rollouts, bulleted feature lists, sub-priority labels like "Must-have" / "P0", reference tables with `### Rate limits`-style sub-headings) — so you don't have to hand-format before running.

## How each iteration works

For the picked feature, the inner `claude -p` session:

1. Reads the full entry and the code it references. Reads neighbouring patterns so the implementation matches house style.
2. Creates new files / edits existing ones.
3. Self-reviews its own diff against the entry's acceptance criteria and any `— DONE` features it might have broken.
4. Rewrites the feature's entry body to describe what it built (files changed, routes added, reviewer notes). Appends `— DONE` to the heading.
5. If it discovers a dependency or sub-feature mid-build, appends it as a new `### N.` entry instead of ballooning the current iteration.

The inner session never commits, never starts servers, never touches other unbuilt features.

## Configuration

Defaults to `opus` + `xhigh` effort for inner sessions. Override per-run:

```bash
CLAUDE_FEAT_MODEL=sonnet CLAUDE_FEAT_EFFORT=high /feat-pig @FEATURES.md
```

Valid effort levels: `low, medium, high, xhigh, max`.

## Safety rails

- `MAX_ITER = 2 × initial_unbuilt + 5` (min 5). Enough headroom for one retry per feature plus a few scope-discovery appends.
- Two consecutive no-progress iterations → loop exits. Prevents burning the cap on a single unimplementable feature.
- `--permission-mode bypassPermissions` on inner sessions (required for autonomy). Scope is sandboxed to one feature per iteration.
- **Loop never commits.** You review the final diff and commit yourself.
- Progress log lives next to the feature file as `.feat-pig.log`. Gitignore it.

## When to use it

Good fit:
- A 10–30-feature plan you've already thought through and want built.
- Greenfield work where the plan is mostly independent features.
- Long weekends / overnight runs where you want to wake up to a working app.

Bad fit:
- Exploratory work where the design keeps shifting.
- Single-feature tasks (just use a normal session).
- Anything needing continuous human judgement between features.

## Known limitations

- Feature bodies need to be **specific**. "Add user authentication" gets a shallow implementation. "Add email/password auth with bcrypt, `/auth/login` + `/auth/register` routes, JWT in httpOnly cookie, middleware at `lib/auth.js`" gets a real one.
- Cross-feature consistency relies on each iteration reading existing DONE entries. If you describe a shared pattern only in feature 1's body, feature 7 may miss it — mention shared patterns in the file's prose sections.
- The loop assumes a single canonical file. Multi-file feature specs aren't supported; inline or link them.

## License

MIT.
