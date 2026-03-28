# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Autoimprove is a Claude Code plugin that implements an autonomous optimization loop. Point it at files to change, a command to check, and a number to improve — then walk away. This is a **documentation-only project** with no source code, build system, or dependencies.

## Repository Structure

```
autoimprove/
├── .claude-plugin/
│   └── plugin.json                      # Plugin manifest
├── skills/
│   └── autoimprove/
│       ├── SKILL.md                     # Skill definition (core of the project)
│       └── references/
│           ├── examples.md              # improve.md templates for all 12 domains
│           └── protocol.md              # Agent-agnostic standalone protocol
├── README.md                            # User-facing docs and quick start
├── CLAUDE.md                            # This file
└── docs/
    └── index.html                       # Reveal.js presentation
```

- `skills/autoimprove/SKILL.md` — The skill definition. Defines `/autoimprove` commands, the `improve.md` format, the optimization loop protocol, bootstrap/eval-init phases, 12 domain templates (perf, ml, docker, k8s, prompt, sql, frontend, ci, automl, rag, skill, image), and hard safety rules.
- `skills/autoimprove/references/protocol.md` — Standalone agent-agnostic protocol (output of `/autoimprove --export`). Any AI agent can follow it without the skill.
- `skills/autoimprove/references/examples.md` — Complete `improve.md` templates for all 12 domains.
- `.claude-plugin/plugin.json` — Plugin manifest for installation via `claude plugin install`.

## Editing Guidelines

- **Keep files in sync**: `SKILL.md`, `README.md`, and `references/protocol.md` all describe the same protocol. Changes to the loop, `improve.md` format, or commands must be reflected in all three.
- `references/examples.md` templates must match the `improve.md` format defined in `SKILL.md`.
- The presentation uses tight CSS to prevent slide overflow — keep code blocks and content brief.

## Architecture Notes

- **Two-phase design**: Tests are mutable during bootstrap (setup), immutable during the optimization loop. These phases never mix.
- **Score extraction**: Three formats — `SCORE: <number>` convention, regex with capture group, or jq expression.
- **Guard metrics**: Secondary metrics that must not regress, even if the primary score improves.
- **Experiment supersession**: The `supersedes` field in experiment JSON marks earlier approaches as obsolete so the agent doesn't retry them.
- **Scope locking**: Natural language scope is resolved to concrete files, confirmed by the user, then locked for the entire loop.
