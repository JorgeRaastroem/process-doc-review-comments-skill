# Process Document Review Comments

A GitHub Copilot skill for working through review comments against a single source document without altering the reviewed document itself.

This repository defines the execution contract for a structured review-response loop: it loads one document, ingests reviewer comments with their source section, interprets the concern, drafts an evidence-grounded response, allows revision, and optionally exports the accepted responses and follow-ups into a stable Markdown artifact.

## What this skill does

The workflow is designed for architecture, design, specification, and investigation documents where review feedback must be handled in a careful, traceable way.

It helps with:

- processing reviewer comments on a single document
- finding the exact section that a comment refers to
- interpreting the concern without inventing facts
- drafting concise reviewer responses
- revising responses until they are accepted
- capturing follow-up actions and exporting them in a consistent format

## Core behavior

The skill enforces these rules:

- exactly one primary document is in scope for the whole loop
- the primary document is never modified during the workflow
- the source section must be identified and tied to each comment
- response drafting is grounded in the primary document and only limited external references when explicitly allowed
- accepted comments and follow-ups are exported in a deterministic Markdown layout

## Typical workflow

1. Provide a single document path.
2. The skill indexes the document and sets up the review loop.
3. Add a reviewer comment and the relevant document section.
4. The skill interprets the concern and proposes a response.
5. Accept or revise the response until it is ready.
6. Save the accepted responses and any follow-up actions as a Markdown export.

## Interaction model

The skill is built around a stateful review loop with these phases:

- initialize the document
- ingest a comment
- interpret and draft a response
- accept or revise the response
- consolidate follow-ups
- exit and optionally save a review export

The state machine and required behavior are defined in `SKILL.md`.

## Repository contents

- `SKILL.md` — the skill contract, workflow, and state machine
- `references/export-format.md` — the required Markdown export format and validation rules
- `references/` — supporting reference material

## Example usage

Invoke the skill for a document such as:

```text
process-doc-review-comments /path/to/design-doc.md
```

The skill then walks through the review-comment loop and asks for the necessary inputs as needed.

## Export output

When saving, the export follows the format described in `references/export-format.md` and includes:

- a review summary with the source document path and date
- a table of accepted comments and their responses
- a table of follow-up actions and statuses

The default export name is based on the source document stem, for example:

```text
<source-document-stem>-review-comments-and-followups.md
```

## Notes

This skill is intended for processing review feedback, not for modifying the reviewed document itself. It is focused on structured interpretation, grounded response drafting, and exportability for downstream review workflows.
