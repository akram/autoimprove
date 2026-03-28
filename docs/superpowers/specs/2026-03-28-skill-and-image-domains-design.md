# Design: Add `skill` and `image` Domain Types to Autoimprove

**Date**: 2026-03-28
**Status**: Draft
**Approach**: Templates + domain-specific protocol guidance (no core protocol changes)

## Summary

Extend autoimprove from 10 to 12 domain types by adding `skill` (Claude Code skill optimization and creation) and `image` (image generation prompt optimization). Both are subjective-output domains with unique evaluation challenges. Changes are documentation-only additions to existing files — no new protocol mechanisms.

## What Changes

| File | Change |
|------|--------|
| `SKILL.md` | Add detection heuristics, bootstrap threat models, eval-init table entries, two multi-shot examples |
| `references/examples.md` | Add two template sections with complete `improve.md` templates, update types table |
| `README.md` | Update domain count (10 to 12) |
| `CLAUDE.md` | Update domain count |

No changes to: core loop algorithm, `improve.md` format, `references/protocol.md`, stopping conditions, score extraction, or experiment logging.

## Domain 1: `skill` — Claude Code Skill Optimization

### What It Optimizes

The quality of a Claude Code skill — its instructions, examples, trigger description, protocol flow, progressive disclosure, and guard rails. The autoimprove loop modifies skill files, runs eval queries through Claude with the skill loaded, grades outputs against assertions, and keeps improvements.

This domain supports both improving existing skills and building new ones from a minimal skeleton — the loop itself is the creator, iteratively fleshing out the skill scored against eval benchmarks from round one.

### Default Scope

When no scope is specified in `improve.md`, the default is the entire skill directory (SKILL.md, references/, scripts/, assets/). The user can narrow this in `improve.md` if they want to lock certain files.

### Detection Heuristic

```
SKILL.md file + eval/ directory with assertion files or references/ directory → skill
```

This is conservative: a bare `SKILL.md` alone won't trigger `skill` detection (to avoid false positives in projects that happen to use autoimprove). The presence of an eval directory or the references/ structure signals that this is a skill being actively developed.

### Eval-Init

Skills are a subjective-output domain. The eval harness requires:
- **Eval queries**: Realistic user prompts (8-10 should-trigger, 8-10 should-not-trigger)
- **Assertions**: Objectively verifiable expected behaviors for each query
- **Eval script**: Runs queries through Claude with the skill loaded, grades against assertions

The eval-init phase scaffolds the eval script and walks the user through creating the initial query/assertion set. The eval queries and assertions are immutable during the optimization loop.

| Field | Value |
|-------|-------|
| Needs golden set? | Yes — eval queries + assertions |
| Needs eval script? | Yes — runs skill through Claude, grades outputs |

### Bootstrap Threat Model

**When goal = higher skill quality (pass rate on eval queries):**

The agent will overfit to eval queries, add excessive verbosity, game trigger descriptions, or weaken the skill's generalizability.

Tests to generate:
- **Trigger false positives**: Near-miss queries that should NOT trigger the skill still don't
- **Trigger false negatives**: Queries that SHOULD trigger the skill still do
- **Protocol compliance**: Agent follows steps in the correct order when the skill is invoked
- **Hard rule adherence**: Skill's safety constraints are respected in output
- **Token efficiency**: Skill body stays under 500 lines; progressive disclosure is used for the rest
- **Generalization**: Skill works on held-out queries not in the eval set (60/40 train/test split)

### `improve.md` Template

```markdown
# autoimprove: better-<skill-name>

## Change
scope: <skill-directory>/
exclude: eval/

## Check
test: python eval/run_skill_eval.py --check-assertions
test-files: eval/
run: python eval/run_skill_eval.py
score: pass_rate: ([\d.]+)
goal: higher
guard: trigger_false_positive: ([\d.]+) < 0.1
guard: token_usage: (\d+) < 50000
timeout: 5m

## Stop
budget: 4h
target: 0.95
stale: 15

## Instructions

Improve the skill's quality measured by eval query pass rate.
The eval runs test prompts through Claude with the skill loaded
and grades outputs against assertions.

Instruction clarity to try:
- Replace vague directives with specific, actionable steps
- Explain the WHY behind ALWAYS/NEVER rules — models follow reasoning better than commands
- Restructure protocol for clearer decision points and branching
- Simplify verbose sections — shorter often outperforms longer
- Use imperative form ("Run the test" not "You should run the test")

Trigger optimization to try:
- Make description focus on WHEN to trigger, not WHAT the skill does
- Include specific symptoms, contexts, and user phrases as triggers
- Add near-miss disambiguation to reduce false positives
- Slightly pushy descriptions combat under-triggering

Example quality to try:
- One excellent example beats many mediocre ones
- Examples should be complete, from real scenarios, well-commented
- Remove examples that duplicate the same pattern
- Add examples for failure modes agents hit repeatedly

Progressive disclosure to try:
- Move rarely-needed content from SKILL.md body to references/
- Keep SKILL.md under 500 lines — reference files for the rest
- Bundle repeated helper code into scripts/ directory
- Cross-reference instead of duplicating content across files

Guard rails to try:
- Close specific loopholes, not just state rules
- Add rationalization tables for discipline-enforcing rules
- Preempt "spirit vs letter" workarounds with explicit reasoning
- Build from observed agent failure patterns in eval runs

What NOT to try:
- Don't modify eval queries or assertions
- Don't overfit to specific eval cases — skill must generalize
- Don't add model-specific behavior
- Don't exceed 500 lines in SKILL.md without progressive disclosure
- Don't repeat content that exists in referenced skills or files
```

### Multi-Shot Example (for SKILL.md)

```markdown
# autoimprove: better-code-review-skill

## Change
scope: skills/code-review/
exclude: eval/

## Check
test: python eval/run_skill_eval.py --check-assertions
test-files: eval/
run: python eval/run_skill_eval.py
score: pass_rate: ([\d.]+)
goal: higher
guard: trigger_false_positive: ([\d.]+) < 0.1
guard: token_usage: (\d+) < 50000
timeout: 5m

## Stop
budget: 4h
target: 0.95
stale: 15

## Instructions

Improve the code review skill's pass rate on the eval benchmark.
The eval set contains 20 queries: 10 should-trigger (code review requests)
and 10 should-not-trigger (refactoring, debugging, general questions).

The skill currently triggers correctly but produces inconsistent output —
sometimes skips the security check step, sometimes gives overly verbose
feedback on trivial issues.

Instruction clarity to try:
- Make the protocol steps more explicit with decision points
- Add examples of good vs bad review feedback
- Simplify the security checklist — too many items causes skipping
- Add "when to be brief" guidance for trivial findings

Trigger optimization to try:
- Current description triggers on "look at my code" (too broad)
- Add disambiguation: code review vs code explanation vs debugging

What NOT to try:
- Don't modify the eval queries or expected behaviors
- Don't add dependencies on other skills
- Don't make the skill model-specific
```

## Domain 2: `image` — Image Generation Prompt Optimization

### What It Optimizes

The quality of image generation prompts — the text that gets passed to an image generation model (DALL-E, Flux, Stable Diffusion, Midjourney, or any custom pipeline). The autoimprove loop modifies the prompt file, the user's eval harness generates images and scores them, and the loop keeps prompt changes that produce higher-scoring images.

### Generation Method

Agnostic. The template defines the scoring pattern and optimization strategies. The user provides their own `run:` command that calls their generation pipeline (API-based, local model, or custom). Autoimprove doesn't care how images are generated — just that the eval harness takes a prompt and produces a score.

### Evaluation Method

Agnostic. The user can use:
- **Automated metrics**: ImageReward, HPS v2, CLIP Score, LAION Aesthetic Score
- **Golden set + human labels**: Reference prompts with quality baselines (via eval-init)
- **MLLM-as-judge**: GPT-4o or Claude evaluating generated images
- **Any combination**

The template's Instructions section recommends automated scoring approaches and variance mitigation, but doesn't require any specific method.

### Detection Heuristic

```
Python files importing diffusers/replicate/stability_sdk,
  OR Python files with openai image generation calls,
  OR prompts/ directory + eval scripts referencing image_reward/clip_score/aesthetic_score/hpsv2
  → image
```

### Eval-Init

Optional. The domain can work with purely automated metrics (no golden set) or with a golden set of reference prompts and quality baselines.

| Field | Value |
|-------|-------|
| Needs golden set? | Optional — automated metrics can work without one |
| Needs eval script? | Yes — generates images from prompt, scores them |

### Bootstrap Threat Model

**When goal = higher image quality (ImageReward, aesthetic score, human preference):**

The agent will game the scoring metric, bloat prompts with excessive detail, overfit to specific generation parameters, or produce unsafe content.

Tests to generate:
- **Prompt length**: Stays under practical limit (~200 tokens — diminishing returns beyond this)
- **Subject preservation**: Core subject and intent unchanged across iterations
- **Safety**: No NSFW, harmful, or policy-violating content in prompts
- **Pipeline validity**: Prompt produces valid output through the generation pipeline (no errors)
- **Style consistency**: If a target style is specified, it's maintained
- **No model-specific syntax**: Unless the user is explicitly targeting a specific model

### Variance Mitigation Guidance

Image generation is stochastic — the same prompt produces different quality images across runs. The Instructions section recommends:

- Generate N images (4-8) per evaluation, average or median the scores
- Use fixed seeds for reproducible comparison across experiments
- Statistical significance: only keep changes where improvement exceeds baseline standard deviation

These are recommendations for how the user structures their eval script, not protocol requirements.

### `improve.md` Template

```markdown
# autoimprove: better-<task>-images

## Change
scope: prompts/<task>.txt
exclude: eval/, data/

## Check
test: python eval/validate_prompt.py --prompt prompts/<task>.txt
test-files: eval/
run: python eval/run_image_eval.py --prompt prompts/<task>.txt
score: image_reward: ([\d.-]+)
goal: higher
guard: clip_score: ([\d.]+) > 0.20
guard: nsfw_count: (\d+) < 1
timeout: 5m

## Stop
budget: 2h
target: 1.5
stale: 10

## Instructions

Improve the image generation prompt to produce higher-quality images
that better match the intended subject and style.

The eval harness generates multiple images from the prompt and
averages their scores to reduce variance from stochastic generation.

Prompt structure to try:
- Add specific medium/style (oil painting, cinematic photography, 3D render, watercolor)
- Specify lighting (golden hour, studio lighting, rim light, dramatic shadows)
- Add composition cues (rule of thirds, shallow depth of field, close-up, wide angle)
- Include mood/atmosphere (ethereal, dramatic, serene, gritty, cinematic)
- Specify color palette (warm tones, muted pastels, high contrast, monochromatic)
- Add artistic references for style anchoring

Prompt refinement to try:
- Reorder — put the most important detail first
- Replace vague words with specific descriptors
- Add negative context via phrasing ("without text or watermarks")
- Simplify — shorter prompts often outperform verbose ones
- Add few concrete details rather than many abstract ones
- Test: comma-separated keywords vs natural language prose

What NOT to try:
- Don't exceed ~200 tokens (diminishing returns, each word contributes less)
- Don't include model-specific syntax (--ar, (weight:1.5)) unless targeting that model
- Don't include NSFW or harmful content
- Don't hardcode for specific random seeds in the prompt text
- Keep the core subject and intent stable across iterations
```

### Multi-Shot Example (for SKILL.md)

```markdown
# autoimprove: better-product-photos

## Change
scope: prompts/product-hero.txt
exclude: eval/, assets/

## Check
run: python eval/run_image_eval.py --prompt prompts/product-hero.txt --num-images 4 --seeds 42,123,456,789
score: image_reward: ([\d.-]+)
goal: higher
guard: clip_score: ([\d.]+) > 0.22
guard: nsfw_count: (\d+) < 1
timeout: 5m

## Stop
budget: 2h
target: 1.2
stale: 10

## Instructions

Improve the product photography prompt for e-commerce hero images.
The current prompt produces acceptable but bland product shots.
Target: professional-quality product photography with clean backgrounds.

The eval generates 4 images with fixed seeds and averages ImageReward scores.
CLIP score guards prompt-image alignment.

Prompt structure to try:
- Specify photography style (commercial product photography, editorial, lifestyle)
- Lighting: softbox, natural window light, gradient background
- Add material descriptors for the product (matte, glossy, textured)
- Composition: centered, negative space, shallow DOF with bokeh
- Color: neutral background, complementary accent colors

Refinement to try:
- Start with the product description, then add environment/style
- "Professional product photography" as an anchor phrase
- Specify what the background should be (not just "clean background")
- Add one aspirational reference ("as seen in Apple product pages")

What NOT to try:
- Don't add humans or lifestyle elements (product-only shots)
- Don't specify exact pixel dimensions in the prompt
- Don't reference specific photographer names
- Keep prompt under 150 tokens for product shots
```

## Changes Summary

### SKILL.md Additions

1. **Init Templates section** (~line 528): Add two detection heuristics
2. **Eval-init domain table** (~line 266): Add `skill` and `image` rows
3. **Bootstrap threat models** (~line 167): Add two new goal-aware threat models
4. **Multi-shot examples** (after existing Example 10): Add Example 11 (skill) and Example 12 (image)

### references/examples.md Additions

1. **Types table** (line 1-19): Add `skill` and `image` rows
2. **New section**: `## skill — Claude Code Skill Optimization` with full template
3. **New section**: `## image — Image Generation Prompt Optimization` with full template

### README.md Changes

1. Update "Supports 10 domain templates" to "Supports 12 domain templates" in the auto-guided setup section (~line 154)
2. Add `skill` and `image` to the template list

### CLAUDE.md Changes

1. Update "10 supported domains" to "12 supported domains"
2. Add `skill` and `image` to the domain list
