---
name: checkout-issue-triage
description: "Investigate escalated checkout or booking issues from an issue and PNR. Use when tracing errors across Mezmo Logs, Sabre, ePNR, Book, or Azure Data Explorer and reporting the error, cause, location, evidence, and next steps."
---

# Checkout Issue Triage

Investigate a GitHub issue using its supplied details and any authorized,
server-side system integrations available in the execution environment. Return
an evidence-backed report; do not infer a root cause from an error code alone.
Do not depend on the user's browser session or ask them to sign in to a system
for the investigation.

## Inputs

Use any supplied issue text or link to extract the PNR, relevant timestamps and
time zone, error codes or messages, channel, offers, and transaction, session,
or correlation identifiers. A PNR by itself is enough to start searching Mezmo.
Use only what is needed to investigate. Do not repeat a PNR or unnecessary
personal data in the final report.

If the issue content or other identifiers are missing, start with the supplied
PNR rather than waiting for issue text or a Cart ID. A link does not imply access
to its contents. Ask necessary questions one at a time.

## Investigation procedure

1. Read the issue and note its claims separately from facts confirmed by
   investigation. Preserve codes and error messages verbatim.
2. Check the corresponding Sabre, ePNR, and Book records
   when those systems and the needed access are available. Use them to validate
   the booking or transaction and gather useful timestamps or identifiers.
3. Search Mezmo Logs using the connection and investigation steps below. Do not
   ask for an incident date merely because multiple logs match: scan the matching
   logs for errors first. Then search Azure Data Explorer logs when available,
   using the same evidence and correlation details.
4. Compare the sequence and timestamps across sources. Distinguish direct
   evidence from issue descriptions and hypotheses. Do not claim a source was
   checked if it was unavailable or not queried.
5. If a system is unavailable, inaccessible, has no configured server-side
   integration, or has no matching record, say so explicitly and identify what
   access or information would unblock the check. Never ask the user to provide
   credentials or session material. Never invent query results or silently
   treat missing evidence as confirmation.
6. Classify the component location as **Checkout**, **Ticketing and Receipts**,
   **Book**, or **Unknown**, based on where the error occurred—not where it was
   reported or noticed later. Record the repository's occurrence stage separately
   as **Checkout**, **T&R**, **Both**, or **Unknown**. Do not force Book into a
   stage value the existing triage record does not support.
7. Assign cause status as **Confirmed**, **Suspected**, or **Unknown**. Use
   Confirmed only when direct, correlated evidence establishes the cause; label
   hypotheses as Suspected and explain their supporting evidence. If unresolved,
   list the specific follow-up questions or checks needed.

Use only authorized, read-only access for investigation. Do not change bookings,
transactions, or system data.

## Mezmo connection and setup

The web application is `https://app.mezmo.com/`. The verified server-side route
is `GET https://api.mezmo.com/v2/export` with a personal access token (PAT) in the
`Authorization: Token <PAT>` header. Read the token from the environment variable
named **`Mezmo-PAT`**; never put its value in chat, source files, command literals,
URLs, diagnostic output, or reports. Treat the PAT as a secret even though
environment-variable storage is not a secret manager.

On Windows, check Process scope first, then User scope if missing or blank:

```powershell
$token = [Environment]::GetEnvironmentVariable('Mezmo-PAT', 'Process')
if ([string]::IsNullOrWhiteSpace($token)) {
    $token = [Environment]::GetEnvironmentVariable('Mezmo-PAT', 'User')
}
```

User scope is important: an already-running VS Code process may not inherit a
new variable. On other platforms, read the exact variable name from the process
environment using the platform's environment API.

If the variable is missing or blank, explain the blocker and provide these
instructions, without asking the user to paste a token:

1. In Mezmo, generate an organization-approved PAT with permission to read the
   relevant logs (`log: read`); restrict its scope and lifetime. An ingestion
   key is not a substitute. Follow company credential-storage policy.
2. On Windows, open **Edit environment variables for your account**, choose
   **User variables > New**, name it `Mezmo-PAT`, and enter the PAT locally as the
   value. Do not share screenshots containing it or enter it in a shell command
   that will be saved in history. If policy requires a secret manager instead,
   have the approved launcher populate the process variable securely.
3. Restart VS Code and its terminals to inherit the variable, or read the saved
   Windows User variable directly as above. On another OS, configure the exact
   process-variable name through an approved secret-aware launcher.
4. Verify presence without displaying the value, then validate read access as
   described below. If access is denied, ask for the approved access to be fixed,
   not for the credential itself.

### API requests and validation

- Send the PAT only to the documented HTTPS API host. Use an HTTP client that
  keeps it out of process arguments and output; disable redirects and use a
  bounded timeout. Do not print request headers or raw authentication errors.
- Supply `from` and `to` as Unix timestamps in milliseconds, `query` as a
  URL-encoded search expression, and `size` as the page size. Use exact quoted
  identifiers rather than sending unrelated customer data.
- First validate with a unique, non-customer query such as
  `connection-check-<random UUID>`, a one-minute window, and `size=1`.
  Require HTTP 200, valid JSON with a `lines` array, and no embedded API error.
  An empty `lines` array is valid for this deliberately unmatched query.
- Do not equate HTTP 200 with successful log retrieval. The v1 endpoint returned
  a success message that queued emailed results instead of streaming logs.
  Use **v2**, which was verified to return `lines` and `pagination_id` directly;
  do not use v1 or request emailed exports.
- Report HTTP failures, timeouts, malformed JSON, unexpected response shapes,
  and API error fields explicitly. Do not treat them as zero matching logs.
  For 401/403, identify authentication/permission failure without exposing the
  token or asking for it in chat.
- Use `pagination_id` for subsequent pages until exhausted. Never conclude there
  are no more errors from a partial page. Observe the documented 10,000-line
  maximum per page; if retrieval is incomplete, report that limitation.
- Keep secrets in memory only for the request and clear references afterward.
  Minimize customer payload output; if temporary log downloads are needed, keep
  them outside the repository and delete the specific files after analysis.

API references: [authentication](https://docs.mezmo.com/docs/api) and
[export API](https://docs.mezmo.com/api-reference/export/export-log-lines).

## Mezmo investigation

1. Use the supplied date/time window when available. Otherwise start with the
   last 30 days and state the actual UTC bounds. Search the exact PNR or supplied
   Cart ID. If nothing matches, widen within known retention/access limits and
   report the bounds searched; do not claim all history was searched.
2. Retrieve and scan **all matching pages**, not just the first log or latest
   occurrence. Multiple matches are expected; do not ask for a date just because
   there are several. Group by time, Cart ID, transaction ID, and application.
3. Inspect failure content even on `INFO` lines: `fullErrorCode`,
   `responseStatus`, `messages`, product responses, `nativeHostMessage`,
   exceptions, and failure redirects. `WARN`/`ERROR` levels help prioritize but
   are not sufficient filters. `_line` can contain the decisive response even
   when it is absent from structured fields; mixed-format text is not necessarily
   valid JSON. Preserve diagnostic text verbatim.
4. Extract Cart ID and transaction/correlation IDs from the error-bearing logs.
   Search those identifiers across applications in a focused time window;
   relevant downstream logs may not contain the PNR. Paginate these searches too.
5. Reconstruct the sequence, separating the original failure from subsequent
   retries, duplicate-cart rejection, cancellation, refund, and later success.
   Do not assume a cancellation/refund request proves it completed. Cite the
   evidence for each error separately and avoid diagnosing from its code alone.
6. If distinct error-bearing occurrences remain ambiguous after correlation,
   report each separately. Ask one targeted question only when needed to map
   them to the reported incident, not merely to reduce the number of log lines.
7. Record UTC timestamps, application names, exact codes/messages, Cart ID when
   useful, and log `_id` references. Distinguish a downstream host message
   observed through Mezmo from an independently checked Sabre/Book record.

## Report format

Return a concise, evidence-backed summary in this shape, omitting unnecessary
personal data and never repeating the PNR:

**Mezmo access:** verified, unavailable, or failed (state the actual outcome).

### Issue summary
Briefly describe the issue or observed sequence. Distinguish issue claims from
confirmed findings. Include **Cart ID** when available and useful.

### Error and cause
Use one row per distinct error, in chronological order:

| Time (UTC) | Error | Cause and status |
|---|---|---|
| <timestamp> | <exact code and decisive message> | <explanation; Confirmed, Suspected, or Unknown> |

Preserve codes and decisive messages verbatim; include long messages immediately
below the table if needed. Follow with one plain-language sentence distinguishing
the original failure from any later rejection or retry.

### Location
State the component (Checkout, Ticketing and Receipts, Book, or Unknown), any
downstream rejection, and occurrence stage (Checkout, T&R, Both, or Unknown).

### Evidence and confidence
Give source/application, timestamps, and log references supporting the findings.
State **High**, **Medium**, or **Low**, scoped to what the evidence establishes:
High requires directly correlated evidence; Medium is partial or indirect;
Low is tentative or missing important sources. Explicitly list relevant systems
that were not independently checked. Do not imply booking verification from
Mezmo access alone.

### Next steps
List concrete checks needed to resolve remaining uncertainty. If resolved,
identify the responsible follow-up rather than inventing additional causes.

When preparing a repository triage entry, follow the Required Fields and
Text Output rules in [the repository workflow](../../../README.md). Preserve
codes verbatim, use `Not provided` for missing codes and `Explicitly none` only
when the source says none occurred, and associate each ERRCODE with its cause,
status, evidence, and stage. Return the proposed entry for review; do not append
it to `triage.txt` until the user has reviewed and approved it.
