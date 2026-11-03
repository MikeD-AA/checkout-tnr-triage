# checkout-tnr-triage

Manual triage of escalations selected from email or Teams. The output is a
plain-text record in [triage.txt](triage.txt), with a post-analysis spreadsheet
update when authorized workbook access is available.

## Workflow

1. Select the email or Teams message to triage. Include relevant replies and
	diagnostic excerpts, but omit credentials, tokens, and unnecessary personal data.
2. Provide the message text for triage, along with a source link or reference
	when available. Selecting a message alone does not give this workspace access
	to Outlook or Teams.
3. Extract the fields below using only the supplied evidence. Preserve code values
	exactly as written, including prefixes, leading zeros, and capitalization.
4. Review the proposed record. Confirm the failure stage and cause where possible;
	leave unresolved details explicitly marked and list follow-up questions.
5. Append the reviewed entry to `triage.txt`, using its template for each escalation.
	Keep related updates under the same entry ID. Do not overwrite earlier entries.

For investigations using the
[checkout issue triage skill](.github/skills/checkout-issue-triage/SKILL.md),
record completed analysis in the **Master** worksheet of **ERRCODE858
Triage_Updated 100626**, following the skill's post-analysis spreadsheet mapping.
Map existing column titles rather than relying on column letters, preserve
existing records, and verify persisted values. Keep **Master** in ascending
transaction-date order (oldest to newest): insert earlier transactions in their
chronological position rather than appending them, keeping complete rows aligned.
When an available workbook or
browser tool requires sign-in, prompt the user to sign in directly and confirm
readiness, preferably after analysis. Never collect credentials or session
tokens. Verify tool access after sign-in; if unavailable, return proposed values
and report the blocker rather than claiming an update.

## Required Fields

| Field | Capture Rule |
| --- | --- |
| Channel | Booking, Change, Change Res, or View Res; use Unknown when not specified in the source. |
| Offers | Capture all applicable offers: Instant Upsell, Bags, Seats, Priority, Mileage Multiplier. Use Unknown when not specified, and Explicitly none only when the source states no offers were involved. |
| ERRCODE | All reported error codes, verbatim. |
| INFCODE | All reported information codes, verbatim. |
| WRNCODE | All reported warning codes, verbatim. |
| Cause of ERRCODE | A cause for each ERRCODE, with supporting evidence and confirmation status. |
| Occurrence stage | Checkout, T&R, Both, or Unknown; map each ERRCODE to its stage if they differ. |

Use `Not provided` for missing codes and `Explicitly none` only when the source
states that no such code occurred. Do not infer the code category from an
unlabeled value; record it in Evidence and ask for clarification.

For causes, use `Confirmed`, `Suspected`, or `Unknown`. A suspected cause must be
identified as such in the source; do not invent a diagnosis from a code alone.
When no cause is supplied, use `Unknown - not provided`.

Determine the stage from where the error occurred, not where it was reported or
noticed later. Use `Both` only when the evidence establishes occurrence in both
Checkout and T&R. Otherwise use `Unknown` and request clarification.

## Text Output

The initial `triage.txt` contains an unfilled template, not an actual escalation.
Replace that template with the first reviewed record and reuse its structure for
subsequent entries. Use a unique entry ID such as `TRIAGE-0001` and an ISO 8601
triage timestamp with a time zone. These provide stable references for a future
tracker without requiring one now.

Retain a short source excerpt supporting the codes, stage, and cause. Record
missing evidence as follow-up questions, not as established facts. If one
escalation contains multiple ERRCODEs, repeat the ERRCODE details block for each
so their causes and stages remain associated.

## Request To Use For Each Escalation

```text
Triage the following email or Teams escalation using this repository's workflow.
Extract Channel (Booking, Change, Change Res, or View Res), all applicable Offers
(Instant Upsell, Bags, Seats, Priority, Mileage Multiplier), ERRCODE, INFCODE,
WRNCODE, cause of each ERRCODE, and whether each error occurred during Checkout
or T&R. Preserve codes verbatim, cite supplied evidence, and mark missing or
unconfirmed details explicitly. Return a proposed plain-text entry using the
triage.txt structure for review before it is appended.

Source type: <Email or Teams>
Source reference: <Link or identifying reference, if available>
Message and relevant diagnostic excerpts:
<Paste here, with secrets removed>
```