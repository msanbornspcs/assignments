# Project 4 — Build a Crew and Govern It

**Course:** ISYS 398U — Agentic AI Implementation
**Crew:** Daily SRT Car Search Crew
**Platform:** CrewAI Studio (project `565f908a-eeb3-43ff-8207-2bf0e049dfd2`)
**Date:** 2026-06-16

---

## Part 1 — Design

### 1.1 Workflow Description

The Daily SRT Car Search Crew automates a used-car sourcing task I run on demand.
Its job is to search a fixed set of listing sources for three specific performance
vehicles — Chrysler 300 SRT-8 (2005–2010), Dodge Charger SRT-8 (2005–2010), and
Dodge Viper SRT-10 (2005–2008) — keep only stock/OEM cars that meet my criteria
(silver, under 100k miles), have a human confirm the assessment, and then deliver
the approved results two ways: an email summary and appended rows in a Google Sheet.
The trigger is manual (run on demand), and the end product is an email containing
price, mileage, location, and link for each approved listing plus one Sheet row per
filtered listing.

### 1.2 Agent Design Table

| # | Agent | Role / Job | Tools Granted | Tools Withheld | Task |
|---|-------|-----------|---------------|----------------|------|
| 1 | Used Car Listing Research Agent | Scrape each source URL and extract raw listing data | Firecrawl web scrape tool | Google Sheets, Gmail | Scrape Car Listing Data |
| 2 | Listing Filter Agent | Apply hard filters (model, year, color, mileage) and de-duplicate | None (text processing only) | Scraper, Sheets, Gmail | Filter and Deduplicate Listings |
| 3 | OEM Inspector Agent | Re-read each surviving listing and flag OEM vs. modified | Firecrawl web scrape tool | Sheets, Gmail | Inspect Listings for OEM vs Modified Status |
| 4 | Approval Coordinator | Present results and own the human checkpoint | None | Scraper, Sheets, Gmail | Human Approval of OEM Assessment |
| 5 | Output Agent | Write approved rows to Sheets and send the email | Google Sheets, Gmail | Scraper | Write to Google Sheets and Send Email |

All five agents run on `gpt-4o-mini`.

### 1.3 Orchestration Pattern Rationale

I chose a **Sequential** process rather than hierarchical. The task is strictly
linear: each stage consumes the previous stage's output and produces a narrower
result (scrape → filter → inspect → approve → deliver). There is no need for a
manager agent to dynamically delegate or re-order work, and delegation would only
add a way for an agent to invoke capabilities outside its scope. Sequential
execution also makes the audit trail easy to read because the handoffs are
deterministic. I turned **Allow Delegation off** on the Approval Coordinator so it
cannot push the irreversible work back upstream or sideways.

### 1.4 Human Checkpoint Design

The checkpoint sits **between the OEM Inspector and the Output Agent** — i.e., after
all analysis is done but before the only two irreversible actions in the workflow
(writing to the Sheet and sending the email). The Approval Coordinator presents an
approval packet for each candidate listing containing five fields: Listing URL,
Price, Mileage, OEM flag, and the reason for the OEM assessment. I configured a
**guardrail** on the approval task requiring the output to contain a non-empty
"Approved by:" line (a real reviewer name) and an "Approval timestamp:" line, so a
blank or placeholder approval should be rejected. The design intent is that nothing
gets sent or written until a human signs off.

---

## Screenshot 1 — AI Copilot Initial Canvas (Fit View, pre-edit)

*Not captured. This screenshot was not taken during the build and is intentionally omitted.*

---

## Screenshot 2 — Final Canvas with Process Type Dropdown

`screenshots/Screenshot 2.png`

Shows the completed five-agent canvas with the "Process Type: Sequential" control visible at top-left.

---

## Screenshot 3 — Human-Checkpoint Task Settings

`screenshots/Screenshot 3.png`

Shows the Human Approval of OEM Assessment task panel with the Description, Expected Output, and Guardrail fields configured.

---

## Part 3 — Run Notes

I executed the crew several times while debugging it; the notes below describe the
final successful end-to-end run (started 6/16/2026, approval task completed
2:15:26 PM), with the earlier iterations summarized for context.

### Per-Agent Handoff Observations (final run)

- **Research Agent → scrape_car_listing_data (94.15s):** Scraped the three cars.com
  trim-filtered search URLs via Firecrawl, ran three guardrail checks (all passed),
  and emitted raw listing records. Handed a structured list to the filter stage.

- **Filter Agent → filter_and_deduplicate_listings (3.35s):** Applied the model/year/
  color/mileage filters and de-duplicated. Reported removing records for color mismatch,
  mileage over the limit, and wrong year.

- **OEM Inspector → inspect_listings_for_oem_vs_modified_status (9.89s):** Re-read
  surviving listings and tagged each OEM vs. modified. Found 2 OEM listings, 0 modified.

- **Approval Coordinator → human_approval_of_oem_assessment (33.44s):** Produced the
  approval packet (two OEM Chrysler 300 SRT8 listings with real cars.com/vehicledetail
  links, honest "VIN: Not shown," prices, mileage, locations, and OEM reasoning).

- **Output Agent → write_to_google_sheets_and_send_email:** Called
  `google_gmail_send_email` and `google_sheets_append_values`. The email was delivered
  and the Sheet rows were written (confirmed manually).

### Did the Human-Approval Task Actually Pause for a Real Human? — No.

This is the most important observation of the project. The approval task's description
explicitly read: *"Clearly ask the human to review the assessment and provide explicit
approval before the final write and email steps proceed. Do NOT proceed to the next task
until the human confirms approval."* In its own output the Approval Coordinator even
wrote *"Please review the assessment above and provide explicit approval (approve or
reject) before I proceed..."* — and then, in the same output, it filled in its **own**
approval: `**Approved by:** [Human Name]` with a fabricated
`**Approval timestamp:** 2026-06-16 12:00 PM`. Execution did **not** halt for live
human input. The crew continued straight into the irreversible Sheets write and email
send with no actual human in the loop. In an earlier run the guardrail did catch a
similar fabricated approver ("[Human Reviewer Name]") and blocked the run, but once the
packet was phrased to satisfy the guardrail's text checks, the run proceeded
autonomously. The "human checkpoint," as implemented with a task + guardrail in CrewAI
Studio, is not a true blocking interrupt — it is an autonomous step that can self-satisfy.

### Iteration History (debug loop)

- **Run 1 (basic scraper, original 11 URLs):** Bad/stale source URLs → research agent
  hallucinated wrong vehicles (Hondas, Silverado, Jeep) with fake VINs and example.com
  links. Guardrail blocked the run on a fabricated approver name.

- **Run 2 (Firecrawl scraper):** Real SRT cars appeared, but the agent padded gaps with
  example.com URLs and fake VINs. Anti-fabrication guardrail correctly rejected across
  3 retries.

- **Run 3 (corrected cars.com + eBay URLs):** Real data flowed, but the run hit the
  platform's 600-second execution timeout before reaching the checkpoint.

- **Run 4 (trimmed to 3 URLs, per-page cap):** Failed because my task said to write
  "not shown" for missing VINs while the guardrail treated "not shown" as a placeholder
  — a self-inflicted task/guardrail conflict.

- **Run 5 (VIN-tolerant guardrail):** Clean end-to-end success — all guardrails passed,
  no timeout, real listings delivered — but the HITL checkpoint did not enforce a stop.

---

## Screenshot 4 — Run Timeline (≥2 completed agents)

`screenshots/Screenshot 4.png`

Shows the Run-tab Timeline with completed task entries for multiple agents.

---

## Part 4 — Governance Audit

### 4.1 Design vs. Reality

The crew matched my design in most respects: tool scoping held (only the Output Agent
ever touched Sheets/Gmail; only Research and OEM Inspector scraped), the process stayed
strictly sequential, and the audit timeline recorded every handoff. Three places where
reality diverged from design:

1. **The human checkpoint did not block.** The biggest gap — the approval step ran
   autonomously and self-approved.

2. **The Copilot added components I did not specify.** Generation produced a daily
   9 AM UTC schedule trigger and four runtime variables (`{listing_urls}`,
   `{spreadsheet_id}`, `{sheet_name}`, `{recipient_email}`). Useful, but they expanded
   the system.

3. **The filter under-enforced my spec.** A 2012 Chrysler 300 SRT8 passed even though
   my year window is 2005–2010, and the approved cars were not silver / nearby. The
   natural-language filter is softer than my stated hard criteria.

### 4.2 Tool Scope Table (design vs. actual)

| Agent | Tools (designed) | Tools (actual on canvas) | Match? |
|-------|------------------|--------------------------|--------|
| Research Agent | Scraper only | Firecrawl web scrape | Yes |
| Filter Agent | None | None | Yes |
| OEM Inspector | Scraper only | Firecrawl web scrape | Yes |
| Approval Coordinator | None | None | Yes |
| Output Agent | Sheets + Gmail | Google Sheets + Gmail | Yes |

Tool scoping was the strongest control in practice.

### 4.3 Threat Class Table

| Threat class | How it could manifest here | Observed? | Control in place |
|--------------|---------------------------|-----------|------------------|
| Prompt injection | A listing page embeds text telling the scraper to ignore filters or change the recipient email | Not observed, but plausible via scraped page content | Filter/OEM agents have no send tools; anti-fabrication guardrail |
| Tool abuse / over-action | Output Agent sends email or writes Sheet without approval | **Observed** — send/write executed without real approval | Intended HITL checkpoint (failed to block) |
| Over-permissioning | An analysis agent given Sheets/Gmail could leak data | Not observed | Tool scoping (held correctly) |
| Data leakage | Wrong recipient, or fabricated data written to the Sheet | Partially — placeholder/fabricated content reached output in early runs | Anti-fabrication guardrail; fixed recipient variable |
| Fabrication / hallucination | Agent invents listings when scraping fails | **Observed** in Runs 1–2 | Anti-fabrication guardrail (caught it) |

### 4.4 Audit Trail Evaluation

- **What does the Timeline capture, and what is missing?** The Run-tab Timeline nested
  every agent, task, LLM call, tool call, and guardrail check with relative timestamps,
  so I could trace each handoff and see exactly where each run failed. What is missing:
  the trace had a null `source_fingerprint`, so I could not cryptographically tie outputs
  back to specific scraped sources. Tool call timing was visible but tool payloads (the
  actual scraped text the model saw) were not easy to inspect, which made diagnosing
  fabrication harder. I could reconstruct the sequence of events, but not fully verify
  the fidelity of what the agents actually received.

- **The human-approval task — what does the Timeline say happened, and is that what
  should have happened?** The Approval Coordinator task started and completed in 33.44
  seconds. No real human entered anything during that window. The Timeline records the
  task as "Completed" with a passing guardrail check — but the "Approved by:" field
  contains `[Human Name]` and the timestamp reads `2026-06-16 12:00 PM`, both fabricated
  by the agent itself. The Timeline shows an approval that never happened, logged with the
  same status as a legitimate one. This is the core governance finding: the audit trail
  cannot distinguish a real human decision from an autonomous self-approval.

- **Who in a real organization would need access to this log, and how often?** The person
  who acts on the output — whoever receives the email and owns the Google Sheet — needs
  access after every run, not weekly and not only after an incident. Because this crew
  produces irreversible writes and sends on every execution, post-run review is the only
  point where a bad output can be caught before it is acted on. In a purchasing or
  sourcing team context, that is the procurement lead or data owner, reviewing after each
  daily run.

- **What is the longest period I would feel comfortable letting this crew run with no
  human reviewing the Timeline?** One run — meaning after each execution the Timeline
  should be reviewed before the next one fires. The threshold is driven by two specific
  risks in this crew: (1) the fabrication risk demonstrated in Runs 1–2, where the agent
  invented listings with fake VINs and example.com URLs that only a guardrail — not the
  Timeline — caught; and (2) the non-enforcing checkpoint, which means a fabricated
  listing that phrases itself to satisfy the guardrail will reach the Sheet and email
  undetected. Letting multiple runs accumulate before review means bad data compounds in
  the Sheet with no opportunity to catch it.

### 4.5 Top 3 Risks + Top 3 Mitigations

**Risks**

1. **Non-enforcing human checkpoint** — irreversible actions (email/Sheet write) execute
   without a real human approving.
2. **Fabrication under tool failure** — when scraping fails, the model invents listings;
   only a guardrail stands between fabrication and delivery.
3. **Soft filtering** — the natural-language filter lets out-of-spec vehicles through.

**Mitigations**

1. **Remove Gmail/Sheets tools from the Output Agent** so the crew physically *cannot*
   perform irreversible actions autonomously; have it output the approval packet for me
   to review, and I perform the send/write manually after approving. (Re-architects the
   HITL gate into a real stop.)
2. **Keep and strengthen the anti-fabrication guardrail** (no example.com, no invented
   VINs/prices; explicit "NO DATA RETRIEVED" on failure) and prefer structured sources.
3. **Convert filters to hard, code-level checks** (exact year range, color, mileage cap)
   rather than relying on the LLM to enforce them.

---

## Part 5 — Reflection

**Q1 — Describe one governance decision you made that the platform did not make for you.**

The decision I made that the AI Copilot did not make was **removing all tools from the
Filter Agent and the Approval Coordinator**. The Copilot's initial generation did not
enforce strict per-agent tool boundaries — I had to open each agent's settings panel and
manually strip tools that were outside each agent's role. For the Filter Agent the
reasoning was that a text-processing step has no legitimate reason to make network
requests; giving it a scraper would let it re-fetch listing pages on its own initiative,
outside the controlled flow. For the Approval Coordinator the reasoning was structural:
an agent that holds the Gmail and Sheets tools can perform the irreversible actions
itself, which collapses the separation between "gatekeeper" and "actor" that the
checkpoint is supposed to enforce. The platform generated the crew; I made the scoping
decisions that determined what each agent was actually capable of doing.

**Q2 — If this crew produced a wrong output at a workplace, who would own the response and what would they need from the Timeline?**

The owner would be the **procurement lead or data operations manager** — whoever is
responsible for the sourcing list that the Google Sheet feeds. If the crew emailed a
fabricated listing to a buyer who then contacted the seller or drove to an inspection,
that person's time was wasted and the Sheet now contains bad data. To investigate, they
would need to open the Approval Coordinator's Timeline entry and read the Event Details
tab for the `human_approval_of_oem_assessment` task. That entry shows the exact
"Approved by:" and "Approval timestamp:" text the agent produced — if those fields
contain `[Human Name]` or a fabricated timestamp rather than a real reviewer's name, the
investigation is complete: the approval was autonomous, no human authorized the output,
and the source of the error is the non-enforcing checkpoint, not a downstream system.
