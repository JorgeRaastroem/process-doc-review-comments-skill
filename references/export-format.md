# Review Comment Export Format

Render saved review-comment artifacts with the exact headings and column order in this reference.

## Metadata and Tables

```markdown
# <Document Name> Review Comments and Follow-ups

**Source document:** `<canonical-source-path>`
**Review date:** `<YYYY-MM-DD>`
**Status:** Consolidated accepted responses; document updates and external investigations remain.

## Review Comments and Accepted Responses

| # | Section | Review comment | Accepted response |
|---|---|---|---|
| 1 | <section> | <verbatim reviewer comment> | <accepted response> |

## Consolidated Follow-ups

| ID | Type | Follow-up | Target section/artifact | Gate or owner | Status |
|---|---|---|---|---|---|
| F1 | Document update | <action> | <target> | <owner or gate> | Open |
```

## Rendering Rules

1. Preserve the verbatim reviewer comment in state.
2. Escape `|` as `\|` only when rendering a Markdown table cell.
3. Replace embedded CRLF or LF line breaks inside a cell with `<br>`.
4. Include accepted responses only in `Review Comments and Accepted Responses`.
5. Render comments in ascending stable comment-ID order and follow-ups in ascending stable
   follow-up-ID order.
6. The `#` column is the stable comment ID, not a row counter. It may contain gaps when entries were
   discarded. Never render discarded entries.
7. Keep comment and follow-up IDs stable; never renumber on export.
8. Use only these follow-up types:
   - `Document update`
   - `Diagram update`
   - `Contract decision`
   - `Investigation`
   - `Store design`
   - `Governance investigation`
   - `Test follow-up`
9. New follow-ups use `Open`. Preserve an explicitly updated status when state already contains one.
10. When there are no accepted comments or no follow-ups, render the corresponding header and
   separator rows with no data rows.
11. Never include credentials, tokens, secrets, or customer-identifying data in the export.
12. The same state must produce the same ordered table rows and cell text.

## Validation

After writing the export, re-read it and require:

- one `Review Comments and Accepted Responses` heading;
- one `Consolidated Follow-ups` heading;
- the comment table has exactly four columns;
- the follow-up table has exactly six columns;
- the number of comment rows equals the accepted-comment count;
- every follow-up type belongs to the closed vocabulary above;
- the output path differs from the primary document path.
