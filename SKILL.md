---
name: deal-intake
description: "Xero Enterprise Deal Intake — guides a seller through submitting a deal for approval. ALWAYS trigger this skill when the user says 'I want to start a strategic AB deal review' or any close variation. Also trigger when a seller wants to submit a deal, run deal intake, generate a deal summary, or get approval for an enterprise deal. The seller pastes a Deal Summary from the Xero Deal Model Google Sheet and Claude runs KPI evaluation, asks structured questions, and produces a formatted deal summary uploaded to Google Drive with an Asana approval task."
---

# Deal Intake Skill

Trigger this skill when a user uploads or references a Deal Model file and asks to run deal intake, submit a deal, or generate a deal summary.

## What this skill does

Runs the full Xero Enterprise Deal Intake workflow entirely inside Claude:
1. Reads the uploaded Deal Model (Google Sheet or XLSX)
2. Evaluates KPIs against approval thresholds
3. Pre-fills seller fields from Salesforce flat file
4. Asks the seller the remaining questions in chat
5. Generates the Deal Summary HTML
6. Uploads to Google Drive
7. Creates an Asana approval task

## Step-by-step instructions

### Step 1 — Guide the seller to generate a Deal Summary

If the seller hasn't already pasted in a Deal Summary, walk them through these steps:

> **Step 1 of 4 — Download the Deal Model template**
> Download your Deal Model here:
> https://docs.google.com/spreadsheets/d/1MoIlq_cYL56YV-zOVViU-DwpwMJ2d_VV2Yf74HKnGio/
> Fill in your deal details, then come back here.

Once they confirm they've filled it in (or if they already have):

> **Step 2 of 4 — Generate the Deal Summary**
> In the Google Sheet, click **Claude Tools** in the menu bar, then select **Create Deal Summary**. This will generate a block of text.

Once they confirm that's done:

> **Step 3 of 4 — Paste the Deal Summary**
> Please paste the Deal Summary text here and I'll run the analysis.

Wait for the seller to paste the Deal Summary text before proceeding.

The partner name, location, and deal model URL are captured automatically from the Google Sheet by the Apps Script. Use them directly from the pasted summary — do not ask the seller for the partner name. If the partner name is blank or "—", note it inline as "Partner name not set in Deal Model" and continue.

### Step 2 — Run the analysis

Save the pasted Deal Summary text to a temp file, then run:

```bash
# (Claude writes the pasted text to /tmp/deal_summary.md first)
cd /sessions/stoic-admiring-allen/mnt/No\ UI\ Deal\ Intake/deal_intake
python3 workflow.py analyze_from_text /tmp/deal_summary.md > /tmp/analysis.json 2>/dev/null
```

Parse the JSON output. Extract:
- `deal.partner_name`, `deal.gross_tcv`, `deal.term_months`, `deal.commencement_date`
- `partner_location` — include in the deal snapshot table next to partner name
- `deal_model_url` — include as a hyperlink "Open Deal Model" in the deal snapshot table and in the Asana task notes
- `approval.tier`, `approval.recommendation`, `approval.metrics`
- `enrichment` — fields already pre-filled from Salesforce flat file
- `questions` — full question list with `pre_filled` and `needs_input` flags

### Step 3 — Present KPI evaluation to the seller

Show the approval decision clearly in chat:

```
**KPI Evaluation — [Partner Name]**

| Metric | Value | Sign-off required |
|--------|-------|-------------------|
[one row per metric with RAG indicator]

**Approval required: [TIER]**
[recommendation narrative]
```

### Step 4 — Confirm enriched fields

If any fields in `enrichment` are non-empty, show them in a table and ask the seller to confirm or correct:

> The following fields have been pre-filled from Salesforce. Please confirm or correct anything that's wrong.

Wait for the seller's response before proceeding. If no enrichment data exists, skip this step.

### Step 5 — Ask unanswered questions (ALL sections)

**Important: ask one question at a time whenever choices are involved.** For free-text questions with no choices, you may group questions within a section. Always wait for the seller's reply before moving to the next question or section.

Sections and order:

**1. Deal context**
- Ask seller name and key risks together (both free text, no choices)
- ~~Do NOT ask about deviations from standard terms~~

**2. Strategic value to Xero** *(future: pre-fill from Gong)*
Ask each question separately:
- Market impact (free text)
- MRR at risk (free text)
- Employees (free text)
- Competitive landscape (free text)
- Relationship status — present as a choice:
  > What is the current relationship status?
  > 1. New Prospect
  > 2. Strategic Partner — QBR cadence
  > 3. Existing Customer — renewal
  > 4. At risk
  > 5. Other — describe
- Current tech stack (free text)
- Manual processes to automate (free text)

**3. Strategic importance to customer**
Before asking, search the web for "[Partner Name] accounting software strategy" or similar to find publicly available information about the partner's strategic priorities, cloud adoption goals, or technology direction. Present any findings to the seller as a suggested answer for approval:
> Based on what I found online, here's a suggested outcome statement for [Partner Name]:
> "[web-sourced summary]"
> Reply "yes" to use this, or type your own.

If no useful web results are found, ask as free text:
- Outcomes for the partner

**4. Deal status** *(future: pull from Salesforce/Gong)*
Ask each separately:
- Deal stage — present as a choice:
  > What stage is the deal at?
  > 1. Identify & Research
  > 2. Discovery & Qualification
  > 3. Value Prop Tailoring
  > 4. Strategic Engagement
  > 5. Negotiation & Closing
  > 6. Implementation & Success
- Key decision makers (free text)
- Expected close date (free text)
- Seller notes (free text)

**5. Billing currency** — present as a choice:
> Billing currency?
> 1. £GBP
> 2. $USD
> 3. $AUD
> 4. $NZD
> 5. $CAD

~~Do NOT ask any other commercial terms — they are no longer in the deal summary.~~

**6. Incentives & engagement model** — ask each row ONE AT A TIME. For each:
> **[Dimension]**
> 1. Min — [minimum acceptable value]
> 2. Ideal — [end state ideal value]
> 3. Other — describe

Wait for the seller's answer before presenting the next row.

Store the outcome text AND a flag (`MIN`, `IDEAL`, or `CUSTOM`) for each row.

**Incentive matrix:**

| Dimension | Min (option 1) | Ideal (option 2) |
|-----------|---------------|-----------------|
| Initiation Rate | 25% within 6 months | 50–100% within 12 months (scaled to partner size) |
| Discount Structure | Short-term 80% / 6 months | Long-term tiered discounts linked to MRR milestones + plan mix |
| Platform Adoption | Bookkeeping | Full Platform Adoption (Books to Tax, Payments and Syft) |
| Contract Term | 12 months (new) | 24 months (strategic) |
| Clawback Protection | Full discount refund on miss | Full refund + milestone gates to prevent premature benefit unlock |
| Migration Support | GMT & MMB | GMT + MMB + Advance Track (enhanced) + train-the-trainer |
| Co-Investment | $5,000 annual marketing fund | Funded marketing ($X × number of orgs) + event sponsorship + bespoke initiative support |
| DIT Marketing | Provision of Xero Sell Through Assets | Allow SB engagement for Sell through and AB visibility |
| Consulting | DTC & remote delivery | Dedicated DTC + regional delivery + dedicated KAM & account team |
| Roadmap Influence | None | Thought leadership roundtable seat (strategic cohort only) |
| ROFR / Governance | Standard provisions | Documented ROFR + auto-renewal + 90-day harmonisation on M&A |

For each section, ask all questions together in a numbered or bulleted list so the seller can reply to all at once.

### Step 6 — Collect all answers into answers.json

Once all questions are answered, build the full answers dict covering every field and write to `/tmp/answers.json`. Include all fields from every section — do not omit any. Fields the seller left blank should be empty strings.

```python
import json
answers = {
    "seller_name": "...",
    "key_risks": "...",
    "deviations_from_standard_terms": "...",
    "market_impact": "...",
    "retention_risk": "...",
    "employees": "...",
    "competitive_landscape": "...",
    "relationship_status": "...",
    "current_tech_stack": "...",
    "manual_processes": "...",
    "outcomes_for_partner": "...",
    "billing_currency": "...",
    "deal_stage": "...",
    "key_decision_makers": "...",
    "expected_close_date": "...",
    "seller_notes": "...",
    "initiation_rate": "...",            "initiation_rate_flag": "MIN|IDEAL|CUSTOM",
    "discount_structure_outcome": "...", "discount_structure_flag": "MIN|IDEAL|CUSTOM",
    "platform_adoption": "...",          "platform_adoption_flag": "MIN|IDEAL|CUSTOM",
    "contract_term_outcome": "...",      "contract_term_flag": "MIN|IDEAL|CUSTOM",
    "clawback_protection": "...",        "clawback_flag": "MIN|IDEAL|CUSTOM",
    "migration_support": "...",          "migration_flag": "MIN|IDEAL|CUSTOM",
    "co_investment_outcome": "...",      "co_investment_flag": "MIN|IDEAL|CUSTOM",
    "dit_marketing": "...",              "dit_marketing_flag": "MIN|IDEAL|CUSTOM",
    "consulting_outcome": "...",         "consulting_flag": "MIN|IDEAL|CUSTOM",
    "roadmap_influence": "...",          "roadmap_flag": "MIN|IDEAL|CUSTOM",
    "rofr_governance": "...",            "rofr_flag": "MIN|IDEAL|CUSTOM"
}
with open('/tmp/answers.json', 'w') as f:
    json.dump(answers, f)
```

### Step 7 — Seller review

Before generating the document, present a full review summary in chat and ask the seller to confirm or correct anything.

Format it clearly as follows:

---
**📋 Deal Review — [Partner Name]**

**KPI Evaluation**
| Metric | Value | Status |
|--------|-------|--------|
| TCV | £Xk | 🟡 Sales Director |
| Cont. Margin % | X% | 🟡 Sales Director |
| CAC Payback | X months | 🟢 Benchmark |
| Avg Realisation Rate | X% | 🟢 Benchmark |
| LTV : CAC | Xx | 🟢 Benchmark |

**Approval required: [TIER]**

**Deal context**
- Seller: [name]
- Key risks: [value]

**Strategic value to Xero**
- Market impact: [value]
- MRR at risk: [value]
- Employees: [value]
- Competitive landscape: [value]
- Relationship status: [value]
- Tech stack: [value]
- Manual processes: [value]

**Strategic importance to customer**
- Outcomes for partner: [value]

**Deal status**
- Stage: [value]
- Decision makers: [value]
- Expected close: [value]
- Billing currency: [value]

**Incentives & engagement model**
| Dimension | Outcome | Flag |
|-----------|---------|------|
| Initiation Rate | [value] | 🟡 Min / 🟢 Ideal / 🔴 Other |
[... all 11 rows ...]

---

Use 🟡 Min, 🟢 Ideal, 🔴 Other for the incentive flags.

Then ask:
> **Does everything look correct?** Reply "yes" to generate the deal summary and submit to Asana, or tell me what to change.

Wait for confirmation before proceeding. If the seller requests changes, update the relevant answers and re-show the review. Only proceed to Step 8 once the seller confirms.

### Step 8 — Generate the HTML deal summary and upload to Google Drive

Build the HTML string directly in memory from the analysis JSON and answers dict — no file generation needed. Structure:

```html
<!DOCTYPE html><html><head><meta charset="UTF-8">
<title>Deal Summary — [Partner] [[Tier]]</title>
<style>
  body { font-family: Arial, sans-serif; font-size: 13px; max-width: 900px; margin: 40px auto; }
  table { border-collapse: collapse; width: 100%; margin: 10px 0; }
  th { background: #c9daf8; padding: 6px 10px; border: 1px solid #ccc; text-align: left; font-weight: bold; }
  td { padding: 6px 10px; border: 1px solid #ccc; }
  .amber { background: #fff2cc; }
  .green { background: #d9ead3; }
  .red   { background: #f4cccc; }
  h2 { font-size: 14px; border-bottom: 2px solid #2e75b6; padding-bottom: 4px; margin-top: 24px; }
</style></head><body>
[title, subtitle, then sections 1–6 as tables]
</body></html>
```

In the **Incentives & engagement model** table, render each flag as a small coloured label after the outcome text:
- `MIN` → `<span style="background:#fff2cc;border:1px solid #c8a800;border-radius:3px;padding:1px 5px;font-size:10px;font-weight:bold;">Min</span>`
- `IDEAL` → `<span style="background:#d9ead3;border:1px solid #6aa84f;border-radius:3px;padding:1px 5px;font-size:10px;font-weight:bold;">Ideal</span>`
- `CUSTOM` → `<span style="background:#f4cccc;border:1px solid #cc0000;border-radius:3px;padding:1px 5px;font-size:10px;font-weight:bold;">Other</span>`

Then upload using `mcp__04ab6d00__create_file`:
- `title`: `Deal Summary — [Partner Name] [[Approval Tier]]`
- `contentMimeType`: `text/html`
- `textContent`: the full HTML string
- `parentId`: `0ANV1MiQbbu1tUk9PVA`
- Do NOT set `disableConversionToGoogleType` — let Drive convert to a Google Doc

The response returns `{id, mimeType, title}`. Build the URL as: `https://docs.google.com/document/d/{id}/edit`

If the returned mimeType is `text/html` (conversion didn't happen), use `https://drive.google.com/file/d/{id}/view` instead — clicking "Open" in Drive will render it as HTML in the browser.

### Step 9 — Create the Asana approval task


Use `mcp__1c3af366__create_tasks` with:
- `default_assignee`: `me`
- `tasks[0].name`: `Deal Approval — [Partner Name] [[Tier abbreviated]] — [TCV]` where tier is abbreviated: "Sales Director" → "SD", "GM / CRO" → "GM", "Benchmark" → "BM"
- `tasks[0].notes`: formatted plain text with KPI assessment, risks, doc link
- `tasks[0].resource_subtype`: `approval`
- `tasks[0].project_id`: `1214802974574972`
- `tasks[0].section_id`: `1214802974928295`

The Asana task permalink will be in the response as `permalink_url`.

### Step 10 — Confirm to the seller

Reply with a summary:
> ✅ **Deal intake complete for [Partner Name]**
> - Approval required: **[TIER]**
> - Deal Summary: [Google Drive link]
> - Asana task: [Asana permalink]

---

## Key file locations

- Workflow script: `/sessions/stoic-admiring-allen/mnt/No UI Deal Intake/deal_intake/workflow.py`
- Salesforce flat file: `/sessions/stoic-admiring-allen/mnt/No UI Deal Intake/salesforce_data.csv`
- Analysis output: `/tmp/analysis.json`
- Answers input: `/tmp/answers.json`
- HTML output: `/tmp/deal_summary_*.html`
- Workspace folder: `/sessions/stoic-admiring-allen/mnt/No UI Deal Intake/`

## Notes

- The Asana workspace GID is `41261904390129`
- The Deal Model template URL is: https://docs.google.com/spreadsheets/d/1MoIlq_cYL56YV-zOVViU-DwpwMJ2d_VV2Yf74HKnGio/
- Use plain `notes` not `html_notes` for Asana tasks — the XML validator rejects `<` `>` `&` characters
- Google Drive stores as HTML (not native Google Doc) — fully viewable in browser
- When Salesforce MCP is live, replace flat file enrichment with `enrich_from_salesforce()`
- When Gong token is provided, set `GONG_ACCESS_KEY` and `GONG_ACCESS_KEY_SECRET` env vars
