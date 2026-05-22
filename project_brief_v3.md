# Project Brief: Marketing Flash Report Web App (v1)

**Document Version:** 3.0 **Prepared For:** Claude Code (Implementation Agent) **Project Codename:** MLS Flash — Web App **Delivery Target:** Production-deployed Streamlit application \+ scheduled ingestion job **Timeline:** 1–2 weeks to v1 **Audience:** Marketing leadership (5–10 users)

---

## 0\. What Changed in v3 (Read First)

This version of the brief differs from v2 in one critical way:

**Tableau-sourced content consumption data is OUT OF SCOPE for v1, pending InfoSec approval.** The data lives in Snowflake but our org's AI governance policy currently restricts access to Snowflake sources beyond Salesforce and Zendesk. A separate approval process is underway.

**In scope for v1:** GA4, Umami, Salesforce. **Deferred to v1.1:** Snowflake content consumption tables (gated/ungated downloads, top assets, Tableau-managed engagement metrics).

The architecture must make adding the deferred source a localized change — a new file in `ingestion/`, new entries in `data_mapping.yaml`, no surgery on the rest of the system.

---

## 1\. Executive Summary

### 1.1 Objective

Replace the manual weekly Marketing Flash Report workflow with a Streamlit web application backed by an automated Python ingestion pipeline. The system delivers the report through a browser-based dashboard with one-click export to `.pptx` and `.xlsx`, since the organization does not grant LLM agents write access to Google Workspace.

### 1.2 Automation Scope (v1)

- **Ingestion:** GA4 (web analytics), Umami (privacy-focused web analytics), Salesforce (pipeline, MQL, S1, S4 metrics).  
- **Metric computation:** Weekly Actuals, QTD, YTD, WoW variance, and pacing vs. target for web and pipeline KPIs.  
- **Narrative generation:** Claude-authored commentary covering pacing, variances, and recommendations — produced on the Monday schedule and regeneratable on demand from the UI with optional custom prompts.  
- **Presentation:** Streamlit dashboard rendering the full report; export to PowerPoint and Excel for users to upload manually into Google Drive.

### 1.3 Out of Scope for v1 (Tracked for v1.1)

- Snowflake-sourced content consumption metrics (gated downloads, ungated downloads, top-performing assets, asset-level engagement)  
- These metrics exist today in a Tableau dashboard fed by Snowflake; access is pending InfoSec approval  
- When approved, integration should require only:  
  - A new client module at `ingestion/snowflake_content_client.py`  
  - New entries in `schemas/data_mapping.yaml` under the `content_consumption` group  
  - New UI components on a "Content KPIs" page (skeleton page should exist in v1, populated in v1.1)

### 1.4 Operational Impact

| Dimension | Current (Manual) | v1 (Web App) |
| :---- | :---- | :---- |
| Time-to-report | 6–8 analyst hours | \< 10 min (export \+ upload) |
| Refresh latency | Weekly only | Real-time once data lands |
| Narrative iteration | Single-shot | On-demand regeneration |
| Versioning | Filename conventions | Snowflake-backed history |
| Workspace permissions | Required | Not required (export workaround) |

### 1.5 Constraint-Driven Design Notes

- **No Google Workspace writes:** Snowflake is system of record; Streamlit is the canonical view; `.pptx`/`.xlsx` exports bridge to Google distribution.  
- **No agentic Claude access to Snowflake:** Python service queries Snowflake using a service account; only aggregated metrics flow to the Claude API. This architectural constraint must be preserved if and when Snowflake content sources are approved in v1.1.

---

## 2\. Technical Stack Specification

### 2.1 Runtime & Framework

| Concern | Choice | Rationale |
| :---- | :---- | :---- |
| Language | Python 3.11+ | Single-language stack across ingestion and UI |
| UI framework | Streamlit 1.30+ | Fastest path to a polished internal dashboard |
| Backend job | Same Python codebase, invoked via cron | No separate API layer needed for v1 |
| Packaging | `uv` or Poetry | Deterministic builds |
| Hosting | Render (default recommendation) | First-class cron \+ secrets \+ OAuth |
| Authentication | Google OAuth, `@menlosec.com` domain only | Restricted to \~5–10 named users |

### 2.2 Required Libraries

| Concern | Library |
| :---- | :---- |
| UI | `streamlit` |
| Auth proxy | Render-native OAuth, or `streamlit-oauth` |
| Data warehouse (output schema only) | `snowflake-connector-python[pandas]` |
| Source ingestion | `google-analytics-data` (GA4), `requests` (Umami), `simple-salesforce` (Salesforce) |
| LLM narrative | `anthropic` |
| Export — PowerPoint | `python-pptx` |
| Export — Excel | `openpyxl` |
| Scheduling | Render Cron Jobs (managed) or system `cron` |
| Config & secrets | `python-dotenv`, `pydantic-settings`, Streamlit `st.secrets` |
| Secrets source of truth | 1Password Business (manual sync to hosting platform for v1) |
| Retries | `tenacity` |
| Logging | `structlog` |
| Testing | `pytest`, `pytest-mock`, `responses` |

### 2.3 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  Monday 06:00 — CRON-triggered ingestion job (Render Cron)      │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐                     │
│  │  Umami   │  │   GA4    │  │ Salesforce │                     │
│  │  API     │  │  API     │  │   API      │                     │
│  └────┬─────┘  └────┬─────┘  └─────┬──────┘                     │
│       └─────────────┴──────────────┘                            │
│                     │                                           │
│                     ▼                                           │
│       ┌──────────────────────────┐                              │
│       │  Snowflake.FLASH_REPORT  │  (output schema)             │
│       │   - WEEKLY_METRICS       │                              │
│       │   - TARGETS              │                              │
│       │   - NARRATIVES           │                              │
│       │   - RUN_HISTORY          │                              │
│       │   - AUDIT_LOG            │                              │
│       └────────────┬─────────────┘                              │
│                    │                                            │
│                    ▼                                            │
│       ┌──────────────────────────┐                              │
│       │  Claude API → narrative  │  (scheduled draft)           │
│       └──────────────────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              Streamlit Web App (Render, always-on)              │
│  ┌─────────┬─────────┬───────────┬──────────┬────────────────┐  │
│  │Overview │ Web KPIs│ Pipeline  │ Content  │   Narrative    │  │
│  │ Pacing  │         │ (SFDC)    │ (v1.1)   │   (Claude)     │  │
│  └─────────┴─────────┴───────────┴──────────┴────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ [Regenerate Narrative] [Export .pptx] [Export .xlsx]   │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  Auth: Google OAuth, restricted to @menlosec.com domain         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                User downloads files and uploads
                into existing Google Drive workflow
```

### 2.4 Module Layout

```
flash_report/
├── app.py                              # Streamlit entry point
├── jobs/
│   └── weekly_ingestion.py             # CRON-invoked ingestion + scheduled narrative
├── config.py                           # pydantic settings, st.secrets bridge
├── ingestion/
│   ├── __init__.py                     # ingestion source registry
│   ├── base.py                         # AbstractIngestionClient interface
│   ├── ga4_client.py
│   ├── umami_client.py
│   ├── salesforce_client.py
│   └── snowflake_content_client.py     # STUB ONLY in v1 — raises NotImplementedError
├── analytics/
│   ├── queries.sql
│   ├── snowflake_reader.py             # reads from FLASH_REPORT schema only
│   ├── snowflake_writer.py             # writes to FLASH_REPORT schema only
│   └── pacing.py                       # WoW, QTD, YTD, target variance math
├── narrative/
│   ├── claude_client.py
│   ├── prompt_templates/
│   │   ├── scheduled.txt
│   │   └── on_demand.txt
│   └── narrative_store.py              # versioned narratives in Snowflake
├── ui/
│   ├── pages/
│   │   ├── 1_Overview.py
│   │   ├── 2_Web_KPIs.py
│   │   ├── 3_Pipeline_KPIs.py
│   │   ├── 4_Content_KPIs.py           # PLACEHOLDER in v1 — shows "Coming in v1.1" banner
│   │   └── 5_Narrative.py
│   ├── components/
│   │   ├── kpi_card.py
│   │   ├── pacing_grade.py
│   │   └── variance_table.py
│   └── auth.py
├── export/
│   ├── pptx_builder.py
│   ├── xlsx_builder.py
│   └── templates/
│       ├── flash_report_template.pptx
│       └── flash_report_template.xlsx
├── fiscal/
│   └── calendar.py                     # FY27 = Feb 2026–Jan 2027, quarters defined
├── schemas/
│   └── data_mapping.yaml
├── tests/
├── .env.example
├── pyproject.toml
└── README.md
```

---

## 3\. Functional Requirements

### 3.1 Phase 1 — Scheduled Ingestion (CRON job)

**Trigger:** `0 6 * * 1` (Monday 06:00 local time)

**Behavior:**

- Pull from GA4, Umami, and Salesforce sequentially.  
- Normalize all source data to the canonical schema in `schemas/data_mapping.yaml`.  
- `MERGE` into `FLASH_REPORT.WEEKLY_METRICS` keyed on `(fiscal_year, fiscal_quarter, fiscal_week, source_platform, metric_name, granularity)` for idempotency.  
- Write a row into `FLASH_REPORT.RUN_HISTORY` capturing start/end timestamps, row counts per source, and status.

**Source-specific notes:**

- **GA4:**  
    
  - Property ID: `properties/319899192`  
  - Use Google Analytics Data API v1  
  - Metrics: `sessions`, `engagedSessions`, `bounceRate`, `averageSessionDuration`, `conversions`, `totalUsers`  
  - Dimensions: `date`, `sessionDefaultChannelGroup`, `landingPage`  
  - Date range: prior Monday 00:00 → prior Sunday 23:59 (source timezone)


- **Umami:**  
    
  - Base URL: `https://cloud.umami.is`  
  - Website ID: `b0ec5dfb-d397-41ab-abb2-1538dc0933a5`  
  - API endpoint: `/api/websites/{websiteId}/stats` and `/api/websites/{websiteId}/pageviews`  
  - Auth: Bearer token (API key in `UMAMI_API_TOKEN`)  
  - Use for: total page views, visitor counts as a privacy-preserving cross-check against GA4


- **Salesforce:**  
    
  - Auth: Connected App (already approved org-wide for Claude integration)  
  - Objects: `Lead`, `Opportunity`, `Campaign`, `CampaignMember`  
  - Key fields: `Status`, `Stage`, `Region__c` (or equivalent), `CreatedDate`, `LastModifiedDate`, `Amount`, `Campaign.Type`  
  - Computed metrics: weekly MQL count, S1 count, S4 count (by region), new pipeline ARR, marketing-sourced closed-won  
  - SOQL queries must use a `LastModifiedDate >=` filter to incrementally pull only new/changed records


- **Snowflake content consumption:** **NOT IMPLEMENTED IN V1.** The stub module at `ingestion/snowflake_content_client.py` should exist as a class skeleton that raises `NotImplementedError("Pending InfoSec approval — see v1.1")` if invoked. This makes the v1.1 expansion a localized change.

**Error handling:**

- `tenacity` exponential backoff (`min=4`, `max=60`, 5 attempts) on every external call.  
- Partial-load detection: row count must hit ≥95% of the rolling 4-week median for that source, else raise `IncompleteIngestionError` (logged but does not halt other sources).  
- A single source failure does *not* halt the whole job — the failed source's previous week's data is reused with a `is_stale=true` flag, and the UI surfaces a banner.  
- All failures emit a Slack webhook alert.

### 3.2 Phase 2 — Scheduled Narrative Generation

**Trigger:** Immediately after Phase 1 completes, in the same cron-invoked job.

**Behavior:**

- Load just-populated metrics \+ historical baselines (rolling 4-week, prior-quarter same-week) \+ target goals from `FLASH_REPORT.TARGETS`.  
- Build the structured prompt from `narrative/prompt_templates/scheduled.txt`.  
- Call the Claude API. Constrain output with delimited sections: *Pacing*, *Variances*, *Recommendations*.  
- Validate the response against expected structure; on parse failure, regenerate once with a corrective system message.  
- Store narrative in `FLASH_REPORT.NARRATIVES` with full versioning (`narrative_id`, `version`, `is_scheduled=true`, `generated_by='SYSTEM'`).

**Critical guardrail:** The system prompt must instruct Claude to ground commentary only in the supplied numeric payload and refuse to invent figures. This is non-negotiable for the InfoSec narrative.

**Error handling:**

- Retries on `429`/`5xx` (3 attempts).  
- If both attempts fail, insert a `status='failed'` placeholder. The report still renders without commentary; the UI offers a regenerate button.

### 3.3 Phase 3 — Streamlit App: Read & Render

**Behavior:**

- On load, read latest `fiscal_week` from `RUN_HISTORY` and pull corresponding rows from `WEEKLY_METRICS` and `NARRATIVES`.  
- Render five pages:  
  1. **Overview** — headline KPIs, pacing grades (A/B/C/D vs. target), data-freshness banner per source, export buttons.  
  2. **Web KPIs** — GA4 \+ Umami metrics; weekly / QTD / YTD columns; WoW variance with conditional formatting.  
  3. **Pipeline KPIs** — Salesforce metrics by region (NA, EMEA, APAC, Japan, Public Sector); MQL/S1/S4 funnels.  
  4. **Content KPIs** — **placeholder page in v1** with banner: "Content consumption metrics coming in v1.1 (pending InfoSec approval for Snowflake source access)."  
  5. **Narrative** — current narrative version with side-panel for prior versions and on-demand regeneration controls.  
- Fiscal-week selector in sidebar enables historical viewing.  
- Caching: `@st.cache_data` with 5-minute TTL on Snowflake reads.

**Error handling:**

- Snowflake unreachable → render last-known-good cached state with a retry button.  
- Stale source flag → yellow banner on affected metrics.

### 3.4 Phase 4 — On-Demand Narrative Regeneration

**Behavior:**

- Narrative page exposes:  
  - **Regenerate** button (default scheduled prompt)  
  - **Custom prompt** free-text field for analyst overlay  
  - **Tone selector**: Executive / Analytical / Brief  
- On submit, app calls `claude_client` with user overlay, writes new versioned row to `NARRATIVES` with `is_scheduled=false` and `generated_by=user_email`.  
- New version becomes default; prior versions accessible via version-history dropdown.

**Guardrails:**

- Locked system prompt prepended to every call; cannot be overridden by user input.  
- Per-user rate limit: 10 regenerations per fiscal week.  
- Every regeneration logs to `AUDIT_LOG` with user identity, prompt overlay, and token usage.

**Error handling:**

- API errors surfaced clearly in the UI.  
- Malformed output triggers one retry; if still bad, preserve prior version and show error toast.

### 3.5 Phase 5 — Export

**Behavior:**

- **Download .pptx** button uses `python-pptx` against `export/templates/flash_report_template.pptx` — pre-built deck with `{{TOKEN_NAME}}` placeholders.  
- **Download .xlsx** button uses `openpyxl` against `export/templates/flash_report_template.xlsx` with explicit cell targeting.  
- Filenames: `Web_Flash_Report_Q{q}-W{w}_{YYYY-MM-DD}.{ext}`.  
- **Copy section** buttons on each table copy Markdown-formatted versions to clipboard for direct paste into Google Slides/Sheets.

**Templates are an input, not an output.** They must be built manually before v1 ships:

- Take the existing Flash Report master deck  
- Replace every numeric/text value position with `{{TOKEN}}` placeholders matching the data mapping YAML  
- Commit to `export/templates/`

**Error handling:**

- Build to in-memory buffer first; only offer download on successful build.

---

## 4\. Data Mapping Schema

The mapping is the single source of truth for the field-to-destination contract. Fields map to **Streamlit components**, **.pptx template placeholders**, and **.xlsx cell positions**.

### 4.1 Schema Structure

```
metric_key:
  source:
    platform: GA4 | UMAMI | SALESFORCE | SNOWFLAKE_CONTENT  # last one deferred to v1.1
    extraction_logic: <free-form description or query reference>
  granularity: [WEEKLY, QTD, YTD]
  target_metric_name: <key into FLASH_REPORT.TARGETS>
  ui_destination:
    page: "Web KPIs"
    component: kpi_card
    label: "Sessions"
    number_format: "{:,.0f}"
  export_destinations:
    pptx:
      template_placeholder: "{{WEB_SESSIONS_WEEKLY}}"
      slide_index: 2
    xlsx:
      sheet_name: "Web KPIs"
      cell_weekly: "C5"
      cell_qtd: "D5"
      cell_ytd: "E5"
  available_in_version: v1   # or v1.1 for deferred metrics
```

### 4.2 v1 Metric Coverage (Representative)

| Metric Key | Source | UI Page | Status |
| :---- | :---- | :---- | :---- |
| `web_visitors` | Umami | Web KPIs | v1 |
| `web_engaged_sessions` | GA4 | Web KPIs | v1 |
| `web_avg_session_duration` | Umami | Web KPIs | v1 |
| `web_conversion_rate` | GA4 | Web KPIs | v1 |
| `web_handraiser_mqls` | GA4 \+ SFDC | Web KPIs | v1 |
| `mql_inbound` | Salesforce | Pipeline KPIs | v1 |
| `mql_outbound` | Salesforce | Pipeline KPIs | v1 |
| `s1_inbound` | Salesforce | Pipeline KPIs | v1 |
| `s1_outbound` | Salesforce | Pipeline KPIs | v1 |
| `mql_by_region` | Salesforce | Pipeline KPIs | v1 |
| `new_pipeline_arr` | Salesforce | Pipeline KPIs | v1 |
| `gated_asset_downloads` | Snowflake | Content KPIs | **v1.1** |
| `ungated_asset_downloads` | Snowflake | Content KPIs | **v1.1** |
| `top_gated_assets` | Snowflake | Content KPIs | **v1.1** |
| `top_ungated_assets` | Snowflake | Content KPIs | **v1.1** |
| `narrative_summary` | Claude | Narrative | v1 |

### 4.3 Mapping Rules

- All three destinations (UI / pptx / xlsx) for a metric must be declared in the same YAML entry.  
- Metrics flagged `available_in_version: v1.1` are loaded into the schema but skipped at ingestion time; UI shows them with a "Coming soon" treatment.  
- A CI test must confirm the export templates contain every `{{TOKEN}}` declared in YAML — prevents drift.

---

## 5\. Environment & Security Constraints

### 5.1 Credential Inventory

| Credential | Purpose | Storage |
| :---- | :---- | :---- |
| Snowflake private key | Auth to FLASH\_REPORT schema (key-pair) | 1Password → Render env var |
| GA4 service account JSON | Analytics Data API | 1Password → Render env var |
| Salesforce OAuth credentials | SF REST/SOQL access | 1Password → Render env var |
| Umami API token | Umami REST access | 1Password → Render env var |
| Anthropic API key | Claude API | 1Password → Render env var |
| Google OAuth client ID/secret | App user authentication | 1Password → Render env var |
| Streamlit auth cookie secret | Session signing | 1Password → Render env var |
| Slack webhook | Failure alerts | 1Password → Render env var |

### 5.2 Secret Management Pattern

- **Source of truth:** 1Password Business shared vault `MLS Flash Report — Production` (separate `Development` vault for local).  
- **Local development:** Use 1Password CLI (`op run`) to inject secrets without writing them to disk.  
- **Production:** Manual copy from 1Password into Render environment variables (v1 simplicity). Migrate to 1Password Service Accounts in v2 if needed.  
- **Quarterly rotation** for all credentials, tracked via 1Password Watchtower reminders.

### 5.3 `.env.example` Structure

```
# === Snowflake ===
SNOWFLAKE_ACCOUNT=
SNOWFLAKE_USER=SVC_FLASH_REPORTER
SNOWFLAKE_PRIVATE_KEY_PATH=/etc/secrets/snowflake_rsa_key.p8
SNOWFLAKE_WAREHOUSE=
SNOWFLAKE_DATABASE=
SNOWFLAKE_SCHEMA=FLASH_REPORT
SNOWFLAKE_ROLE=FLASH_REPORTER_ROLE

# === GA4 ===
GA4_SERVICE_ACCOUNT_PATH=/etc/secrets/ga4_sa.json
GA4_PROPERTY_ID=properties/319899192

# === Umami ===
UMAMI_BASE_URL=https://cloud.umami.is
UMAMI_WEBSITE_ID=b0ec5dfb-d397-41ab-abb2-1538dc0933a5
UMAMI_API_TOKEN=  # from 1Password at runtime

# === Salesforce ===
SALESFORCE_USERNAME=
SALESFORCE_CLIENT_ID=
SALESFORCE_CLIENT_SECRET=
SALESFORCE_DOMAIN=login

# === Anthropic ===
ANTHROPIC_API_KEY=
ANTHROPIC_MODEL=claude-opus-4-7
ANTHROPIC_MAX_TOKENS=1500

# === Streamlit & Auth ===
STREAMLIT_AUTH_COOKIE_SECRET=
GOOGLE_OAUTH_CLIENT_ID=
GOOGLE_OAUTH_CLIENT_SECRET=
ALLOWED_EMAIL_DOMAINS=menlosec.com
ALLOWED_USER_EMAILS=

# === Runtime ===
LOG_LEVEL=INFO
ENVIRONMENT=production
SLACK_ALERT_WEBHOOK=
FISCAL_YEAR_START=2026-02-01
```

### 5.4 Authentication

- **Google OAuth restricted to `@menlosec.com`** via Render-native OAuth or `streamlit-oauth`.  
- Explicit allowlist of 5–10 email addresses configured via `ALLOWED_USER_EMAILS`.  
- Sessions signed with `STREAMLIT_AUTH_COOKIE_SECRET`.

### 5.5 Security Constraints

- **Snowflake role:** `SELECT` only on source data (none in v1), `INSERT`/`UPDATE`/`MERGE` on `FLASH_REPORT.*` only. No DDL beyond initial schema creation.  
- **No secrets in logs:** Pydantic `repr=False`, structlog redaction of sensitive keys.  
- **Audit trail:** Every narrative regen, export, and report view logs to `AUDIT_LOG`.  
- **Rate limits:** Per-user regen cap (10/week) enforced server-side.  
- **Network egress:** HTTPS only to `*.snowflakecomputing.com`, `*.googleapis.com`, `api.anthropic.com`, Salesforce instance, `cloud.umami.is`.  
- **No inbound exposure** beyond the auth-proxied Streamlit port.  
- **Dependency scanning:** `pip-audit` in CI on every PR.

---

## 6\. Fiscal Calendar Definition

Encode the following in `fiscal/calendar.py`:

- **FY27** \= February 2026 through January 2027  
- **Quarters:**  
  - Q1: Feb–Apr  
  - Q2: May–Jul  
  - Q3: Aug–Oct  
  - Q4: Nov–Jan  
- **Week 1** of each quarter is the **first full week** (Monday–Sunday) of the quarter's first month  
- Given the existing reports show "Q1-W12" landing in late April, this convention should reproduce that  
- Provide utility functions:  
  - `get_current_fiscal_week(date) -> (fiscal_year, fiscal_quarter, fiscal_week)`  
  - `get_week_date_range(fy, q, w) -> (start_date, end_date)`  
  - `get_qtd_date_range(fy, q, as_of) -> (start_date, end_date)`  
  - `get_ytd_date_range(fy, as_of) -> (start_date, end_date)`

Edge case to confirm with stakeholder: if Q1 starts mid-week (Feb 1 is a Sunday), is the first full week Feb 2–8, or does it include the partial Feb 1? Default to **first full Monday-starting week**; confirm before implementation.

---

## 7\. Targets Source

Targets live in the Google Sheet at: `https://docs.google.com/spreadsheets/d/1T54wNfxWhtn3yPJ4anc0Q3xbpJdsetyb0ovk79NjbvY/edit?gid=1159078579`

**Loading approach:**

- One-time export of targets to a CSV  
- CSV loaded into `FLASH_REPORT.TARGETS` via a manual `targets_loader.py` script run quarterly  
- Avoids runtime dependency on Google Sheets API for slowly-changing data

**Schema** (already defined in `snowflake_setup.sql`):

- `(fiscal_year, fiscal_quarter, metric_name, target_type, target_value, seasonality_pct)`  
- `seasonality_pct` captures quarter-of-year distribution (Q1: 27%, Q2: 30%, Q3: 30%, Q4: 13% for MQLs per the source deck)

---

## 8\. Acceptance Criteria for v1

The implementation is complete when:

1. The Monday cron job runs end-to-end against GA4, Umami, and Salesforce without manual intervention, landing metrics \+ scheduled narrative in Snowflake.  
2. The Streamlit app renders the current week's report within 3 seconds of load on a warm cache.  
3. Any of the 5–10 authorized `@menlosec.com` users can authenticate via Google OAuth, view the report, and regenerate the narrative with a custom prompt.  
4. Both `.pptx` and `.xlsx` exports produce files that visually match the existing master templates and open cleanly in Google Slides / Google Sheets after manual upload.  
5. The Content KPIs page renders with a "Coming in v1.1" banner and no errors — no broken UI elements from the deferred source.  
6. A single source failure (e.g., Umami API outage) does not block the report — affected metrics show a `stale` banner; the rest of the report renders normally.  
7. Every narrative regeneration is captured in `AUDIT_LOG` with user identity, prompt overlay, and version.  
8. No credential value appears in source control, logs, or exception traces.  
9. A dry-run mode (`python -m flash_report.jobs.weekly_ingestion --dry-run`) executes against a sandbox Snowflake schema without touching production data.  
10. The `snowflake_content_client.py` stub exists, raises `NotImplementedError` clearly, and the `data_mapping.yaml` entries for v1.1 metrics are present but flagged.

---

## 9\. v1.1 Expansion Path (Reference Only — Do Not Build)

When InfoSec approves Snowflake content source access, v1.1 work consists of:

1. Snowflake admin grants `SELECT` on the BI-team-identified tables to `FLASH_REPORTER_ROLE` (one `GRANT` statement)  
2. Implement `ingestion/snowflake_content_client.py` (replace `NotImplementedError` stub)  
3. Add the source's metrics to the ingestion job's source registry  
4. Populate the placeholder Content KPIs page with real components  
5. Update `.pptx` and `.xlsx` templates to include the new metric placeholders  
6. Update Claude's prompt template to incorporate the new metrics into narrative generation  
7. Update the data flow diagram in the InfoSec one-pager to reflect the production reality

**No core architecture changes should be required.** If they are, the v1 design failed an extensibility requirement.

---

## Pre-Handoff Checklist for Claude Code

Before starting the build:

- [x] **GA4 property ID:** `properties/319899192`  
- [x] **Umami base URL \+ website ID:** `cloud.umami.is`, `b0ec5dfb-d397-41ab-abb2-1538dc0933a5`  
- [x] **Salesforce credentials:** Pre-approved org-wide for Claude integration  
- [x] **Fiscal calendar:** FY27 \= Feb 2026–Jan 2027, quarters defined, Week 1 \= first full week  
- [x] **Targets spreadsheet:** Located; loading approach defined (manual CSV → Snowflake)  
- [x] **Hosting:** Render (recommended default)  
- [x] **Auth:** Google OAuth, `@menlosec.com` domain restriction  
- [ ] **Snowflake admin:** Runs `snowflake_setup.sql` and provides account ID \+ private key file  
- [ ] **OAuth client:** Created in Google Cloud Console; client ID \+ secret stored in 1Password  
- [ ] **1Password vault:** `MLS Flash Report — Production` and `Development` vaults created with relevant team members  
- [ ] **Export templates:** `.pptx` and `.xlsx` masters manually tokenized and committed to `export/templates/`  
- [ ] **Allowlist:** Specific `@menlosec.com` email addresses of the 5–10 v1 users  
- [ ] **Slack webhook:** Provisioned for failure alerts  
- [ ] **Umami API key:** Rotated (compromised in chat) and re-stored in 1Password

Items deferred to v1.1 are tracked separately and explicitly out of scope.  
