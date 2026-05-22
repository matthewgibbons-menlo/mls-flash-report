# InfoSec Review: Marketing Flash Report App — Snowflake Access Request

**Requested by:** \[Your name\], Marketing **Date:** May 2026 **Decision needed by:** \[Target date\] **Risk classification request:** Low

---

## TL;DR

We are building an internal Streamlit web app that automates the weekly Marketing Flash Report. The app needs **read-only access to one Snowflake schema** containing aggregated content engagement data (the same data that already feeds the Tableau Content Consumption dashboard).

**This is not a Claude-queries-Snowflake architecture.** A Python service queries Snowflake using a least-privilege service account, aggregates the results into weekly metric rollups, and sends only those aggregates to the Claude API for narrative generation. Claude itself never touches Snowflake, never sees row-level data, and never sees customer-identifying information.

The pattern is identical to the already-approved Salesforce \+ Claude flow, just pointed at a different read-only data source.

---

## What the app does

1. **Monday 6:00 AM:** A scheduled Python job pulls weekly KPIs from four sources — GA4, Umami, Salesforce (already approved), and a single read-only Snowflake schema.  
2. **Aggregation:** The job computes weekly, quarter-to-date, and year-to-date totals and writes them to a separate `FLASH_REPORT` schema in Snowflake.  
3. **Narrative generation:** The aggregated numbers (\~30 KPIs, no row-level data) are sent to the Claude API along with target goals. Claude returns prose commentary on pacing and variances.  
4. **Presentation:** Marketing leadership (5–10 users) views the report via an authenticated Streamlit web app and exports to `.pptx`/`.xlsx` for distribution.

---

## Data flow diagram

```
   ┌─────────────────────┐
   │   Snowflake         │
   │   (existing data)   │
   │                     │
   │   Read-only access  │  ◄──── Python service (service account, key-pair auth)
   │   to ONE schema     │        - SELECT only, on specified tables
   │                     │        - No DDL, no cross-schema access
   └─────────┬───────────┘
             │
             │  Raw rows queried via Snowflake Connector (Python)
             ▼
   ┌─────────────────────┐
   │   Python Service    │
   │   (Streamlit app)   │
   │                     │
   │   Aggregates rows   │
   │   into weekly KPIs  │
   └─────────┬───────────┘
             │
             │  ONLY aggregated metrics sent to Claude
             │  (e.g., "gated_downloads_weekly: 452")
             │  No PII, no row-level data, no customer IDs
             ▼
   ┌─────────────────────┐
   │   Anthropic API     │
   │   (Claude)          │
   │                     │
   │   Returns prose     │
   │   commentary        │
   └─────────┬───────────┘
             │
             │  Narrative text written back to FLASH_REPORT schema
             ▼
   ┌─────────────────────┐
   │   Streamlit UI      │
   │   (authenticated,   │
   │    Google OAuth,    │
   │    @menlosec.com    │
   │    domain only)     │
   └─────────────────────┘
```

---

## What Claude sees vs. what it doesn't

### Claude sees (sent in API calls)

- Aggregated weekly/QTD/YTD numbers for \~30 KPIs (e.g., "engaged\_sessions\_weekly: 2,406")  
- Target goals (e.g., "engaged\_sessions\_target\_qtd: 4,435")  
- Pre-computed variance percentages (e.g., "pacing\_pct: 54.25")  
- Top-level asset titles in aggregate (e.g., "top\_gated\_asset: State of Browser Security Report — 285 downloads")

### Claude does NOT see

- Row-level data of any kind  
- Customer identifiers, account names, or contact information  
- Individual user behavior (sessions tied to a specific person)  
- Salesforce opportunity dollar amounts at the deal level (only aggregated MQL/S1 counts)  
- IP addresses, email addresses, or any PII  
- Any data outside the specific FLASH\_REPORT schema

---

## Why this is the same risk profile as the approved Salesforce flow

| Dimension | Approved Salesforce \+ Claude | Proposed Snowflake \+ Claude |
| :---- | :---- | :---- |
| Architecture | App queries SFDC, sends aggregates to Claude | App queries Snowflake, sends aggregates to Claude |
| Claude's role | Receives aggregated data in prompts | Receives aggregated data in prompts |
| Direct Claude access to data source | None | None |
| Data sensitivity | Customer pipeline data | Aggregated content engagement metrics (lower sensitivity) |
| Access control | Service account, scoped | Service account, scoped to single schema |
| PII exposure | Limited via aggregation | None — schema contains no PII |

**The proposed architecture is materially lower-risk than the already-approved Salesforce flow** because the underlying data is less sensitive (marketing content engagement vs. customer pipeline) and the aggregation happens before any data leaves our infrastructure.

---

## Security controls in place

### Snowflake side

- Dedicated service account `SVC_FLASH_REPORTER` with RSA key-pair authentication (no passwords)  
- Least-privilege role `FLASH_REPORTER_ROLE`:  
  - `SELECT` only on a specified list of source tables  
  - `INSERT`/`UPDATE` only within the new `FLASH_REPORT` schema  
  - No `DROP`, no cross-schema writes, no admin privileges  
- Full audit trail: every query and write is logged in Snowflake's query history and the app's own `RUN_HISTORY` and `AUDIT_LOG` tables

### Anthropic API side

- API key stored in 1Password, injected at runtime via environment variables  
- Stateless API calls — no data persistence at Anthropic  
- Request to enable Zero Data Retention on our Anthropic plan (if not already enabled)  
- All payloads sent to Claude are reviewed and logged before transmission

### Application side

- Hosted on \[Render / Cloud Run — TBD\] with HTTPS only  
- Google OAuth authentication restricted to `@menlosec.com` domain  
- Explicit user allowlist (5–10 named individuals)  
- All user actions (report views, narrative regenerations, exports) logged to `AUDIT_LOG`  
- Per-user rate limits on narrative regeneration (10/week)  
- Secrets managed via 1Password Business shared vault with quarterly rotation reminders  
- Source code in a private repository with `pip-audit` running on every PR

### Network side

- Outbound HTTPS only to: `*.snowflakecomputing.com`, `*.googleapis.com`, `api.anthropic.com`, the Salesforce instance, and the Umami host  
- No inbound exposure beyond the authenticated Streamlit port

---

## Specific access requested

**Read-only `SELECT` access** for `FLASH_REPORTER_ROLE` on the following tables (or the equivalent — pending BI team confirmation of exact table names):

- The Snowflake table(s) underlying the Content Consumption Tableau dashboard  
- Specifically the tables/views that produce the "Gated" and "Ungated" asset download metrics

**No other Snowflake access is requested.** The role will not have access to:

- Customer data tables  
- Financial/billing data  
- Any HR or personnel data  
- Any schema other than the source content schema and the new `FLASH_REPORT` schema

---

## Business value

The current Flash Report process consumes 6–8 analyst hours per week and is prone to manual error. The Marketing Leadership team relies on these reports for weekly pacing decisions. Automating this:

- Frees \~30 analyst hours/month for higher-value work  
- Reduces report delivery latency from Monday afternoon to Monday morning  
- Eliminates copy/paste errors that have caused incorrect decisions in past quarters  
- Provides versioned, auditable narrative commentary  
- Standardizes the report format across regions

**Approved Salesforce \+ Claude flows have already established the pattern.** This request extends that pattern to a less-sensitive data source.

---

## Precedent

| Approved access | When | Pattern |
| :---- | :---- | :---- |
| Salesforce \+ Claude | \[Date\] | App queries SFDC → aggregates → Claude |
| Zendesk \+ Claude | \[Date\] | App queries Zendesk → aggregates → Claude |
| **Snowflake (FLASH\_REPORT) \+ Claude** | **This request** | **App queries Snowflake → aggregates → Claude** |

---

## What we need from InfoSec

1. **Confirmation** that this architecture (Python service queries Snowflake; only aggregates sent to Claude) does not fall under the same restriction as agentic Claude tools querying Snowflake directly.  
2. **Approval** for the `FLASH_REPORTER_ROLE` to receive `SELECT` access on the specific content consumption tables.  
3. **Any additional controls** InfoSec wants to see (e.g., specific audit log retention, ZDR confirmation, additional access reviews) before sign-off.

---

## Appendix A — Documents available on request

- Full project brief (`PROJECT_BRIEF.md`)  
- Snowflake DDL and grant script (`snowflake_setup.sql`)  
- Data mapping schema (`data_mapping.yaml`)  
- Sample Claude API payload (showing aggregated-only data)  
- Sample Claude API response (narrative output)

---

## Appendix B — Contacts

- **Project owner:** \[Your name\], Marketing  
- **BI sponsor:** \[BI guy's name\], Business Intelligence  
- **Technical implementer:** Built via Claude Code, code reviewed by \[your name\]  
- **InfoSec reviewer:** \[TBD\]

