# Issue Intake and Automated Triage Plan

## Goal

Let internal reporters submit checkout or booking issues through a GitHub Issue
Form in the `AAInternal` organization. When an issue is created (or updated
with requested details), automatically run the repository's evidence-based
triage process and post the findings as a structured comment and set of labels
on that issue. Do not write triage findings to `triage.csv`.

## Proposed design

### 1. Enterprise Context (`AAInternal`)
- **Repository visibility and access:** Hosted inside the `AAInternal` GitHub
  Enterprise organization with private/internal visibility restricted to
  authorized American Airlines team members.
- **Runner topology:** Use `AAInternal` enterprise self-hosted or
  VNet-integrated GitHub Actions runners where required to reach internal
  services (Sabre, ePNR, Book, internal Azure Data Explorer clusters) alongside
  `https://api.mezmo.com/v2/export`.
- **Secret and identity management:** Store credentials in GitHub Actions
  secrets / enterprise secret stores (using `MEZMO_PAT` in Actions while
  supporting `Mezmo-PAT` for local VS Code runs) and prefer Azure workload
  identity federation (OIDC) for Azure Data Explorer and Azure OpenAI access.

### 2. Execution Architecture (Option A: Deterministic Collector + Enterprise LLM)
Split the automated workflow into two isolated stages so credentials are never
exposed to the LLM context:
1. **Stage 1 — Deterministic Data Collection Script:**
   - Parse the structured GitHub Issue Form body.
   - Validate inputs (e.g., 6-character alphanumeric PNR, date/time window,
     error codes, Quantum Metric session ID).
   - Query Mezmo (`GET https://api.mezmo.com/v2/export`) with full pagination,
     first by PNR/Cart ID and then in a second pass across applications using
     correlated Cart IDs and transaction/correlation IDs extracted from the
     initial hits.
   - Query available internal integrations (Sabre, ePNR, Book, Azure Data
     Explorer) in read-only mode, recording explicit status (`verified`,
     `unavailable`, `failed`) for each source.
   - Scrub secrets and redact raw PNR values from the collected bundle before
     passing it downstream.
2. **Stage 2 — LLM Triage Assessment & Comment Publisher:**
   - Run in a separate step/process with **no access to `MEZMO_PAT` or backend
     credentials**.
   - Send the sanitized evidence bundle and the rubric from
     `.github/skills/checkout-issue-triage/SKILL.md` to an AA-approved LLM
     endpoint (such as internal Azure OpenAI or GitHub Models).
   - Treat the reporter's free-text `Issue` description strictly as untrusted
     user claims to be verified against log evidence.
   - Post or update a single idempotent GitHub issue comment containing both the
     narrative report and the structured summary table (replacing `triage.csv`),
     and apply standardized GitHub labels for filtering.

### 3. Multi-turn / Asynchronous Clarification Flow
Because GitHub Actions cannot block interactively with `ask_user`:
- If critical correlation details are ambiguous (e.g., multiple unrelated failed
  checkouts across different dates with no clear match), the bot posts the
  partial findings, lists the specific follow-up question(s), and applies the
  `triage:needs-info` label.
- Re-runs are triggered automatically when the reporter edits the issue body
  (`issues: [opened, edited]`) or posts a `/retriage` comment
  (`issue_comment: [created]`), updating the existing bot comment in place.

### 4. PNR Handling and Redaction
- Because the issue lives in `AAInternal`, access is governed by repository
  permissions.
- Once the collector script correlates the PNR to a `Cart ID` (or completes its
  initial lookup), evaluate optionally masking the raw PNR in the issue title/body
  or keeping it strictly omitted from the bot's triage comment so downstream
  notifications and copied reports do not propagate the PNR.

## Intake form

Create a dedicated issue form at `.github/ISSUE_TEMPLATE/checkout-triage.yml`
that automatically applies the `triage:intake` label and captures these fields:

| Field | ID | Type | Required | Notes |
| --- | --- | --- | --- | --- |
| PNR | `pnr` | `input` | Yes | 6-character record locator. Do not echo in the automated comment. |
| Issue | `issue_description` | `textarea` | Yes | Free-text description of the reported problem. Treated as untrusted claims to verify. |
| Transaction date | `transaction_date` | `input` | No | ISO date or timestamp with time zone (defaults to searching last 30 days if omitted). |
| Error Code | `error_code` | `textarea` | No | Preserve `ERRCODE`, `INFCODE`, and `WRNCODE` values verbatim; do not infer cause from a code alone. |
| Quantum Metric session | `qm_session` | `input` | No | Session URL or ID used for correlation where available. |
| Channel | `channel` | `dropdown` | No | Options: `Booking`, `Change`, `Change Res`, `View Res`, `Unknown` (default). |
| Ancillary | `ancillary` | `checkboxes` / `dropdown` | No | Options aligned with repo: `Instant Upsell`, `Bags`, `Seats`, `Priority`, `Mileage Multiplier`, `Explicitly none`, `Unknown`. |

Confirm before creating the template whether `Ancillary` should use checkboxes
(allowing multiple selections plus `Explicitly none`) and whether `Error Code`
should remain a single text field accepting multiple codes or separate inputs
for `ERRCODE`, `INFCODE`, and `WRNCODE`.

## Automated Issue Output (Replacing `triage.csv`)

To preserve the structured tracking previously stored in `triage.csv` without
committing CSV changes on every issue:

1. **Structured Summary Table in the Bot Comment** (placed above the standard
   report sections):
   - **Transaction Date (UTC)**
   - **Primary Cart ID** (and any related retry Cart IDs)
   - **Purchased** (`Yes` / `No` / `Unresolved`)
   - **Adyen Response at Time of Error**
   - **Ancillaries Attempted**
   - **Component Location** (`Checkout`, `Ticketing and Receipts`, `Book`, `Unknown`)
   - **Occurrence Stage** (`Checkout`, `T&R`, `Both`, `Unknown`)
   - **Cause Status** (`Confirmed`, `Suspected`, `Unknown`)
   - **Confidence** (`High`, `Medium`, `Low`)
2. **Standard Narrative Sections** (from `SKILL.md`):
   - **System Access Status** (Mezmo, Sabre, ePNR, Book, ADX)
   - **Issue Summary**
   - **Error and Cause** (chronological table + plain-language summary)
   - **Location**
   - **Evidence and Confidence** (with UTC timestamps and log `_id`s)
   - **Next Steps / Follow-up Questions**
3. **Automated GitHub Labels** for repo-wide filtering and metrics:
   - Status: `triage:completed`, `triage:needs-info`, `triage:access-blocked`
   - Stage: `stage:checkout`, `stage:tnr`, `stage:both`, `stage:unknown`
   - Cause: `cause:confirmed`, `cause:suspected`, `cause:unknown`
   - Outcome: `purchased:yes`, `purchased:no`, `purchased:unknown`

## Implementation phases

1. **Finalize intake semantics and `AAInternal` prerequisites**
   - Confirm the `Ancillary` multi-select design and `Error Code` input format.
   - Confirm the target `AAInternal` runner group (for Sabre/ePNR/Book/ADX
     network reachability) and the approved enterprise LLM endpoint (Azure
     OpenAI or GitHub Models).
   - Define the standard label set to be provisioned in the repository.

2. **Add the GitHub Issue Form and parser**
   - Create `.github/ISSUE_TEMPLATE/checkout-triage.yml` with clear reporter
     guidance warning against pasting credentials, tokens, or payment card data.
   - Build a deterministic parser module (e.g., Python) that extracts the
     Markdown headings generated by the GitHub Issue Form into a typed JSON
     schema and handles missing optional fields, multiline error codes, and
     malformed inputs safely.

3. **Build the deterministic collector and align `SKILL.md` / `README.md`**
   - Implement the read-only data collector script:
     - Support both `MEZMO_PAT` (GitHub Actions secret) and `Mezmo-PAT` (local
       environment variable).
     - Perform the Mezmo connection validation check, paginated PNR/Cart ID
       search, and secondary correlation search across applications by
       extracted `Cart ID` and transaction IDs.
     - Add modular adapters for Sabre, ePNR, Book, and ADX that report explicit
       unavailability when not configured rather than failing silently.
   - Update `.github/skills/checkout-issue-triage/SKILL.md` and `README.md`:
     - Remove legacy `triage.txt` instructions.
     - Replace the automated `triage.csv` write requirement with the GitHub
       issue comment and label output (or scope `triage.csv` strictly to manual
       local runs if still desired).
     - Extract the system prompt / rubric so both local VS Code skill runs and
       the GitHub Actions LLM step share a single source of truth.

4. **Automate assessment and issue comments via GitHub Actions**
   - Create `.github/workflows/issue-triage.yml`:
     - Trigger on `issues: [opened, edited]` (when labeled `triage:intake`) and
       `issue_comment: [created]` (when a comment contains `/retriage`).
     - Enforce least-privilege `permissions` (`issues: write`, `contents: read`)
       and verify the triggering actor is an authorized `AAInternal` member.
     - Execute Stage 1 (Collector with `MEZMO_PAT`) and write a sanitized,
       PNR-scrubbed evidence artifact to runner temp storage.
     - Execute Stage 2 (LLM Evaluator without `MEZMO_PAT`) to generate the
       structured Markdown report and label recommendations.
     - Upsert a single bot comment identified by a hidden HTML marker
       (`<!-- checkout-triage-bot -->`) so edits and `/retriage` runs update
       the existing comment instead of spamming duplicates.
     - Apply the computed triage labels to the issue.

5. **Validate and roll out to `AAInternal`**
   - Add unit tests for the issue-body parser, Mezmo pagination/correlation
     logic, PNR redaction, and prompt-injection sanitization using fixture
     payloads.
   - Dry-run the workflow against test issues covering: confirmed checkout
     errors, downstream T&R errors, missing optional fields, ambiguous
     multi-attempt logs (`triage:needs-info`), and simulated API auth failures
     (`triage:access-blocked`).
   - Verify automated runs never modify `triage.csv` or leak secrets/PNRs in
     workflow logs or comments.
   - Migrate to `AAInternal`, configure enterprise secrets/runners, and pilot
     with the triage team.

## Acceptance criteria

- Reporters in `AAInternal` can submit issues via the GitHub Issue Form with
  **PNR** and **Issue** required and **Transaction date**, **Error Code**,
  **Quantum Metric session**, **Channel**, and **Ancillary** optional.
- Opening a matching issue (or commenting `/retriage`) runs the two-stage
  collector + enterprise LLM pipeline without relying on a user's browser
  session or exposing `MEZMO_PAT` to the LLM context.
- The assessment follows the repository's evidence rules, paginates all matching
  logs, correlates secondary logs by Cart ID/transaction ID, and explicitly
  reports unavailable systems or incomplete searches.
- The issue receives a single idempotent bot comment containing both the
  structured summary fields (formerly in `triage.csv`) and the full narrative
  report, plus filtering labels—without echoing the PNR or sensitive data.
- Ambiguous issues are labeled `triage:needs-info` with concrete follow-up
  questions and cleanly re-evaluated when the reporter updates the issue.
- The automation does not modify `triage.csv` or alter any booking or
  transaction records.
