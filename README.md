# Prompt

`prompt` is a legacy prompt and skill reference repository.

Its top-level `SKILL.md` is named `prompt-forge` and contains a long-form
prompt-generation engine with architect, red-team, refiner, few-shot, and
quality-check phases. This repository is useful as a reference source, but it
is not the current compact `prompt-forge` package.

## What This Is

Use this repository to inspect or reuse:

- long-form system-prompt generation instructions,
- adversarial prompt hardening phases,
- attack-vector reference material,
- architecture templates,
- few-shot design patterns,
- prompt quality checklists.

## What This Is Not

- It is not a runtime application.
- It is not a library with install/build commands.
- It is not a collection of user prompts.
- It is not the canonical compact `prompt-forge` package.
- It is not intended to replace requirement specs, task plans, or decision
  analysis skills.

## Status

Reference repository on branch `master`.

The material is still useful, but this repo should be read as the older,
heavier prompt-forge source. For an active, smaller skill package with
`agents/openai.yaml`, use the `prompt-forge` repository.

## Relationship To `prompt-forge`

```text
prompt
  older, longer reference package
  includes detailed attack vectors, templates, and quality checklist

prompt-forge
  newer, compact installable skill package
  keeps the prompt architecture workflow with a smaller surface
```

When migrating content from `prompt` to `prompt-forge`, preserve the useful
workflow gates and references. Do not blindly copy all prose.

## Repository Layout

```text
.
  README.md
  SKILL.md
  references/
    attack_vectors.md
    architecture_templates.md
    fewshot_patterns.md
    quality_checklist.md
```

## Trigger Boundary

This skill is relevant when the user asks to:

- write a system prompt,
- design an AI assistant prompt,
- optimize or harden an existing prompt,
- create an AI persona,
- adapt a prompt for GPT, Claude, Gemini, or another model,
- add few-shot examples or negative constraints.

It is not the right tool when the primary deliverable is code, a task DAG, a
requirements document, or a business decision analysis.

## Usage

Clone the reference repository:

```bash
git clone https://github.com/jupiternaut/prompt.git
cd prompt
```

Inspect the skill and references:

```bash
sed -n '1,180p' SKILL.md
sed -n '1,120p' references/attack_vectors.md
sed -n '1,120p' references/quality_checklist.md
```

If installing manually into an agent skill directory, keep `SKILL.md` and the
`references/` directory together so relative links continue to work.

## Tests And Checks

There is no automated test suite. Use the checklist and references directly:

```bash
sed -n '1,220p' references/quality_checklist.md
```

For a manual quality check, confirm that the final prompt has:

```text
role or task definition
input handling rules
workflow
negative constraints
output contract
edge-case behavior
model-specific formatting when needed
```

## Related Repositories

- `prompt-forge`: active compact prompt-architecture skill package.
- `Claude-skills`: broader adversarial-analysis and browser-ops skill pack.

## Maintainer

jupiternaut / Geng Ruofan
