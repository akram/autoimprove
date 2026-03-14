# autoimprove

Autonomous optimization loop for AI agents. Point it at files to change, a command to check, and a number to improve — then walk away.

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) and [Shopify Liquid PR #2056](https://github.com/Shopify/liquid/pull/2056).

## How it works

```
LOOP:
  Agent modifies files → git commit → run check → extract score
  → improved? keep : discard → log → repeat
```

No human in the loop. The agent decides what to try based on domain instructions and past experiment history.

## Quick start

1. Install the skill in Claude Code:

```bash
claude mcp add-skill https://github.com/zanetworker/autoimprove.git
```

2. Create an `improve.md` in your project:

```markdown
# autoimprove: make-it-faster

## Change
files: src/handler.go
context: test/, benchmark/

## Check
run: go test ./... && go test -bench=. -benchmem
score: ns/op:\s+([\d.]+)
goal: lower
timeout: 3m

## Stop
budget: 4h
stale: 15

## Instructions

Reduce allocations and avoid unnecessary work in hot paths.
Try buffer reuse, fast-path patterns, and byte-level operations.
Don't change the public API.
```

3. Run it:

```bash
# Inside Claude Code
/autoimprove

# Headless (overnight)
claude -p "run /autoimprove on improve.md" --allowedTools bash,read,write,edit
```

## The `improve.md` format

A single markdown file — part config, part prompt:

| Section | Purpose |
|---------|---------|
| `## Change` | Files the agent can modify + read-only context |
| `## Check` | Command to run, score extraction, goal direction, timeout |
| `## Stop` | When to quit: budget, rounds, target, stale |
| `## Agent` | Provider and model for headless mode |
| `## Instructions` | Free-form domain guidance for the agent |

## Agent-agnostic

The skill works inside Claude Code, but the format is agent-agnostic. Export a standalone protocol:

```bash
/autoimprove --export    # generates program.md
```

Then point any agent at it:

```bash
codex -p "follow program.md"
gemini -p "follow program.md"
aider --message "follow program.md"
```

## Templates

Scaffold an `improve.md` for your domain:

```bash
/autoimprove init              # auto-detect
/autoimprove init --type perf  # performance
/autoimprove init --type ml    # ML training
/autoimprove init --type docker
/autoimprove init --type k8s
/autoimprove init --type prompt
/autoimprove init --type sql
/autoimprove init --type frontend
/autoimprove init --type ci
```

## Experiment log

Results are saved to `.autoimprove/experiments/` as structured JSON:

```
.autoimprove/
  baseline.json
  experiments/
    001-reduce-allocations.json
    002-cache-integer-tostring.json
```

View them with:
```bash
# Inside Claude Code — the agent reads these automatically
# Outside — they're just JSON files, pipe through jq
ls .autoimprove/experiments/*.json | xargs jq '{id, title, status, score, delta_pct}'
```

## How is this different from AutoML?

AutoML searches a predefined parameter grid. Autoimprove lets an AI agent make creative, open-ended changes — rewrite algorithms, delete code, try novel approaches. The search space is unbounded.

## License

MIT
