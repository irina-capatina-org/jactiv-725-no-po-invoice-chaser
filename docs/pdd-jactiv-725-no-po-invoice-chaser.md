# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-22 | 0.1 | uipath-analyst | Analyst | Initial analysis from source document request-work/request-details.md (converted from docs/jactiv-725-request-details.docx), version 1.0 dated 16 September 2026, authored by UiPath Cartographer |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-725 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-725 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser

| Field | Value |
|-------|-------|
| Process Full Name | NoPoInvoiceChaser |
| Business objective | Replace a daily manual Coupa review with a fully automated weekday run that identifies invoices from the past seven days with no properly linked purchase order and delivers a summary count to the AP SME via Slack, enforcing the no-PO-no-pay policy consistently and earlier than the current ad-hoc process |
| Owning department | Accounts Payable (Finance) |

**Delivery Team**

| Role | Name / Contact |
|------|---------------|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack member ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Attribute | Value |
|-----------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable control check — Finance |
| Short description | Automated weekday run that queries Coupa for invoices with status draft or new dated within the past seven days, excludes credit notes and invoices with a properly linked PO, counts the remainder, and sends one Slack direct message to the SME with the count and a filtered Coupa link; sends nothing on a clean day |
| Required roles | Unattended robot (Orchestrator-triggered); SME receives notification only |
| Trigger and schedule | Time trigger — weekdays at 10:00 Romania time (Europe/Bucharest) |
| Volume (items per run) | Sample run observed 194 qualifying invoices; typical daily volume [SME REVIEW] |
| Average handling time | Manual: ~daily task, duration unquantified in source [SME REVIEW]; Automated target: seconds per run |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low — data is structured, rules are deterministic; exact rate [SME REVIEW] |
| Input data | Coupa invoice list filtered by status (draft, new) and invoice date (past 7 days) |
| Output data | One Slack Block Kit direct message to SME with invoice count and filtered Coupa URL; no output on a clean day |

## 4. Scope

**In scope**

- Weekday execution triggered at 10:00 Romania time (Europe/Bucharest)
- Querying Coupa for invoices with status draft or new and invoice date within the past seven calendar days
- Excluding credit notes from the qualifying population
- Detecting invoices where a PO number appears only in the invoice description (not properly linked) and treating them as missing-PO
- Counting the invoices that qualify (no linked PO, correct status, correct date window, not a credit note)
- Composing one Slack Block Kit direct message to the SME containing the qualifying count, a policy reminder, an action request, and a filtered Coupa list URL
- Sending the Slack message to Irina Capatina (Slack member ID WLX9BD8FN) when the count is greater than zero
- Sending no Slack message on a day when the qualifying count is zero
- Reporting a run that cannot complete as a failed run (Orchestrator job failure)

**Out of scope**

- Purchase-order creation or modification
- Invoice approval or payment release
- Any modification of Coupa records
- Supplier communication
- Direct notification to the requester named on the invoice
- Requester follow-up tracking or confirmation of PO linkage closure
- Full invoice lifecycle automation
- Audit-retention design
- Listing individual invoices in the Slack message
- Retry, fallback or alternative recovery behaviour within the process (a failing run is a failed run; see BR-08 and open question OQ-01 regarding the future-state diagram)

## 5. To-Be Process (High Level)

The automation is a short, fully unattended linear sequence — read, filter, count, send — running once each weekday at 10:00 Romania time via an Orchestrator time trigger.

**Steps that are automated (replacing manual work):**

1. Opening the Coupa invoice list and applying filters (status, date window, credit-note exclusion) — eliminates manual filter setup and the risk of inconsistent timing.
2. Identifying invoices without a properly linked PO, including those where a PO number appears only in the description — eliminates manual per-invoice judgement.
3. Grouping and counting qualifying invoices — eliminates manual copy-paste and tallying.
4. Composing and sending the structured Slack Block Kit notification — eliminates manual message drafting and delivery.

**Steps that remain human:**

- Receiving the Slack notification and acting on it (SME raises or links purchase orders).
- Raising a purchase order for an invoice that lacks one (Requester / SME — out of scope for the automation).

**What disappears entirely:**

- The manual daily Coupa review session.
- Ad-hoc timing that allows missing-PO invoices to remain unseen until a supplier chases payment.
- Ambiguity between a clean result and a missed run.

No AI classification, human-in-the-loop approval step, retry logic, or fallback path is included in the normal process path. A run that fails is reported as a failed Orchestrator job; no secondary notification is sent within the process.

## 6. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.0 | Orchestrator time trigger fires at 10:00 Romania time (Europe/Bucharest) on a weekday | Orchestrator | Robot process starts | Schedule must be configured for the Europe/Bucharest timezone. Non-weekday triggers must not fire (BR-01 [SME REVIEW] — weekday-only scheduling is a scheduler configuration, not a process rule; confirm Orchestrator schedule covers Mon–Fri only) |
| 1.1 | Calculate the invoice date window: window_end = today's date; window_start = today minus 7 calendar days | Automation | window_start and window_end variables populated | Used in the Coupa query and in the Slack message footer. See BR-04 |
| 2.0 | Query Coupa invoice list via API (or web UI [SME REVIEW]) applying filter: status IN (draft, new) AND invoice_date >= window_start AND invoice_date <= window_end | Coupa | Paginated list of invoices matching status and date criteria returned | PO-linkage is on invoice lines, not the header — the query retrieves all candidate invoices for line-level inspection. See BR-04. Confirm whether Coupa API or UI scraping is the intended integration method — [SME REVIEW] |
| 2.1 | Validate that the Coupa response is well-formed (HTTP 200 / non-empty structure) | Coupa | Response validated | If response is invalid → S1 (system error path). Future-state diagram shows up to 3 retries for Coupa query; source text (BR-08/BR-09) states no retry required. See OQ-01 |
| 3.0 | **BEGIN per-invoice loop** — for each invoice returned | Automation | Loop initialised | Iterate over all records in the Coupa response |
| 3.1 | Check invoice type: is invoice type = credit note? | Automation | Boolean result | Rule BR-03: credit notes excluded |
| 3.1.A | If credit note → skip invoice, record exclusion reason = "credit note" | Automation | Invoice removed from qualifying set | Exclusion count may be logged to Orchestrator for observability [DEFAULT] |
| 3.2 | Check PO linkage: does the invoice have a properly linked purchase order on at least one invoice line? | Coupa | Boolean result — linked PO found or not | Rule BR-02: a PO number present only in the invoice description field does not satisfy this check. Linkage must be at line level |
| 3.2.A | If PO is properly linked → skip invoice | Automation | Invoice removed from qualifying set | Invoice meets the PO requirement; no action needed |
| 3.2.B | If no properly linked PO → retain invoice in qualifying set | Automation | Invoice remains in qualifying set | Applies to both: (a) no PO reference anywhere, (b) PO number in description only. See BR-01, BR-02 |
| 3.3 | **END per-invoice loop** | Automation | qualifying_count = number of retained invoices | All invoices processed |
| 4.0 | Decision: is qualifying_count > 0? | Automation | Branch: send message OR clean day | See BR-05, BR-07 |
| 4.1.A | If qualifying_count = 0 → clean day: end run without sending any Slack message | Automation | Run completes; no Slack message sent | BR-07. Note: section 7.1 of source states a congratulatory message should be sent on a clean day — this contradicts BR-07 and the scope table. See OQ-02 |
| 4.1.B | If qualifying_count > 0 → proceed to message composition | Automation | Proceed to step 5.0 | BR-05 |
| 5.0 | Compose the Coupa filtered list URL: https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D={{window_start}}&q%5Binvoice_date_lteq%5D={{window_end}}&q%5Bstatus_eq%5D=draft | Automation | coupa_url variable populated | URL filters by date window and status=draft. The URL cannot further filter to "no PO" because PO linkage is a line-level field not exposed in the Coupa list URL. See BR-06 |
| 5.1 | Compose Slack Block Kit JSON payload with: Title = ":receipt: {{invoice_count}} invoices need a purchase order"; Body line 1 = ":warning: {{invoice_count}} invoices from the last seven days have no purchase order linked."; Body line 2 = ":no_entry: An invoice without a linked PO cannot be matched or paid under our no-PO-no-pay policy, and payment to the supplier stalls until it is fixed."; Body line 3 = ":point_right: Please make sure a purchase order exists for these invoices and is correctly linked to each one."; Button = "Open the list in Coupa" (primary, links to coupa_url); Footer = ":calendar: Invoices dated {{window_start}} to {{window_end}} · :robot_face: No-PO Invoice Chaser · checked {{run_date}}" | Automation | Slack Block Kit JSON payload ready | invoice_count appears in title AND first body line (by design, readable in Slack sidebar without opening). Source states the exact Block Kit JSON is recorded verbatim in architectural considerations section 4 — content not provided in request document; see OQ-03 |
| 6.0 | Send Slack direct message to SME (Slack member ID WLX9BD8FN) via Slack HTTP Request activity with the composed Block Kit payload | Slack | Slack message delivered to Irina Capatina | BR-06. Message is a direct message by Slack member ID, never by email address. Confirm Slack API token and workspace [SME REVIEW] |
| 6.1 | Validate Slack delivery response (HTTP 200 / ok: true) | Slack | Delivery confirmed | If delivery fails → S2 (system error path) |
| 7.0 | Run completes successfully; Orchestrator job status = Successful | Orchestrator | Job logged as Successful | End of normal path |

## 7. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|-------------|---------------------|----------|
| Coupa | API or Web UI [SME REVIEW] | [SME REVIEW] — confirm whether Coupa REST API or UI scraping is used | [SME REVIEW] | Credentials stored in Orchestrator Asset or CyberArk [DEFAULT] | Sample URL hostname: uipath-test.coupahost.com (test environment confirmed from source). PO linkage is on invoice lines, not headers. Production hostname [SME REVIEW] |
| Slack | API (HTTP) | Slack HTTP Request activity (UiPath HTTP Request or Slack connector) | OAuth bot token | Bot token stored in Orchestrator Asset or CyberArk [DEFAULT] | Target recipient: Slack member ID WLX9BD8FN (Irina Capatina). Message format: Block Kit JSON. Protocol: Slack Web API HTTPS. Workspace [SME REVIEW] |
| UiPath Orchestrator | Orchestrator | Orchestrator scheduling | Robot machine trust | Managed by Orchestrator | Time trigger: weekdays 10:00 Europe/Bucharest. Delivery model [SME REVIEW] — Automation Cloud or Automation Suite |

## 8. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|----------------|
| BR-01 | Apply the no-PO-no-pay policy: invoices without a properly linked purchase order are flagged as non-compliant | BR-001 | 3.2, 3.2.B |
| BR-02 | A PO number typed into the invoice description but not properly linked at line level does not satisfy the PO requirement; such invoices are treated as missing-PO | BR-002 | 3.2, 3.2.B |
| BR-03 | Credit notes are excluded from the qualifying population | BR-003 | 3.1, 3.1.A |
| BR-04 | Include only invoices with status draft or new and an invoice date within the past seven calendar days | BR-004 | 1.1, 2.0 |
| BR-05 | Count the qualifying invoices; the Slack message reports that count; individual invoices are not listed in the message | BR-005 | 3.3, 4.1.B, 5.0 |
| BR-06 | Send the count to the SME (Irina Capatina, Slack member ID WLX9BD8FN) via Slack direct message, including a policy note, an action request, and a link to the filtered Coupa invoice list | BR-006 | 5.0, 5.1, 6.0 |
| BR-07 | Send no Slack message when a successful query returns zero qualifying invoices (clean day) | BR-007 | 4.1.A |
| BR-08 | No retry, fallback or recovery behaviour is required within the process; a run that cannot complete is simply reported as a failed run | BR-008 | 2.1, 6.1, 7.0 |
| BR-09 | A run that cannot complete produces no notification; no secondary message path exists | BR-009 | 2.1, 6.1 |
| BR-10 | The automation does not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion | BR-010 | All steps |

## 9. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|-------------------|--------|
| B1 | Credit note excluded | 3.1 | Invoice type is credit note | Skip invoice; record exclusion reason = "credit note"; continue loop. Per BR-03 |
| B2 | Description-only PO | 3.2 | PO number found in invoice description field only; no proper line-level PO linkage | Treat as missing-PO; retain in qualifying set. Per BR-02 |
| B3 | Clean day — no qualifying invoices | 4.0 | qualifying_count = 0 after full loop | End run without sending any Slack message. Per BR-07. Note: source section 7.1 contradicts this — see OQ-02 |

## 10. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|-------------------|----------|-------------|--------|
| S1 | Coupa query failure | Coupa returns non-200 HTTP status, malformed response, or connection timeout at step 2.0–2.1 | High | No retry per BR-08 (text); future-state diagram shows 3 retries — see OQ-01 [SME REVIEW] | Raise Orchestrator job exception; job status = Failed; no Slack message sent per BR-09 |
| S2 | Slack delivery failure | Slack HTTP call returns non-200 or ok: false at step 6.1 | High | No retry per BR-08 (text); future-state diagram shows 3 retries — see OQ-01 [SME REVIEW] | Raise Orchestrator job exception; job status = Failed; no secondary notification per BR-09 |
| S3 | Application unresponsive / element not found | Robot cannot interact with the target application at any step | High | None [DEFAULT] | Raise Orchestrator job exception; job status = Failed |
| S4 | Network timeout | General network unreachability to Coupa or Slack at any step | High | None [DEFAULT] | Raise Orchestrator job exception; job status = Failed |
| S5 | Credential expiry | Orchestrator Asset or CyberArk credential for Coupa or Slack is expired or revoked | High | None [DEFAULT] | Raise Orchestrator job exception; job status = Failed |
| S6 | Unhandled exception | Any unhandled runtime exception at any step | High | None [DEFAULT] | Raise Orchestrator job exception; job status = Failed; no notification per BR-09 |

## 11. Data Definitions

| Field | Type | Source | Target | Validation | Required |
|-------|------|--------|--------|------------|----------|
| invoice_date | Date | Coupa invoice header | Filter window comparison | Must be within window_start to window_end inclusive | Yes |
| invoice_status | String (enum) | Coupa invoice header | Filter logic | Must be "draft" or "new"; all other statuses excluded | Yes |
| invoice_type | String | Coupa invoice header | Credit-note exclusion logic | If = "credit note" → excluded | Yes |
| po_link | Boolean / object | Coupa invoice line(s) | PO-linkage check | True only if a PO is properly linked at line level; presence of PO number in description field = false | Yes |
| invoice_description | String | Coupa invoice header/line | PO-description check | Scanned to detect description-only PO numbers (not satisfying BR-02) | Yes |
| qualifying_count | Integer | Automation (computed) | Slack message, decision gate | Count of invoices passing all filters (status, date, not credit note, no linked PO); must be ≥ 0 | Yes |
| window_start | Date | Automation (computed: run_date minus 7 days) | Coupa query, Slack message footer, coupa_url | Must be a valid calendar date | Yes |
| window_end | Date | Automation (computed: run_date) | Coupa query, Slack message footer, coupa_url | Must be a valid calendar date ≥ window_start | Yes |
| run_date | Date | Automation (system date at trigger time) | Slack message footer | Must be a weekday in Europe/Bucharest timezone | Yes |
| invoice_count | Integer | Automation (= qualifying_count) | Slack message title and body line 1 | Must be > 0 when message is sent | Yes |
| coupa_url | String (URL) | Automation (computed) | Slack message Block Kit button | Parameterised with window_start, window_end, status=draft; must be a valid URL | Yes |
| slack_member_id | String | Hardcoded in process config | Slack API call recipient field | Must equal WLX9BD8FN | Yes |
| slack_bot_token | String | Orchestrator Asset / CyberArk | Slack HTTP Request auth header | Must be a valid active Slack bot OAuth token | Yes |
| coupa_api_credential | String | Orchestrator Asset / CyberArk [DEFAULT] | Coupa API / UI authentication | Must be valid and unexpired | Yes |
| exclusion_reason | String | Automation (computed) | Orchestrator log | One of: "credit note", "PO linked", informational only | No |

**Slack message template placeholders:**

| Placeholder | Resolved value |
|-------------|---------------|
| {{invoice_count}} | qualifying_count (integer) |
| {{coupa_url}} | Computed filtered Coupa list URL |
| {{window_start}} | run_date minus 7 days |
| {{window_end}} | run_date |
| {{run_date}} | Date of the current run |

**Coupa list URL pattern (from source):**

`https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft`

## 12. Environment and Constraint Signals

| Attribute | Signal |
|-----------|--------|
| Delivery model | [SME REVIEW] — source references Orchestrator scheduling and an Orchestrator-labelled trigger in the future-state diagram; whether this is Automation Cloud or Automation Suite (on-premises/cloud) is not stated |
| Product exclusions | No AI classification, document understanding, or human-in-the-loop activity. Source section 4.4 explicitly excludes AI agents and human approval steps |
| Orchestration constraints | Unattended robot; time trigger weekdays 10:00 Europe/Bucharest; single job per day; no queue design in source |
| Document storage | Not applicable — process produces no output documents |
| Signing modality | Not applicable — no document signing in scope |
| Robot attendance | Unattended — process runs on a schedule with no human interaction required in the normal path. Source section 4.1 confirms "fully automated" with no human review retained |
| Coupa environment (test) | uipath-test.coupahost.com (confirmed from source sample URL) |
| Coupa environment (production) | [SME REVIEW] |
| Slack workspace | [SME REVIEW] |

## 13. Canonical Test Data

| Field | Value | Role | Source location |
|-------|-------|------|----------------|
| qualifying_count (sample) | 194 | Input / validation — observed live sample count on one run | Section 4.3 of source: "with the count from a live sample: 194 invoices..." |
| Coupa test hostname | uipath-test.coupahost.com | Input — test environment URL base | Section 4.3 notification example URL |
| Coupa URL query param — invoice_date_gteq | q%5Binvoice_date_gteq%5D= | Input — URL-encoded filter parameter name | Section 4.3 example URL |
| Coupa URL query param — invoice_date_lteq | q%5Binvoice_date_lteq%5D= | Input — URL-encoded filter parameter name | Section 4.3 example URL |
| Coupa URL query param — status | q%5Bstatus_eq%5D=draft | Input — URL-encoded status filter | Section 4.3 example URL |
| SME Slack member ID | WLX9BD8FN | Input — hardcoded recipient for Slack DM | Sections 4.3, 5, 6, 8.1 of source |
| SME email | irina.capatina@uipath.com | Validation — confirms identity of Slack member ID WLX9BD8FN | Sections 5, 6, 8.1 of source |
| SME name | Irina Capatina | Validation | Sections 5, 6, 8.1 of source |
| Slack message title template | ":receipt: {{invoice_count}} invoices need a purchase order" | Expected output | Section 4.3 of source |
| Slack message body line 1 | ":warning: {{invoice_count}} invoices from the last seven days have no purchase order linked." | Expected output | Section 4.3 of source |
| Slack message body line 2 | ":no_entry: An invoice without a linked PO cannot be matched or paid under our no-PO-no-pay policy, and payment to the supplier stalls until it is fixed." | Expected output | Section 4.3 of source |
| Slack message body line 3 | ":point_right: Please make sure a purchase order exists for these invoices and is correctly linked to each one." | Expected output | Section 4.3 of source |
| Slack button label | "Open the list in Coupa" | Expected output | Section 4.3 of source |
| Slack button style | primary | Expected output | Section 4.3 of source |
| Slack footer template | ":calendar: Invoices dated {{window_start}} to {{window_end}} · :robot_face: No-PO Invoice Chaser · checked {{run_date}}" | Expected output | Section 4.3 of source |
| Invoice status values in scope | draft, new | Input — filter criteria | Sections 2, 3, 4.2, 5 of source |
| Date window length | 7 calendar days | Input — filter criteria | BR-004, sections 4.3, 5 of source |
| Trigger time | 10:00 | Input — schedule | Sections 1.2, 2, 4.1, 4.2 of source |
| Trigger timezone | Romania time (Europe/Bucharest) | Input — schedule | Sections 1.2, 4.1 of source |
| Clean day — expected Slack output | No message sent | Expected output | BR-007, scope table, section 1.2 of source |
| Invoice type excluded | credit note | Input — exclusion filter | BR-003, sections 5, 7.1 of source |
| Description-only PO — classification | Treated as missing PO (included in count) | Expected output | BR-002, section 7.1 of source |

## 14. Decomposition Signals

- **Distinct processing stages:** Three clear stages — (1) Coupa data retrieval and filtering, (2) count computation and message composition, (3) Slack notification delivery. Each could be a separate workflow sequence, but the source explicitly describes this as "a short linear process — read, filter, format, send" and says "anything beyond read, filter, format, send is out of scope."
- **Per-item transactional processing:** A per-invoice loop is present (steps 3.0–3.3) for credit-note exclusion and PO-linkage checking, but the output is a single aggregate count, not per-item transactions. Queue/Dispatcher–Performer pattern is not indicated; volume is described as low and the source explicitly rejects that complexity.
- **Document understanding with human validation:** Not present in source. Data is structured Coupa records; no OCR or document classification is involved.
- **Multiple output channels:** Single output channel only — one Slack direct message. No email, no file output, no secondary channel.
- **Reporting:** Not present in source beyond the single Slack message. No separate reporting workflow.
- **Queue/batch mentions:** Not present in source. Volume described as "approximately one manual review per day" and the data is "a handful of invoices a day." No queue design indicated.

## 15. Assumptions, Dependencies and Open Questions

1. **OQ-01 — Retry behaviour contradiction [SME REVIEW]:** The future-state diagram (image2.png, section 3.2 of source) shows up to 3 retries for the Coupa query and up to 3 retries for the Slack message send, followed by a Slack error-message path when retries are exhausted. This directly contradicts BR-08 ("No retry, fallback or recovery behaviour is required") and BR-09 ("a run that cannot complete produces no notification"). The text is treated as authoritative for this PDD; the retry and error-notification paths are excluded. SME must confirm which governs the build.
2. **OQ-02 — Clean-day notification contradiction [SME REVIEW]:** Section 7.1 (Exceptions table) states that when no qualifying invoices are found the automation should "Send a slack message saying congrats that all invoices have a po assigned." This contradicts BR-07, the scope table ("No message on a clean day"), and section 1.2 objective ("Send nothing on a day when no invoice qualifies"). BR-07 and the scope table are treated as authoritative. SME must confirm whether a clean-day Slack message is required and, if so, provide the exact wording.
3. **OQ-03 — Block Kit JSON payload not provided [SME REVIEW]:** Source section 4.3 states "The exact Block Kit JSON is recorded verbatim in the architectural considerations, section 4." That section is not present in the source document. The developer must obtain the canonical JSON from the SME before building. The element descriptions in section 4.3 are sufficient to reconstruct it but the verbatim payload must be confirmed.
4. **OQ-04 — Coupa integration method [SME REVIEW]:** The source states Coupa access type as "read" but does not specify whether the integration uses the Coupa REST API or UI scraping. PO-linkage data is on invoice lines and must be accessible via whichever method is chosen. Confirm the integration method and provide API credentials / endpoint documentation.
5. **OQ-05 — Coupa production hostname [SME REVIEW]:** The source provides only the test environment URL (uipath-test.coupahost.com). The production Coupa hostname must be confirmed before UAT and go-live.
6. **OQ-06 — Orchestrator delivery model [SME REVIEW]:** Source references Orchestrator scheduling but does not state whether the deployment target is Automation Cloud, Automation Suite, or standalone Orchestrator. This gates infrastructure provisioning.
7. **OQ-07 — Slack workspace and bot token [SME REVIEW]:** Source does not name the Slack workspace or provide the bot token or app credentials. The bot must have permission to send direct messages to member ID WLX9BD8FN. Confirm workspace, token storage location (Orchestrator Asset or CyberArk), and that the bot is already installed.
8. **OQ-08 — Date format in Coupa URL and Slack footer [SME REVIEW]:** Section 9, action 3 of the source notes "Agree the final reminder wording and a readable date and amount format." The date format for window_start and window_end in the Coupa URL query string and the Slack footer has not been confirmed. Confirm the required format (e.g., YYYY-MM-DD, DD/MM/YYYY).
9. **OQ-09 — Volume confirmation [SME REVIEW]:** Source states the task is "approximately one manual review per day" and the sample showed 194 qualifying invoices. Typical daily volume and peak volume are unconfirmed. Volume gates robot execution time and any Coupa API rate-limit considerations.
10. **OQ-10 — Referenced appendix not available:** Source document references Section 4 "architectural considerations" containing the verbatim Block Kit JSON and process maps. The process maps are available as image1.png and image2.png. The architectural considerations section was not included in the converted source document and is a missing input. See also OQ-03.
11. **[DEFAULT] Credential storage:** Orchestrator Asset or CyberArk is assumed for all credentials (Coupa, Slack bot token) per standard UiPath security practice. Confirm the actual credential store in use.
12. **[DEFAULT] Orchestrator job failure handling:** A run that cannot complete raises an Orchestrator job exception and the job status is set to Failed. No secondary notification is sent (BR-09). This is the standard UiPath unattended failure model and is applied here because the source explicitly requires it.
13. **[DEFAULT] Exclusion logging:** Credit-note and PO-linked exclusions are logged to the Orchestrator job log at Information level for observability. This does not affect the business output and can be removed if unwanted.

## 16. Success Criteria

1. A test run executes at 10:00 Romania time (Europe/Bucharest) on a weekday and completes without error.
2. The process queries Coupa and returns only invoices with status draft or new and an invoice date within the past seven calendar days; invoices outside this window or with any other status are absent from the result set.
3. Credit notes are excluded from the qualifying population and do not contribute to the invoice count.
4. An invoice where a PO number appears only in the invoice description field (not linked at line level) is counted as a missing-PO invoice.
5. The Slack message reports the correct total count of invoices with no properly linked PO; no individual invoice details are listed in the message.
6. One Slack direct message is delivered to Slack member ID WLX9BD8FN (Irina Capatina) when the qualifying count is greater than zero.
7. The Slack message is a Block Kit message containing: the qualifying count in both the title and the first body line; the policy sentence; the action request; a primary-styled button linking to the filtered Coupa invoice list URL; and a footer with the date window and run date.
8. The Coupa list URL embedded in the message opens the Coupa invoice list filtered to the same seven-day date window that the run used.
9. When the qualifying count is zero, no Slack message of any kind is sent and the Orchestrator job completes with status Successful.
10. The automation makes no modifications to Coupa records and does not create or modify any purchase orders.
11. A run that encounters a system error (Coupa unreachable, Slack delivery failure, unhandled exception) results in an Orchestrator job status of Failed with no Slack notification sent.
