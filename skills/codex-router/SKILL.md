---
name: codex-router
description: "Route implementation work to Codex CLI; Claude specs, reviews, verifies."
---

# Codex Router

Claude Code sessions only. Codex/other harnesses: skip; never self-delegate.

Rationale: Claude Fable 5 is the most intelligent model but is not available as part of subscription (consumes usage credits; expensive);  Claude Opus/Sonnet included in subscription with weekly limits but underperform in certain use cases; Codex subscriptipn has generous weekly token limits. GPT-5.5+ is usually the better and faster model at writing/implementing code; Claude wins at ergonomics — judgment, design, spec-writing, review, orchestration. So Codex types, Claude thinks and verifies.

## Route

Delegate to Codex (default for bulk/mechanical work):

- implementation from a frozen spec; refactors; mechanical migrations
- bug fixes with a known repro; test writing; coverage fills
- CI fixes, dependency bumps, scripts/tooling
- bulk codebase exploration where raw reading ≫ the answer

Keep in Claude:

- design, API design, architecture, naming, UX judgment
- tasks where writing the spec IS the work (ambiguity = design)
- tiny edits: a single obvious change, roughly under 20–30 lines — delegation overhead loses. Size is the tiebreaker, obviousness is the gate: if the change forces any decision it's not tiny regardless of line count.
- bug fixes with NO reliable repro — diagnosis first = ambiguity = Claude; freeze the cause, then delegate the fix
- anything needing session tools: MCP (browser/computer-use/chronicle), 1Password, secrets
- destructive/irreversible ops, releases, pushes, GitHub mutations — Claude-side per git rules
- review of Codex output — never delegated, never skipped

Mixed (design + build in one ask): Claude designs first, freezes the spec, delegates the build-out. A third first-class route, not an afterthought — most "design X and implement it" prompts land here.

Heuristic: prompt reads as a work order → delegate; writing it forces decisions → design, Claude.

## Invoke

Prompt via temp file, never inline quoting:

```bash
P=$(mktemp); cat >"$P" <<'EOF'
<goal, repo + key paths, constraints ("don't touch X"), non-goals, proof expected, output shape>
EOF
command codex exec --sandbox workspace-write -C <repo> \
  -m gpt-5.6-sol \
  -c model_reasoning_effort="high" \
  -c approval_policy="on-request" \
  -c approvals_reviewer="auto_review" \
  -o /tmp/codex-last.md - <"$P" 2>/dev/null
```

- Model default: `gpt-5.6-sol`, effort `high` — pin explicitly; don't rely on user config.
- `command codex` bypasses the interactive zsh wrapper;
- stderr suppressed (thinking noise bloats context); drop `2>/dev/null` only to debug a failing run
- read `-o` file for the result; don't parse the JSONL stream
- long runs: Bash run_in_background, read `-o` file on exit; don't kill quiet runs <30 min
- parallel independent tasks OK: separate repos/dirs, separate `-o` files
- outside a git repo add `--skip-git-repo-check`

Follow-up fixes — cheaper than fresh runs, keeps context. `resume` takes no `-C`, `--sandbox`, or `--ask-for-approval`: run from the repo dir and pass sandbox/approval as `-c` config keys:

```bash
(cd <repo> && command codex exec resume --last \
  -c sandbox_mode="workspace-write" -c approval_policy="on-request" -c approvals_reviewer="auto_review" \
  -o /tmp/codex-last.md - <"$P2" 2>/dev/null)
```

## Prompt contract

Codex starts with zero session context. Every prompt: goal, exact repo/paths, constraints, non-goals, proof expected (exact test command), output shape ("report files changed + test output"). Spec quality decides success.

## Verify (Claude, always)

- `git status -sb` + read the full diff; judge like a contributor PR
- run focused tests yourself or demand proof output; Codex claims are advisory
- iterate via resume; after 2 failed rounds, take over and do it directly
- normal closeout still applies

## Economics

Win = generation + exploration tokens moved to Codex; Claude spends only on spec + diff review. Don't ping-pong trivia through delegation; don't re-read what Codex already summarized.