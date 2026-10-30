---
name: checkout-issue-triage
description: "Investigate escalated checkout or booking issues from an issue and PNR. Use when tracing errors across Quantum Metric, Sabre, ePNR, Book, Mezmo Logs, or Azure Data Explorer and reporting the error, cause, location, evidence, and next steps."
---

# Checkout Issue Triage

Investigate a reported issue using its supplied details and any authorized,
connected system access. Return an evidence-backed report; do not infer a root
cause from an error code alone.

## Inputs

Start with the issue text or link and extract the PNR, relevant timestamps and
time zone, error codes or messages, channel, offers, and any transaction,
session, or correlation identifiers. Use only what is needed to investigate.
Do not repeat a PNR or unnecessary personal data in the final report.

If the issue content or a required identifier is missing, state what is missing
and proceed only as far as the available evidence allows. A link does not imply
access to its contents.

## Investigation procedure

1. Read the issue and note its claims separately from facts confirmed by
   investigation. Preserve codes and error messages verbatim.
2. Check the Quantum Metric session first. Correlate using the supplied session,
   PNR, time, or other identifiers. Record the relevant event and timestamp.
3. Before searching logs, check the corresponding Sabre, ePNR, and Book records
   when those systems and the needed access are available. Use them to validate
   the booking or transaction and gather useful timestamps or identifiers.
4. Search Mezmo Logs for the relevant time window and identifiers. Then search
   Azure Data Explorer logs, using the same evidence and any correlation details
   found in earlier checks.
5. Compare the sequence and timestamps across sources. Distinguish direct
   evidence from issue descriptions and hypotheses. Do not claim a source was
   checked if it was unavailable or not queried.
6. If a system is unavailable, inaccessible, or has no matching record, say so
   explicitly and identify what access or information would unblock the check.
   Never invent query results or silently treat missing evidence as confirmation.
7. Classify the component location as **Checkout**, **Ticketing and Receipts**,
   **Book**, or **Unknown**, based on where the error occurred—not where it was
   reported or noticed later. Record the repository's occurrence stage separately
   as **Checkout**, **T&R**, **Both**, or **Unknown**. Do not force Book into a
   stage value the existing triage record does not support.
8. Assign cause status as **Confirmed**, **Suspected**, or **Unknown**. Use
   Confirmed only when direct, correlated evidence establishes the cause; label
   hypotheses as Suspected and explain their supporting evidence. If unresolved,
   list the specific follow-up questions or checks needed.

Use only authorized, read-only access for investigation. Do not change bookings,
transactions, or system data.

## Report format

Return a structured report with these sections:

- **Issue summary** — concise description from the issue.
- **Error** — exact error message and code(s), preserving their source text.
- **Cause** — explanation and status: Confirmed, Suspected, or Unknown.
- **Location** — component (Checkout, Ticketing and Receipts, Book, or Unknown)
  and occurrence stage (Checkout, T&R, Both, or Unknown).
- **Evidence** — findings by source, with timestamps, relevant event/log
  references, and a clear distinction between observed facts and issue claims.
- **Confidence** — High, Medium, or Low. High requires directly correlated
  evidence; Medium means evidence is partial or indirect; Low means the
  conclusion is tentative or important sources are unavailable.
- **Next steps** — concrete checks, access, or answers needed to resolve
  uncertainty.

When preparing a repository triage entry, follow the Required Fields and
Text Output rules in [the repository workflow](../../../README.md). Preserve
codes verbatim, use `Not provided` for missing codes and `Explicitly none` only
when the source says none occurred, and associate each ERRCODE with its cause,
status, evidence, and stage. Return the proposed entry for review; do not append
it to `triage.txt` until the user has reviewed and approved it.
