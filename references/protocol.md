# Autoimprove Protocol

This is a self-contained protocol for autonomous optimization. Any AI agent can follow it.

## How to Use This File

You are an autonomous optimization agent. Your job is to improve a measurable score by modifying specific files, testing the result, and keeping only changes that improve the score. You never ask for human input. You run until a stopping condition is met.

## Setup

1. Read `improve.md` in the current directory
2. Parse the structured headers to extract:
   - `Change.files` — the ONLY files you may modify
   - `Change.context` — files you may read for reference
   - `Check.run` — the command to evaluate your changes
   - `Check.score` — how to extract the score from output
   - `Check.goal` — whether lower or higher is better
   - `Check.timeout` — max time per experiment
   - `Stop.*` — when to stop the loop
3. Create `.autoimprove/` directory and `.autoimprove/experiments/` subdirectory
4. Run the check command on the unmodified codebase to establish a baseline
5. Save the baseline score and current git commit SHA to `.autoimprove/baseline.json`

## Loop

Repeat until a stopping condition is met:

1. **Plan**: Read past experiments from `.autoimprove/experiments/`. Review what worked, what failed, and why. Propose a new change that you have not tried before.

2. **Implement**: Modify ONLY the files listed in `Change.files`. Make a focused, minimal change.

3. **Commit**: `git add <changed files> && git commit -m "autoimprove: <short description>"`

4. **Evaluate**: Run the check command. If it times out, kill it and treat as failure.

5. **Score**: Extract the score from stdout using the extraction method in `Check.score`.

6. **Decide**:
   - If the check command failed (non-zero exit or timeout): `git reset --hard HEAD~1`, log as "error"
   - If the score improved: keep the commit, update the baseline, log as "kept"
   - If the score did not improve: `git reset --hard HEAD~1`, log as "discarded"

7. **Log**: Save experiment JSON to `.autoimprove/experiments/NNN-slug.json` with this schema:
   ```json
   {
     "id": <number>,
     "title": "<short description>",
     "status": "kept | discarded | error",
     "commit": "<sha if kept, null if discarded>",
     "score": <number or null if error>,
     "baseline_score": <number>,
     "delta_pct": <number or null>,
     "duration_s": <number>,
     "reasoning": "<why you tried this>",
     "files_changed": ["<paths>"],
     "timestamp": "<ISO 8601>"
   }
   ```

8. **Check stopping conditions**:
   - `budget`: total wall-clock time since loop started > budget → stop
   - `rounds`: experiment count >= rounds → stop
   - `target`: score reached target value → stop
   - `stale`: consecutive non-improving experiments >= stale → stop

## Rules

- NEVER modify files outside `Change.files`
- NEVER modify the check command, scoring, or evaluation
- NEVER skip the git commit before evaluating
- ALWAYS log every experiment including failures
- ALWAYS read past experiments before proposing changes
- ALWAYS leave the git tree clean after discarding
- Complexity must justify itself — a marginal improvement with ugly code is not worth keeping
- Deleting code for equal results IS worth keeping
- When stuck after several failures, try a fundamentally different approach instead of variations on the same idea

## Domain Instructions

Read the `## Instructions` section of `improve.md` for domain-specific guidance on what to try and what to avoid.
