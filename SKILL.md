---
name: process-doc-review-comments
argument-hint: "[document-path]"
description: |
  Processes review comments against one source document through an interactive interpretation,
  response, revision, and export loop. Use when the user asks to "process document review comments",
  "start a review comment loop", "interpret reviewer feedback", "draft comment responses",
  "work through architecture review comments", "respond to document comments", or
  "save review comments and answers".
---

# Document Comment Response Loop

**Execution:** The implicit model-neutral execution contract applies.

## Overview

Run a structured review-comment loop against exactly one primary document. Collect each comment and
its exact document section, ground the interpretation in the primary document, revise the proposed
response until accepted, and optionally export accepted responses plus consolidated follow-ups.

## When to Use

- Process architecture, design, specification, or investigation-document review comments.
- Interpret reviewer feedback and draft a response without modifying the reviewed document.
- Work through several comments while preserving accepted answers and follow-up actions.
- Export the completed review discussion in a stable Markdown format.

## When NOT to Use

- Do not process Azure DevOps PR threads - use `drive-pr-to-green`.
- Do not review a code diff - use `texas-review`.
- Do not author or redesign the primary document - use `architect-and-design`.
- Do not compare or jointly review multiple primary documents - start a separate invocation for each.

## Input Contract

Accept exactly one primary document.

1. Use `$ARGUMENTS` as the document path when it contains one path.
2. When no path is supplied, ask for one document path.
3. Reject multiple paths. Ask the user to choose one primary document.
4. Resolve and retain the canonical primary-document path for the entire loop.
5. Reject attempts to replace or add another primary document after initialization. Offer to exit
   and start a new invocation instead.
6. Treat pasted reviewer text as comment context, never as another primary document.

**IMPORTANT:** Never modify the primary document during this workflow.

### Direct document references

Read a referenced document only when all conditions are true:

1. The primary document literally names or links the reference.
2. The reference is needed to verify a claim in the current comment.
3. The reference resolves directly from the primary document.
4. The reference depth is exactly one.

Do not follow links found inside a referenced document. Record which direct reference supported the
response. A reference supplies evidence but never becomes a second primary document.

## State

Use session-scoped structured storage so the loop survives context compaction. Store:

- primary path and source fingerprint;
- loop status and pending gate;
- next stable comment and follow-up identifiers;
- section, verbatim comment, interpretation, response revisions, accepted response, and status;
- follow-ups and their source comment identifiers;
- direct references consulted.

Do not depend only on conversation history. Mutate no state when a form is cancelled or declined.

## Workflow

### Phase 1: Initialize the document

1. Validate the single-document input contract.
2. Read the primary document and index its headings, diagrams, and tables.
3. For a large document, read headings first and retrieve sections on demand.
4. Record the source fingerprint so source drift can be disclosed during export.
5. Enter `AWAITING_COMMENT`.

**Success criteria:** One canonical primary document is loaded and the section index is available.

### Phase 2: Ingest one comment

Use a structured user gate with these required fields:

- **Review comment** - the complete reviewer comment.
- **Document section** - the heading, subsection, diagram, table, or quoted passage containing it.

Also allow an explicit **Exit** action.

Validate that the section is identifiable in the primary document. When the user names a quoted
passage rather than a heading, locate that passage and use its enclosing section.

**Success criteria:** One pending entry has a stable ID, verbatim comment, and resolved source
section.

### Phase 3: Interpret and draft

Read the named section and enough surrounding context to understand the comment. Classify the
concern as one or more of:

- clarification;
- terminology;
- contradiction;
- missing decision;
- missing evidence;
- incomplete flow;
- failure mode;
- ownership or contract gap.

Separate the response into:

1. **Interpretation** - what concern the reviewer is raising.
2. **Proposed response** - concise text suitable for replying to the reviewer.
3. **Options** - only when more than one materially valid response exists.

Label statements as documented fact, inference, proposal, or unresolved gap when the distinction
matters. Use the primary document first and only then consult an allowed direct reference.

**Success criteria:** The pending entry has an evidence-grounded interpretation and proposed
response without invented claims.

### Phase 4: Response gate

Present exactly these actions:

| Action | Transition |
|---|---|
| `Accept` | Freeze the response and enter `AWAITING_COMMENT`. |
| `Revise` | Collect revision instructions, append a response version, and re-render the gate. |
| `Exit` | Enter `PENDING_EXIT_GATE`. |

Apply these rules:

- Free-form text after `Revise` is a revision instruction, not a new review comment.
- Do not accept another comment while the current response remains pending.
- Cancellation, empty input, or a wrong-button report mutates nothing.
- After cancellation or a wrong-button report, re-present the same pending response verbatim.
- Keep comment IDs stable across every revision.
- When accepted, derive follow-ups and assign stable follow-up IDs immediately.
- A cancelled, declined, empty, or wrong-button submission at any gate mutates nothing and
  re-renders that gate verbatim.

**Success criteria:** The response is accepted, remains pending for another revision, or proceeds to
the explicit exit gate.

### Phase 5: Consolidate follow-ups

Use only these follow-up types:

- `Document update`
- `Diagram update`
- `Contract decision`
- `Investigation`
- `Store design`
- `Governance investigation`
- `Test follow-up`

Deduplicate equivalent actions without renumbering existing IDs. Preserve the source comment IDs in
state. New follow-ups begin with `Status=Open`.

**Success criteria:** Each accepted response has zero or more stable, traceable follow-ups.

### Phase 6: Exit and optional save

Present exactly these actions:

| Action | Result |
|---|---|
| `Save comments and responses` | Render the fixed Markdown format, write it, and exit. |
| `Exit without saving` | Exit without creating an export. |
| `Continue reviewing comments` | Return to `AWAITING_COMMENT`. |

When `Exit` is selected from `RESPONSE_PENDING`, present this pending-entry gate instead:

| Action | Result |
|---|---|
| `Return to response` | Return to `RESPONSE_PENDING` without mutation. |
| `Discard pending and save accepted responses` | Mark the pending entry discarded, then export. |
| `Discard pending and exit without saving` | Mark the pending entry discarded, then exit. |

Default the export filename to:

```text
<source-document-stem>-review-comments-and-followups.md
```

Default the destination to the runtime-provided session artifact directory:
`<session-folder>/files/`. Resolve and record its absolute path before writing. When the runtime
does not expose that directory, ask the user for a destination instead of guessing. Never write
beside the primary document by default. For a custom repository destination, confirm the path and
apply the default-branch write gate before creating the file. Never overwrite an existing file
without explicit approval.

Render the document using [references/export-format.md](references/export-format.md). Re-read the
written file and confirm both required table headers and their column counts.

Recompute the SHA-256 hash of the primary document's file bytes before export. When it differs from
the loaded hash, disclose source drift and require confirmation before saving responses grounded in
the older document version.

**Success criteria:** The user explicitly exits with no export, or the export exists in the exact
defined format.

## State Machine

| State | Input | Next state | Mutation |
|---|---|---|---|
| `UNINITIALIZED` | Valid document | `AWAITING_COMMENT` | Store source and section index. |
| `AWAITING_COMMENT` | Comment + section | `RESPONSE_PENDING` | Create stable comment entry. |
| `AWAITING_COMMENT` | Exit | `EXIT_GATE` | None. |
| `RESPONSE_PENDING` | Accept | `AWAITING_COMMENT` | Freeze response and follow-ups. |
| `RESPONSE_PENDING` | Revise | `RESPONSE_PENDING` | Append response version. |
| `RESPONSE_PENDING` | Cancel/wrong button | `RESPONSE_PENDING` | None; re-render identical gate. |
| `RESPONSE_PENDING` | Exit | `PENDING_EXIT_GATE` | Retain and disclose pending entry. |
| `PENDING_EXIT_GATE` | Return to response | `RESPONSE_PENDING` | None. |
| `PENDING_EXIT_GATE` | Discard and save | `SUCCESS` | Discard pending entry; write export. |
| `PENDING_EXIT_GATE` | Discard without save | `SUCCESS` | Discard pending entry; no export. |
| `EXIT_GATE` | Continue | `AWAITING_COMMENT` | None. |
| `EXIT_GATE` | Save comments and responses | `SUCCESS` | Write and validate export. |
| `EXIT_GATE` | Exit without saving | `SUCCESS` | No export. |

`EXIT_GATE` is reachable only from `AWAITING_COMMENT`. A cancelled, declined, empty, or
unrecognized response at any state changes nothing and re-renders the same gate.

## Examples

### Start the loop

**Input:** `/document-comment-response-loop docs/design/example.md`

**Behavior:** Load `example.md`, then ask for the review comment and required section.

### Revise a response

**Input:** `Revise - make the response less verbose and separate the two questions.`

**Behavior:** Keep the same comment ID, revise the response, and show the response gate again.

### Recover from cancellation

**Input:** `Bring back the last response; I clicked the wrong button.`

**Behavior:** Change no state and re-present the pending response verbatim.

### Reject a second document

**Input:** `Also use docs/design/second.md as another source document.`

**Behavior:** Refuse to add it as a second primary document. Consult it only as one-level evidence
when the primary document directly references it and the current comment requires it.

## Completion

Use at most five bullets:

```markdown
**Result:** SUCCESS | PARTIAL | BLOCKED
- Primary document processed
- Accepted comment count
- Follow-up count
- Export path, when saved
- Pending gate or exact resume action, only for PARTIAL
```

Use `SUCCESS` after an explicit save or exit-without-save choice. Use `PARTIAL` only when the active
loop state is persisted but the workflow is interrupted before an exit choice. Use `BLOCKED` for an
unreadable document, unresolved multiple-document input, denied write destination, or another
deterministic safety failure.
