# Industry Onboarding Checks

CoCo skill for verifying new hire onboarding completeness for **Industry Principal & Architect** employees. Runs automated SQL and Slack checks against Pigment, Sigma, Workday, and Snowflake-certified reporting tables, and flags manual checks that require human follow-up.

**Active employees only.** Every query joins `SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY` on `IS_EMPLOYEE_ACTIVE = TRUE`. Terminated or inactive employees are silently excluded — they are never checked.

---

## What It Checks

All 12 onboarding tasks are organized by the three phases from the [Industry Principals & Architects Process Repo](https://docs.google.com/spreadsheets/d/1lC4-qnDxQzGzYwlQ5_y6XPON3ilkx-B2-gZgfP4GdTY/edit?gid=0#gid=0):

| # | Phase | Check | Type | Data Source |
|---|-------|-------|------|-------------|
| 1 | In-Year Planning | Employee Record | ✅ Automated | `IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR` |
| 2 | In-Year Planning | Territories | ✅ Automated | `IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR` |
| 3 | In-Year Planning | ETM Alignments | ✅ Automated | `IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR` |
| 4 | In-Year Planning | Industry Alignments | ⚠️ Manual Check | Pigment UI |
| 5 | In-Year Planning | Targets | ✅ Automated | `IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR` |
| 6 | In-Year Planning | Baselines (all 12 months) | ✅ Automated | `IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR` |
| 7 | In-Year Planning | Specialist Metadata | ✅ Automated | `IT.PIGMENT.RAW_SPECIALIST_ATTRIBUTES_INYR` |
| 8 | Access | Sigma Access | ✅ Automated | `SIGMA_COMPUTING_DATA_SHARE.AUDIT_LOGS.USERS` |
| 9 | Access | Slack: #cx-specialists | ✅ Automated | Slack MCP (Architects only) |
| 10 | Access | Slack: #cx-specialists-managers | ✅ Automated | Slack MCP (Architect Mgmt only) |
| 11 | Systems | CiQ | ⚠️ Manual Check | CiQ |
| 12 | Systems | Attainment Dashboard | ✅ Automated | `SNOW_CERTIFIED.SALES_PERFORMANCE.AGG_INDUSTRY_USER_CONSUMPTION_ATTAINMENT` |

**10 automated checks** · **2 manual checks** (flagged `MANUAL CHECK` with action guidance)

**Result values:** `PASS` · `NEEDS REVIEW` · `MANUAL CHECK`

---

## Data Sources

| Table | Purpose |
|-------|---------|
| `IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR` | Checks 1–6: comp plan, territories, ETM, targets, baselines |
| `IT.PIGMENT.RAW_SPECIALIST_ATTRIBUTES_INYR` | Check 7: specialist group/sub-group metadata |
| `SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY` | Active employee gate (`IS_EMPLOYEE_ACTIVE = TRUE`) + enrichment (email, title, region, hire date, manager) |
| `SIGMA_COMPUTING_DATA_SHARE.AUDIT_LOGS.USERS` | Check 8: Sigma account status and license type |
| Slack MCP (`mcp_slack-snow_slack_list_channel_members`) | Checks 9–10: channel membership for `#cx-specialists` and `#cx-specialists-managers` |
| `SNOW_CERTIFIED.SALES_PERFORMANCE.AGG_INDUSTRY_USER_CONSUMPTION_ATTAINMENT` | Check 12: attainment dashboard presence |

---

## Modes

### Single-Person Mode
Checks one employee by EEID. Requires the employee's Snowflake email for the Sigma Access check.

### Whole-Org Mode
Runs all 10 automated checks across every **Workday-active** employee in the Pigment roster. Emails for Sigma and Slack matching are sourced from `D_EMPLOYEE_WORKDAY` — no manual email input required.

The count query used to scope the run:
```sql
SELECT COUNT(DISTINCT r.EEID) AS total_specialists
FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
WHERE r.EEID IS NOT NULL AND r.ACTIVE_RECORD = TRUE
```

---

## How to Use

Trigger phrases:
- `industry onboarding checks`
- `check if [name] is onboarded`
- `run onboarding readiness for the industry org`
- `is my industry specialist set up`
- `new hire check`

The skill will:
1. Ask: one person or whole org?
2. Confirm active employee count before running
3. Dynamically detect the active fiscal year
4. Run all 10 automated checks in parallel (SQL + Slack MCP)
5. Present a phase-grouped summary with `PASS` / `NEEDS REVIEW` and action steps
6. Surface the 2 manual checks (`MANUAL CHECK`) with what to verify and where
7. Write a **CSV audit log** + **HTML KPI report** to `audits/`

---

## Output Format

### Single-Person (chat summary)
```
=== ONBOARDING CHECK RESULTS: Jane Doe (EEID: 20331) ===

PHASE 1: IN-YEAR PLANNING
  ✅ Check 1 — Employee Record        PASS
  ✅ Check 2 — Territories            PASS
  ❌ Check 3 — ETM Alignments         NEEDS REVIEW  → Set Primary and Secondary ETM in Pigment
  ⚠️  Check 4 — Industry Alignments   MANUAL CHECK  → Verify directly in Pigment with manager
  ✅ Check 5 — Targets                PASS
  ✅ Check 6 — Baselines              PASS
  ✅ Check 7 — Metadata               PASS

PHASE 2: ACCESS
  ✅ Check 8 — Sigma Access           PASS
  ✅ Check 9 — Slack: CX-Specialists  PASS
  ✅ Check 10 — Slack: Managers       PASS  [Architect Mgmt only]

PHASE 3: SYSTEMS
  ⚠️  Check 11 — CiQ                  MANUAL CHECK  → Verify in CiQ
  ✅ Check 12 — Attainment Dashboard  PASS

AUTOMATED: 1 NEEDS REVIEW, 9 PASS   |   MANUAL CHECKS: 2 require human verification
```

### Whole-Org (chat summary)
A check-level PASS / NEEDS REVIEW / MANUAL CHECK count table followed by a list of all employees with actionable issues, enriched with Workday context (title, region, hire date, manager).

---

## HTML Report

Every whole-org run generates a self-contained HTML report at:
```
~/.snowflake/cortex/skills/industry-onboarding-checks/audits/report_YYYYMMDD_HHMMSS.html
```

The report has three sections:

### 1. KPI Tiles

**Row 1 — org-level summary (3 tiles):**
| Tile | Value | Color |
|------|-------|-------|
| Total Active | All active employees checked | Neutral |
| Needs Review | Employees with ≥1 actionable issue (excl. C7 known gap) | Red if > 0 |
| Passing All Automated | Employees with no actionable issues | Green |

**Row 2 — risk profile (3 tiles, all employees bucketed):**
| Tile | Who | Color |
|------|-----|-------|
| High Priority | C1, C5, C6, or C12 failing — comp or attainment at risk | Red |
| Medium Priority | C8/C9/C10 failing only — access issues, no comp impact | Amber |
| Low / Monitor | C7 metadata gap only — known org-wide issue | Gray |

### 2. Summary Table
Check-by-check row with NEEDS REVIEW count, Passed count, and MANUAL CHECK indicator.

### 3. Action Items
Per-check detail sections for every check with NEEDS REVIEW > 0. Each section shows affected employees in a table, fix instructions, and links to Process Repo / Validations Sheet / IT.

---

## Audit Log (CSV)

Every run writes a CSV to:
```
~/.snowflake/cortex/skills/industry-onboarding-checks/audits/audit_YYYYMMDD_HHMMSS.csv
```

Columns: `run_at`, `eeid`, `name`, `preferred_name`, `business_title`, `region`, `hire_date`, `manager`, `quota_role`, one column per check (`PASS` / `NEEDS REVIEW` / `MANUAL CHECK`), `overall_automated`, `action_needed`

To view historical audits:
```bash
ls ~/.snowflake/cortex/skills/industry-onboarding-checks/audits/
```

---

## Key Notes

- **Active employees only** — `IS_EMPLOYEE_ACTIVE = TRUE` from `D_EMPLOYEE_WORKDAY` is enforced on every query. Inactive employees (terminated, on leave) are silently excluded and never checked.
- **Corp bonus plan employees** are excluded from Targets (Check 5) and Baselines (Check 6) — they have no quotas by design.
- **Slack checks (9 and 10)** are automated via the Slack MCP. Check 9 applies to `Industry Architect` and `Industry Architect Mgmt` roles only; Check 10 applies to `Industry Architect Mgmt` only. Both are skipped (`N/A`) for `Industry GTM` and `Industry GTM Mgmt`.
- **Attainment Dashboard** (Check 12) is often downstream of metadata and baseline gaps. Resolve C1 → C5 → C6 → C7 first, then re-check C12.
- **C7 org-wide gap** — `SPECIALIST_SUB_GROUP = NULL` for all employees is a known gap. Confirm with Field Ops before acting. Employees missing entirely from `RAW_SPECIALIST_ATTRIBUTES_INYR` are a higher priority.
- **Fiscal year dates are dynamic** — the skill derives all 12 month-end dates from live Pigment data. No hardcoded FY values.
- `ACTIVE_RECORD` is a BOOLEAN column — use `ACTIVE_RECORD = TRUE` (not the string `'TRUE'`).
- `QUOTA_START_DATE` is VARCHAR — always cast with `TO_DATE(LEFT(..., 10), 'MM/DD/YYYY')`.

---

## Related Skills

- [`industry-specialists-revops-checks`](../industry-specialists-revops-checks/README.md) — Similar Pigment-only checks for ongoing RevOps validation. Use for periodic reviews; use this skill specifically for new hire onboarding.
- [`industry-alignment-review`](../industry-alignment-review/README.md) — Monthly industry/subindustry classification review.

---

## Source Docs

- [Process Repo (gid=0)](https://docs.google.com/spreadsheets/d/1lC4-qnDxQzGzYwlQ5_y6XPON3ilkx-B2-gZgfP4GdTY/edit?gid=0#gid=0)
- [Validations (gid=829379486)](https://docs.google.com/spreadsheets/d/1lC4-qnDxQzGzYwlQ5_y6XPON3ilkx-B2-gZgfP4GdTY/edit?gid=829379486#gid=829379486)
