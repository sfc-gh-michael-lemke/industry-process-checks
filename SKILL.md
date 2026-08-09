---
name: industry-process-checks
description: "Run Industry team checks across three workflows: (1) New Hire Onboarding — validates onboarding tasks for Industry Principal & Architect employees across In-Year Planning, Access, and Systems phases; (2) EOM Close — checks accounts and opportunities for missing or changed industry data; (3) Recent Transfers — lists employees who transferred in or out in the last 15 days. Use when: checking if a new Industry hire is fully onboarded, running onboarding readiness checks, validating an Industry specialist is set up before ICM sync, auditing the whole org for onboarding gaps, running EOM data quality checks, checking for recent transfers, accounts with consumption but no industry, industry reclassifications on accounts or opps. Triggers: industry onboarding checks, new hire check, onboarding check, industry onboarding, onboarding readiness, is my specialist onboarded, onboarding gaps, check new hire, industry principal onboarding, industry architect onboarding, new hire readiness, onboarding status, recent transfers, transfers last 15 days, EOM close, end of month checks, consumption no industry, account reclassification, opp reclassification, industry changes."
---

# Industry Team Checks

This skill covers all checks defined in the [Industry Principals & Architects Process Repo](https://docs.google.com/spreadsheets/d/1lC4-qnDxQzGzYwlQ5_y6XPON3ilkx-B2-gZgfP4GdTY/edit?gid=0#gid=0) across three workflows:

| Workflow | Trigger | Checks |
|----------|---------|--------|
| **New Hire Onboarding** | New hire joined the team | C1–C12 (employee, comp, access, systems) |
| **EOM Close** | End of month data quality review | Consumption, account & opp reclassification |
| **Recent Transfers** | Identify recent transfers | Employees who transferred in/out last 15 days |

**Active employees only.** All employee queries join `SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY` on `WORKDAY_EMPLOYEE_ID` with `IS_EMPLOYEE_ACTIVE = TRUE`.

Each automatable check returns **PASS** or **NEEDS REVIEW**. Manual checks are flagged as **MANUAL CHECK**.

---

## Workflow

### Step 1: Select Workflow

⚠️ **Always ask this first** using `ask_user_question`:

```
Question: "Which checks do you want to run?"
Options:
  - "New Hire Onboarding"  → proceed to Step 1a (scope: one person or whole org)
  - "EOM Close"            → proceed directly to EOM checks (no scope question needed — always whole-org)
  - "Recent Transfers"     → run Check 13 (INDUSTRY_CHECK_FOR_TRANSFERS) directly
```

---

### Step 1a: Onboarding Scope — One Person or Whole Org?

*(Onboarding workflow only)*

```
Question: "Do you want to check one person or the whole Industry Principal & Architect org?"
Options:
  - "One person" → ask for EEID (text input, defaultValue: "20331")
  - "Whole org"  → proceed to Step 1b
```

**If one person:** add `AND r.EEID = '<EEID>'` to every onboarding query.

**If whole org:** no EEID filter — aggregate PASS/NEEDS REVIEW counts per check, then list every employee that needs review. Run Step 1c to pre-fetch emails before starting the checks.

---

### Step 1b: Whole-Org Confirmation ⚠️

Before running whole-org queries, first run this count:

```sql
SELECT COUNT(DISTINCT r.EEID) AS total_specialists
FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
WHERE r.EEID IS NOT NULL AND r.ACTIVE_RECORD = TRUE
```

Then ask the user via `ask_user_question`:
> "Found **{total_specialists}** active Industry Principals & Architects. This will run automated checks across all of them. Proceed?"
> Options: "Yes, run all checks" / "Cancel"

Only continue if confirmed.

---

### Step 1c: Pre-flight Enrichment (Whole-Org Mode Only)

Before running any checks, fetch enriched context for all active employees and hold the results in memory:

```sql
SELECT
    r.EEID,
    r.D_EXISTING_EMPLOYEES          AS pigment_name,
    e.EMPLOYEE_PREFERRED_NAME       AS preferred_name,
    e.EMPLOYEE_EMAIL,
    e.EMPLOYEE_BUSINESS_TITLE,
    e.EMPLOYEE_REGION,
    e.EMPLOYEE_HIRE_DATE_AT,
    e.EMPLOYEE_MANAGER_NAME         AS workday_manager,
    e.IS_EMPLOYEE_ACTIVE,
    p.SFDC_USER_ID
FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
LEFT JOIN (
    SELECT EEID, MAX(SFDC_USER_ID) AS SFDC_USER_ID
    FROM IT.PIGMENT.PIGMENT_ROSTER
    WHERE EEID IS NOT NULL
    GROUP BY EEID
) p ON r.EEID = p.EEID
WHERE r.EEID IS NOT NULL AND r.ACTIVE_RECORD = TRUE
ORDER BY e.EMPLOYEE_PREFERRED_NAME
```

This enrichment table drives all remaining checks:
- `EMPLOYEE_EMAIL` — required for Check 8 (Sigma) and Checks 9/10 (Slack)
- `IS_EMPLOYEE_ACTIVE` — **INNER JOIN ensures only active employees are returned**; inactive employees (e.g., terminated, on leave) are excluded from all checks automatically
- `EMPLOYEE_PREFERRED_NAME`, `EMPLOYEE_BUSINESS_TITLE`, `EMPLOYEE_REGION`, `EMPLOYEE_HIRE_DATE_AT` — included in the final summary for richer context
- `SFDC_USER_ID` — available if needed for future checks

---

### Step 3: Parallel Execution Plan

> **EOM Close** and **Recent Transfers** workflows skip Steps 3/3b entirely — go directly to Step 3c (EOM) or run Check 13 (Transfers). The parallel plan below applies to **New Hire Onboarding only**.

**For maximum speed, launch all independent operations simultaneously in a single message.** Do not run checks one at a time.

**ROUND 1 — fire ALL of these in one parallel batch immediately after confirmation:**

| Operation | Purpose | Tool |
|-----------|---------|------|
| Step 1c (enrichment) | Provides email; gates Check 8 | `snowflake_sql_execute` |
| Check 1 — Employee Record | Independent | `snowflake_sql_execute` |
| Check 2 — Territories | Independent | `snowflake_sql_execute` |
| Check 3 — ETM Alignments | Independent | `snowflake_sql_execute` |
| Check 5 — Targets | Independent | `snowflake_sql_execute` |
| Check 6 — Baselines | Independent (month-end dates hardcoded for FY2026) | `snowflake_sql_execute` |
| Check 7 — Metadata | Independent (different table) | `snowflake_sql_execute` |
| Check 12 — Attainment | Independent | `snowflake_sql_execute` |
| Check 13 — Transfers | Independent | `snowflake_sql_execute` |
| Slack: collect `#cx-specialists` all pages | Gates Check 9 | `mcp_slack-snow_slack_list_channel_members` (paginate) |
| Slack: collect `#cx-specialists-managers` all pages | Gates Check 10 | `mcp_slack-snow_slack_list_channel_members` (paginate) |

**ROUND 2 — fire as soon as each dependency completes (can also be parallel):**

| Operation | Depends on |
|-----------|------------|
| Check 8 — Sigma Access | Step 1c (employee emails) |
| Check 9 — Slack: CX-Specialists | #cx-specialists member list |
| Check 10 — Slack: CX-Mgrs | #cx-specialists-managers member list |

> **Slack note:** The `mcp_slack-snow_slack_list_channel_members` call returns member emails directly. No separate `mcp_slack-snow_slack_search_users` call is needed — match architect emails from Step 1c directly against the collected channel email sets.

---

### Step 3b: Check Queries

---

#### Phase: In-Year Planning

> All Checks 1–6 plus Checks 11 and 13 run independently in Round 1. Checks 7–9 depend on email/Slack data and run in Round 2.

---

**Check 1 — Employee Record (INDUSTRY_CHECK_EMPLOYEE_RECORD)**
> Confirms employee exists in Pigment with active record and ops ready date.
> **PASS:** `ACTIVE_RECORD = TRUE` and `FILTER_STATUS = 'Ops Ready'`
> **Risk if NEEDS REVIEW:** Comp plan has not been processed by Field Ops.

```sql
SELECT
    r.EEID,
    r.D_EXISTING_EMPLOYEES          AS name,
    r.INSEAT_MANAGER_COMP_START     AS manager,
    r.QUOTA_ROLE,
    r.FILTER_STATUS,
    r.ACTIVE_RECORD,
    r.OPS_READY_DATE_PLANNING_ONLY,
    IFF(r.FILTER_STATUS = 'Ops Ready', 'PASS', 'NEEDS REVIEW') AS check_result,
    IFF(r.FILTER_STATUS = 'Ops Ready', NULL,
        'Contact Field Ops to mark record as Ops Ready in Pigment') AS action_needed
FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
WHERE r.EEID = '<EEID>'
  AND r.ACTIVE_RECORD = TRUE
```

*Whole-org version:* Filter `WHERE r.EEID IS NOT NULL AND r.ACTIVE_RECORD = TRUE AND r.FILTER_STATUS != 'Ops Ready'` to surface only NEEDS REVIEW results directly. Keep the INNER JOIN to D_EMPLOYEE_WORKDAY to exclude inactive employees.

---

**Check 2 — Territories (INDUSTRY_CHECK_TERRITORIES)**
> Checks for bonus plan conflicts and that district/patch are configured per role type (managers need district; ICs need both).
> **Risk if NEEDS REVIEW:** Employee cannot access correct accounts in SFDC; consumption attribution broken.

```sql
SELECT
    EEID,
    D_EXISTING_EMPLOYEES AS name,
    CASE WHEN QUOTA_ROLE ILIKE '%Mgmt%' THEN 'Yes' ELSE 'No' END AS is_manager,
    CASE
        WHEN CORPORATE_BONUS_PLAN IS NOT NULL THEN 'NEEDS REVIEW'
        WHEN QUOTA_ROLE ILIKE '%Mgmt%' AND INDUSTRY_DISTRICT IS NULL THEN 'NEEDS REVIEW'
        WHEN QUOTA_ROLE NOT ILIKE '%Mgmt%' AND (INDUSTRY_DISTRICT IS NULL OR INDUSTRY_PATCH IS NULL) THEN 'NEEDS REVIEW'
        ELSE 'PASS'
    END AS check_result,
    CASE
        WHEN CORPORATE_BONUS_PLAN IS NOT NULL
            THEN 'Corporate bonus plan is populated (should be null)'
        WHEN QUOTA_ROLE ILIKE '%Mgmt%' AND INDUSTRY_DISTRICT IS NULL
            THEN 'Industry district is missing'
        WHEN QUOTA_ROLE NOT ILIKE '%Mgmt%' AND INDUSTRY_DISTRICT IS NULL AND INDUSTRY_PATCH IS NULL
            THEN 'Industry district and patch are both missing'
        WHEN QUOTA_ROLE NOT ILIKE '%Mgmt%' AND INDUSTRY_DISTRICT IS NULL
            THEN 'Industry district is missing'
        WHEN QUOTA_ROLE NOT ILIKE '%Mgmt%' AND INDUSTRY_PATCH IS NULL
            THEN 'Industry patch is missing'
        ELSE NULL
    END AS action_needed
FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
WHERE r.ACTIVE_RECORD = TRUE
  AND r.EEID = '<EEID>'  -- remove for whole-org
```

*Whole-org:* remove `AND r.EEID = '<EEID>'`.

---

**Check 3 — ETM Alignments (INDUSTRY_CHECK_ETM_ALIGNMENTS)**
> Validates that QUOTA_ROLE, HC_FUNCTION_COMP_START, PRIMARY_ETM_FUNCTION_COMP_START, and SECONDARY_ETM_FUNCTION_COMP_START all consistently indicate Architect or GTM/Principal. Also flags bonus plan conflicts.
> **Risk if NEEDS REVIEW:** Org hierarchy and territory routing will be incorrect; reporting and comp affected.

```sql
SELECT
    EEID,
    D_EXISTING_EMPLOYEES AS name,
    CASE
        WHEN CORPORATE_BONUS_PLAN IS NOT NULL
            THEN 'NEEDS REVIEW'
        WHEN NOT (
            (
                QUOTA_ROLE ILIKE '%Architect%'
                AND HC_FUNCTION_COMP_START ILIKE '%Architect%'
                AND PRIMARY_ETM_FUNCTION_COMP_START ILIKE '%Architect%'
                AND SECONDARY_ETM_FUNCTION_COMP_START ILIKE '%Architect%'
            )
            OR (
                (QUOTA_ROLE ILIKE '%GTM%' OR QUOTA_ROLE ILIKE '%Principal%')
                AND (HC_FUNCTION_COMP_START ILIKE '%GTM%' OR HC_FUNCTION_COMP_START ILIKE '%Principal%')
                AND (PRIMARY_ETM_FUNCTION_COMP_START ILIKE '%GTM%' OR PRIMARY_ETM_FUNCTION_COMP_START ILIKE '%Principal%')
                AND (SECONDARY_ETM_FUNCTION_COMP_START ILIKE '%GTM%' OR SECONDARY_ETM_FUNCTION_COMP_START ILIKE '%Principal%')
            )
        )
            THEN 'NEEDS REVIEW'
        ELSE 'PASS'
    END AS check_result,
    CASE
        WHEN CORPORATE_BONUS_PLAN IS NOT NULL
            THEN 'Corporate bonus plan is populated (should be null)'
        WHEN NOT (
            (
                QUOTA_ROLE ILIKE '%Architect%'
                AND HC_FUNCTION_COMP_START ILIKE '%Architect%'
                AND PRIMARY_ETM_FUNCTION_COMP_START ILIKE '%Architect%'
                AND SECONDARY_ETM_FUNCTION_COMP_START ILIKE '%Architect%'
            )
            OR (
                (QUOTA_ROLE ILIKE '%GTM%' OR QUOTA_ROLE ILIKE '%Principal%')
                AND (HC_FUNCTION_COMP_START ILIKE '%GTM%' OR HC_FUNCTION_COMP_START ILIKE '%Principal%')
                AND (PRIMARY_ETM_FUNCTION_COMP_START ILIKE '%GTM%' OR PRIMARY_ETM_FUNCTION_COMP_START ILIKE '%Principal%')
                AND (SECONDARY_ETM_FUNCTION_COMP_START ILIKE '%GTM%' OR SECONDARY_ETM_FUNCTION_COMP_START ILIKE '%Principal%')
            )
        )
            THEN 'Role fields do not consistently indicate Architect or GTM/Principal across HC_FUNCTION, PRIMARY_ETM_FUNCTION, SECONDARY_ETM_FUNCTION'
        ELSE NULL
    END AS action_needed
FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
WHERE r.ACTIVE_RECORD = TRUE
  AND r.EEID = '<EEID>'  -- remove for whole-org
```

*Whole-org:* remove `AND r.EEID = '<EEID>'`.

---

**Check 5 — Targets (INDUSTRY_CHECK_TARGETS)**
> Managers need primary target only. ICs need both primary and secondary. Corp bonus plan employees excluded.
> **Risk if NEEDS REVIEW:** Employee has no attainment target; compensation will be incorrect.

```sql
SELECT
    EEID,
    D_EXISTING_EMPLOYEES AS name,
    CASE WHEN QUOTA_ROLE ILIKE '%Mgmt%' THEN 'Yes' ELSE 'No' END AS is_manager,
    CASE
        WHEN QUOTA_ROLE ILIKE '%Mgmt%' AND CALC_PRIMARY_INCR_CONSUMPTION IS NULL
            THEN 'NEEDS REVIEW'
        WHEN QUOTA_ROLE NOT ILIKE '%Mgmt%' AND (CALC_PRIMARY_INCR_CONSUMPTION IS NULL OR CALC_SECONDARY_INCR_CONSUMPTION IS NULL)
            THEN 'NEEDS REVIEW'
        ELSE 'PASS'
    END AS check_result,
    CASE
        WHEN QUOTA_ROLE ILIKE '%Mgmt%' AND CALC_PRIMARY_INCR_CONSUMPTION IS NULL
            THEN 'Primary incremental consumption is missing'
        WHEN QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_PRIMARY_INCR_CONSUMPTION IS NULL AND CALC_SECONDARY_INCR_CONSUMPTION IS NULL
            THEN 'Primary and secondary incremental consumption are both missing'
        WHEN QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_PRIMARY_INCR_CONSUMPTION IS NULL
            THEN 'Primary incremental consumption is missing'
        WHEN QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_SECONDARY_INCR_CONSUMPTION IS NULL
            THEN 'Secondary incremental consumption is missing'
        ELSE NULL
    END AS action_needed
FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
WHERE r.ACTIVE_RECORD = TRUE
  -- AND r.EEID = '<EEID>'  -- uncomment for single-person
```

*Whole-org:* no EEID filter. *Single-person:* add `AND r.EEID = '<EEID>'`.

---

**Check 6 — Baselines (INDUSTRY_CHECK_BASELINES)**
> Per-month check: managers need primary baseline; ICs need both primary and secondary. Months before quota start are skipped.
> **Risk if NEEDS REVIEW:** Monthly quota basis for attainment calculation will be wrong.

```sql
WITH base AS (
    SELECT
        EEID,
        D_EXISTING_EMPLOYEES AS name,
        QUOTA_ROLE,
        CASE WHEN QUOTA_ROLE ILIKE '%Mgmt%' THEN 'Yes' ELSE 'No' END AS is_manager,
        TRY_TO_DATE(LEFT(QUOTA_START_DATE, 10), 'MM/DD/YYYY') AS quota_start,
        IFF(quota_start <= '2026-02-28' AND (CALC_FEB_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_FEB_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Feb', NULL) AS feb_issue,
        IFF(quota_start <= '2026-03-31' AND (CALC_MAR_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_MAR_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Mar', NULL) AS mar_issue,
        IFF(quota_start <= '2026-04-30' AND (CALC_APR_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_APR_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Apr', NULL) AS apr_issue,
        IFF(quota_start <= '2026-05-31' AND (CALC_MAY_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_MAY_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'May', NULL) AS may_issue,
        IFF(quota_start <= '2026-06-30' AND (CALC_JUN_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_JUN_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Jun', NULL) AS jun_issue,
        IFF(quota_start <= '2026-07-31' AND (CALC_JUL_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_JUL_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Jul', NULL) AS jul_issue,
        IFF(quota_start <= '2026-08-31' AND (CALC_AUG_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_AUG_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Aug', NULL) AS aug_issue,
        IFF(quota_start <= '2026-09-30' AND (CALC_SEP_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_SEP_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Sep', NULL) AS sep_issue,
        IFF(quota_start <= '2026-10-31' AND (CALC_OCT_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_OCT_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Oct', NULL) AS oct_issue,
        IFF(quota_start <= '2026-11-30' AND (CALC_NOV_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_NOV_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Nov', NULL) AS nov_issue,
        IFF(quota_start <= '2026-12-31' AND (CALC_DEC_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_DEC_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Dec', NULL) AS dec_issue,
        IFF(quota_start <= '2027-01-31' AND (CALC_JAN_PRIMARY_CONSUMPTION_BASELINE IS NULL OR (QUOTA_ROLE NOT ILIKE '%Mgmt%' AND CALC_JAN_SECONDARY_CONSUMPTION_BASELINE IS NULL)), 'Jan', NULL) AS jan_issue
    FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
    INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
        ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
    WHERE r.ACTIVE_RECORD = TRUE
      -- AND r.EEID = '<EEID>'  -- uncomment for single-person
),
rolled AS (
    SELECT
        EEID,
        name,
        is_manager,
        ARRAY_TO_STRING(
            ARRAY_COMPACT(ARRAY_CONSTRUCT(
                feb_issue, mar_issue, apr_issue, may_issue, jun_issue, jul_issue,
                aug_issue, sep_issue, oct_issue, nov_issue, dec_issue, jan_issue
            )), ', '
        ) AS issue_months
    FROM base
)
SELECT
    EEID,
    name,
    is_manager,
    IFF(issue_months = '', 'PASS', 'NEEDS REVIEW') AS check_result,
    IFF(issue_months = '', NULL, 'Missing baselines for: ' || issue_months) AS action_needed
FROM rolled
ORDER BY check_result DESC, name
```

> **Note:** Month-end dates are hardcoded for FY2026 (Feb 2026 – Jan 2027). Update these dates when the fiscal year rolls over. Managers-only check: secondary baselines skipped for `QUOTA_ROLE ILIKE '%Mgmt%'`.

---

**Check 7 — Metadata (INDUSTRY_CHECK_METADATA)**
> PASS = SPECIALIST_GROUP is populated. NEEDS REVIEW = SPECIALIST_GROUP is NULL.
> **Risk if NEEDS REVIEW:** Role type unidentified; Sigma access and SFDC reporting will be wrong.

```sql
SELECT
    a.EMPLOYEE_ID AS eeid,
    a.EMPLOYEE AS name,
    a.SPECIALIST_GROUP,
    CASE
        WHEN a.SPECIALIST_GROUP IS NULL THEN 'NEEDS REVIEW'
        ELSE 'PASS'
    END AS check_result,
    CASE
        WHEN a.SPECIALIST_GROUP IS NULL THEN 'Specialist group is missing — add to RAW_SPECIALIST_ATTRIBUTES_INYR in Pigment'
        ELSE NULL
    END AS action_needed
FROM IT.PIGMENT.RAW_SPECIALIST_ATTRIBUTES_INYR a
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON a.EMPLOYEE_ID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
  -- AND a.EMPLOYEE_ID = '<EEID>'  -- uncomment for single-person
ORDER BY check_result DESC, name
```

*Whole-org:* no EEID filter. The query returns all active specialists from RAW_SPECIALIST_ATTRIBUTES_INYR. For whole-org mode, scope to Industry employees by cross-referencing EEIDs from RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR.

---

#### Phase: Access

---

**Check 8 — Sigma Access (INDUSTRY_CHECK_SIGMA_ACCESS)**
> PASS = employee has an active Sigma account. Uses Workday email as the join key.
> **Important:** Use `INNER JOIN` to `D_EMPLOYEE_WORKDAY` (not LEFT JOIN) to avoid returning manager/former-employee EEIDs that appear in the Pigment roster but are not active Industry team members.
> **Risk if NEEDS REVIEW:** Employee cannot access Sigma dashboards.

```sql
WITH sigma_users AS (
    SELECT DISTINCT USER_EMAIL
    FROM SIGMA_COMPUTING_DATA_SHARE.AUDIT_LOGS.USERS
    WHERE USER_IS_ACTIVE = TRUE
)
SELECT
    r.EEID,
    r.D_EXISTING_EMPLOYEES AS name,
    e.EMPLOYEE_EMAIL,
    CASE
        WHEN s.USER_EMAIL IS NULL THEN 'NEEDS REVIEW'
        ELSE 'PASS'
    END AS check_result,
    CASE
        WHEN s.USER_EMAIL IS NULL THEN 'Not found as an active user in Sigma — file a Lift ticket to provision access'
        ELSE NULL
    END AS action_needed
FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
LEFT JOIN sigma_users s ON e.EMPLOYEE_EMAIL = s.USER_EMAIL
WHERE r.ACTIVE_RECORD = TRUE
  -- AND r.EEID = '<EEID>'  -- uncomment for single-person
ORDER BY check_result DESC, name
```

> **Why INNER JOIN:** The Pigment roster sometimes contains manager/ex-employee EEIDs (e.g. `INSEAT_MANAGER_COMP_START` values) that have `ACTIVE_RECORD = TRUE` but no active Workday record. Using `LEFT JOIN employee_emails` returns those rows as false NEEDS REVIEW — the INNER JOIN to D_EMPLOYEE_WORKDAY with `IS_EMPLOYEE_ACTIVE = TRUE` scopes the result to the correct 70 active employees.

*Single-person:* add `AND r.EEID = '<EEID>'`. *Whole-org:* no EEID filter needed.

---

**Check 9 — Slack: CX-Specialists (INDUSTRY_CHECK_SLACK_CX_SPECIALISTS)**
> **Industry Architects only** (QUOTA_ROLE `Industry Architect` or `Industry Architect Mgmt`). Skip for all other roles.
> PASS = architect's email is in the `#cx-specialists` member list.
> NEEDS REVIEW = architect email not found in channel.
> **Risk if NEEDS REVIEW:** Architect misses team communications and tool release announcements.

**How to run (MCP — started in Round 1):**

1. Collect all pages of `#cx-specialists` members (channel ID: `C0A7D6EB79Q`) using `mcp_slack-snow_slack_list_channel_members` with `limit=200`. Each page returns member emails. Paginate via cursor until `No more members available`. Build a lowercase email set.

2. For each Architect (use `EMPLOYEE_EMAIL` from Step 1c / single-person email from Step 1):
   - Lowercase the email and check if it exists in the collected email set
   - `PASS` if present; `NEEDS REVIEW` with action `Add to #cx-specialists — verify Slack account provisioned first` if absent

> No `mcp_slack-snow_slack_search_users` calls needed — the member list already includes emails. This avoids one API call per person.

> Channel IDs (hardcoded): `#cx-specialists` = `C0A7D6EB79Q`; `#cx-specialists-managers` = `C0A7S4KJELR`

> **Known issue:** Users with no Slack account (e.g. very new hires) will show NEEDS REVIEW. The action covers both cases: verify account exists, then add to channel.

---

**Check 10 — Slack: CX-Specialists-Managers (INDUSTRY_CHECK_SLACK_CX_MGRS)**
> **Industry Architect Mgmt only** (QUOTA_ROLE = `Industry Architect Mgmt`). Skip for all other roles.
> PASS = architect manager's email is in the `#cx-specialists-managers` member list.
> NEEDS REVIEW = email not found in channel.
> **Risk if NEEDS REVIEW:** Manager misses cross-team operations discussions.

**How to run (MCP — started in Round 1):**

1. Collect all pages of `#cx-specialists-managers` members (channel ID: `C0A7S4KJELR`) using `mcp_slack-snow_slack_list_channel_members`. Build a lowercase email set.

2. For each `Industry Architect Mgmt` person: lowercase their email and check against the set.
   - `PASS` if present; `NEEDS REVIEW` with action `Add to #cx-specialists-managers — verify Slack account provisioned first` if absent

> Same email-set approach as Check 9 — no separate `search_users` calls needed.

---

#### Phase: Systems

---

**Check 11 — CiQ** ⚠️ *Manual Check*
> Cannot be automated. CiQ is the attainment payout platform. Verify the employee has been provisioned in CiQ directly.
> Note as `MANUAL CHECK — Verify in CiQ` in the summary.

---

**Check 12 — Attainment Dashboard (INDUSTRY_CHECK_ATTAINMENT_REPORTING)**
> PASS = employee has records in the attainment reporting table. NEEDS REVIEW = no rows found.
> Uses `TO_VARCHAR(EEID)` to match the attainment table's VARCHAR WORKDAY_EMPLOYEE_ID key.
> **Risk if NEEDS REVIEW:** Employee does not appear in attainment dashboards — compensation visibility broken.

```sql
WITH attainment_ids AS (
    SELECT DISTINCT WORKDAY_EMPLOYEE_ID
    FROM SNOW_CERTIFIED.SALES_PERFORMANCE.AGG_INDUSTRY_USER_CONSUMPTION_ATTAINMENT
)
SELECT
    r.EEID,
    r.D_EXISTING_EMPLOYEES AS name,
    CASE
        WHEN a.WORKDAY_EMPLOYEE_ID IS NULL THEN 'NEEDS REVIEW'
        ELSE 'PASS'
    END AS check_result,
    CASE
        WHEN a.WORKDAY_EMPLOYEE_ID IS NULL THEN 'EEID not found in attainment table — confirm C7 metadata and C6 baselines are configured first, then recheck'
        ELSE NULL
    END AS action_needed
FROM IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
LEFT JOIN attainment_ids a ON TO_VARCHAR(r.EEID) = a.WORKDAY_EMPLOYEE_ID
WHERE r.ACTIVE_RECORD = TRUE
  -- AND r.EEID = '<EEID>'  -- uncomment for single-person
ORDER BY check_result DESC, name
```

---

#### Phase: Transfers

---

**Check 13 — Transfers (INDUSTRY_CHECK_FOR_TRANSFERS)**
> Lists all Industry employees with a transfer in or out in the last 15 days. Informational — not a PASS/FAIL check. Run this alongside onboarding checks to surface recently transferred employees who may need onboarding re-validation.
> **Scope:** All Industry employees (not scoped to new hires only).

```sql
WITH latest_pigment_roster AS (
    SELECT EEID, JOB_TITLE, HC_FUNCTION, TRANSFER_IN_DATE, TRANSFER_OUT_DATE
    FROM IT.PIGMENT.PIGMENT_ROSTER
    WHERE DS_DATE = (SELECT MAX(DS_DATE) FROM IT.PIGMENT.PIGMENT_ROSTER)
      AND (
          TRY_TO_DATE(LEFT(TRANSFER_IN_DATE, 10), 'MM/DD/YYYY') >= CURRENT_DATE - 15
          OR TRY_TO_DATE(LEFT(TRANSFER_OUT_DATE, 10), 'MM/DD/YYYY') >= CURRENT_DATE - 15
      )
)
SELECT
    r.EEID,
    r.D_EXISTING_EMPLOYEES AS name,
    r.QUOTA_ROLE,
    r.INSEAT_MANAGER_COMP_START AS manager,
    r.ACTIVE_RECORD,
    r.FILTER_STATUS,
    r.OPS_READY_DATE_PLANNING_ONLY,
    p.TRANSFER_IN_DATE,
    p.TRANSFER_OUT_DATE,
    CASE
        WHEN p.TRANSFER_IN_DATE IS NOT NULL AND p.TRANSFER_OUT_DATE IS NOT NULL THEN 'Transfer In/Out'
        WHEN p.TRANSFER_IN_DATE IS NOT NULL THEN 'Transfer In'
        WHEN p.TRANSFER_OUT_DATE IS NOT NULL THEN 'Transfer Out'
    END AS transfer_type
FROM latest_pigment_roster p
INNER JOIN IT.PIGMENT.RAW_INDUSTRY_ROSTER_QUOTA_SUMMARY_INYR r ON p.EEID = r.EEID
INNER JOIN SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY e
    ON r.EEID = e.WORKDAY_EMPLOYEE_ID AND e.IS_EMPLOYEE_ACTIVE = TRUE
ORDER BY COALESCE(TRY_TO_DATE(LEFT(p.TRANSFER_IN_DATE, 10), 'MM/DD/YYYY'),
                  TRY_TO_DATE(LEFT(p.TRANSFER_OUT_DATE, 10), 'MM/DD/YYYY')) DESC, name
```

> If any transfers are found, present the count and list the employees with their transfer type and date. Note: transferred-in employees should have all onboarding checks re-validated (re-run C1–C12 for them). Transferred-out employees should be confirmed inactive in Pigment.

---

### Step 3c: EOM Close Checks

*(EOM Close workflow only — run when user selects "EOM Close" in Step 1)*

Run all three checks in parallel. No scope confirmation needed.

---

#### EOM Check 1 — Consumption No Industry (INDUSTRY_CHECK_CONSUMPTION_NO_INDUSTRY)
> Finds accounts that have consumption revenue but no industry assigned in Salesforce.
> **Risk:** Account revenue is unattributed to an industry — affects reporting and compensation.
> **Important:** Always filter to `FORECAST_RUN_DATE = MAX(FORECAST_RUN_DATE)` and the current `FISCAL_YEAR`. The table stores one row per account per forecast run per calendar date — summing without this filter multiplies revenue ~188x across all runs.

```sql
WITH latest_run AS (
    SELECT MAX(FORECAST_RUN_DATE) AS max_run_date,
           MAX(FISCAL_YEAR) AS current_fy
    FROM FINANCE.CUSTOMER.PRODUCT_REVENUE_FORECAST_SALES_PLANNING
    WHERE FORECAST_RUN_DATE = (SELECT MAX(FORECAST_RUN_DATE) FROM FINANCE.CUSTOMER.PRODUCT_REVENUE_FORECAST_SALES_PLANNING)
)
SELECT
    prf.salesforce_account_name AS name,
    CASE
        WHEN SUM(prf.total_product_revenue) > 0 AND MAX(acct.salesforce_account_industry) IS NULL
            THEN 'NEEDS REVIEW'
        ELSE 'PASS'
    END AS check_result,
    CASE
        WHEN SUM(prf.total_product_revenue) > 0 AND MAX(acct.salesforce_account_industry) IS NULL
            THEN 'Account has consumption ($' || TO_VARCHAR(ROUND(SUM(prf.total_product_revenue)/1000000, 1)) || 'M) but no industry assigned'
        ELSE NULL
    END AS review_notes
FROM FINANCE.CUSTOMER.PRODUCT_REVENUE_FORECAST_SALES_PLANNING prf
CROSS JOIN latest_run
LEFT JOIN SNOW_CERTIFIED.SALESFORCE_ACCOUNT.DD_SALESFORCE_ACCOUNT acct
    ON prf.salesforce_account_id = acct.salesforce_account_id
WHERE prf.FORECAST_RUN_DATE = latest_run.max_run_date
  AND prf.FISCAL_YEAR = latest_run.current_fy
GROUP BY prf.salesforce_account_name
HAVING SUM(prf.total_product_revenue) > 0
ORDER BY check_result DESC, SUM(prf.total_product_revenue) DESC
```

---

#### EOM Check 2 — Account Reclassification (INDUSTRY_CHECK_FOR_ACCT_RECLASSIFICATION)
> Returns all accounts where industry or sub-industry changed in the last month.
> **Risk:** Industry attribution has shifted — may affect quota, compensation, or territory routing.

```sql
WITH changes AS (
  SELECT
    curr.ACCOUNT_ID,
    curr.ACCOUNT_NAME,
    curr.DS AS change_date,
    prev.INDUSTRY AS old_industry,
    curr.INDUSTRY AS new_industry,
    prev.SUB_INDUSTRY AS old_sub_industry,
    curr.SUB_INDUSTRY AS new_sub_industry
  FROM SALES.METRO_STAGING.ACCOUNTS_DAILY curr
  JOIN SALES.METRO_STAGING.ACCOUNTS_DAILY prev
    ON curr.ACCOUNT_ID = prev.ACCOUNT_ID
    AND curr.DS = DATEADD(day, 1, prev.DS)
  WHERE curr.DS >= DATEADD(month, -1, CURRENT_DATE)
    AND (curr.INDUSTRY != prev.INDUSTRY
     OR curr.SUB_INDUSTRY != prev.SUB_INDUSTRY)
)
SELECT * FROM changes
ORDER BY change_date DESC
```

---

#### EOM Check 3 — Opportunity Reclassification (INDUSTRY_CHECK_FOR_OPP_RECLASSIFICATION)
> Returns all opportunities where industry or sub-industry changed in the last month.
> **Risk:** Opportunity attribution has shifted — may affect forecast accuracy and comp calculation.

```sql
WITH changes AS (
  SELECT
    curr.OPP_ID,
    curr.OPP_NAME,
    curr.ACCOUNT_ID,
    curr.ACCOUNT_NAME,
    curr.DS AS change_date,
    prev.INDUSTRY AS old_industry,
    curr.INDUSTRY AS new_industry,
    prev.SUB_INDUSTRY AS old_sub_industry,
    curr.SUB_INDUSTRY AS new_sub_industry,
    curr.STAGE_NAME,
    curr.OPPORTUNITY_ACV,
    curr.CLOSE_DATE
  FROM SALES.METRO_STAGING.OPPORTUNITIES_DAILY curr
  JOIN SALES.METRO_STAGING.OPPORTUNITIES_DAILY prev
    ON curr.OPP_ID = prev.OPP_ID
    AND curr.DS = DATEADD(day, 1, prev.DS)
  WHERE curr.DS >= DATEADD(month, -1, CURRENT_DATE)
    AND (
      (curr.INDUSTRY != prev.INDUSTRY OR (curr.INDUSTRY IS NULL) != (prev.INDUSTRY IS NULL))
      OR (curr.SUB_INDUSTRY != prev.SUB_INDUSTRY OR (curr.SUB_INDUSTRY IS NULL) != (prev.SUB_INDUSTRY IS NULL))
    )
)
SELECT * FROM changes
ORDER BY change_date DESC
```

> Present counts and a list of changes. For NEEDS REVIEW accounts (no industry), escalate to Field Ops. For reclassified accounts/opps, note the old vs new industry so the team can verify the change was intentional.

---

### Step 4: Present Phase-Grouped Summary

After running all checks, present results grouped by phase. For every NEEDS REVIEW or MANUAL CHECK item, include the action needed.

**Single-person summary format:**

```
=== ONBOARDING CHECK RESULTS: [Employee Name] (EEID: XXXXX) ===

PHASE 1: IN-YEAR PLANNING
  ✅ Check 1 — Employee Record        PASS
  ✅ Check 2 — Territories            PASS
  ❌ Check 3 — ETM Alignments         NEEDS REVIEW  → Role fields inconsistent across HC/ETM functions
  ✅ Check 5 — Targets                PASS
  ✅ Check 6 — Baselines              PASS
  ❌ Check 7 — Metadata               NEEDS REVIEW  → Specialist group is missing

PHASE 2: ACCESS
  ✅ Check 8 — Sigma Access           PASS
  ✅ Check 9 — Slack: CX-Specialists PASS (or ❌ NEEDS REVIEW → Add to #cx-specialists)
  ✅ Check 10 — Slack: Managers      PASS (or ❌ NEEDS REVIEW → Add to #cx-specialists-managers) [Architects only]

PHASE 3: SYSTEMS
  ⚠️  Check 11 — CiQ                  MANUAL CHECK  → Verify in CiQ
  ✅ Check 12 — Attainment Dashboard  PASS

TRANSFERS (last 15 days): 0 transfers found

AUTOMATED: 2 NEEDS REVIEW, 7 PASS
MANUAL CHECKS: 1 check requires human verification
```

**Whole-org summary format:**

Present a summary table of check results across all active employees:

| Check | PASS | NEEDS REVIEW | MANUAL CHECK |
|-------|------|--------------|---------------|
| Employee Record | N | N | — |
| Territories | N | N | — |
| ETM Alignments | N | N | — |
| Targets | N | N | — |
| Baselines | N | N | — |
| Metadata | N | N | — |
| Sigma Access | N | N | — |
| Slack: CX-Specialists | N | N | — (Architects only) |
| Slack: CX-Mgrs | N | N | — (Architect Mgmt only) |
| CiQ | — | — | ⚠️ All |
| Attainment Dashboard | N | N | — |

Then list each employee with NEEDS REVIEW on any check, with the specific checks and actions needed.

---

### Step 5: Write Audit Files (CSV + HTML Report)

After presenting the summary, write two files: a timestamped CSV audit log and an HTML report with KPI tiles.

> **CRITICAL — Script Execution Method:**
> The Python scripts below are large. **Never use a bash heredoc** (`cat << 'EOF'`) to write them — heredocs get silently truncated and the script will fail or produce incomplete output.
> **Always use the `write` tool** to write the complete script to `/tmp/gen_onboarding_report.py`, then execute with `python3 /tmp/gen_onboarding_report.py`.
> Workflow:
> 1. `write` tool → `/tmp/gen_onboarding_report.py` (full script content)
> 2. `bash` tool → `python3 /tmp/gen_onboarding_report.py`

---

#### Step 5a: CSV Audit Log

```python
import csv, os
from datetime import datetime

SKILL_DIR = os.path.expanduser('~/.snowflake/cortex/skills/industry-onboarding-checks')
AUDITS_DIR = os.path.join(SKILL_DIR, 'audits')
os.makedirs(AUDITS_DIR, exist_ok=True)

timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
filename = os.path.join(AUDITS_DIR, f'audit_{timestamp}.csv')

rows = [
    {
        'run_at': timestamp,
        'eeid': <eeid>,
        'name': <name>,
        'manager': <manager>,
        'check_1_employee_record': <result>,
        'check_2_territories': <result>,
        'check_3_etm_alignments': <result>,
        'check_4_industry_alignments': 'MANUAL CHECK',
        'check_5_targets': <result>,
        'check_6_baselines': <result>,
        'check_7_metadata': <result>,
        'check_8_sigma_access': <result>,  # PASS/NEEDS REVIEW for both single and whole-org
        'check_9_slack_cx_specialists': <result>,  # PASS/NEEDS REVIEW (Architects only); 'N/A' for GTM roles
        'check_10_slack_cx_managers': <result>,    # PASS/NEEDS REVIEW (Architect Mgmt only); 'N/A' for all others
        'check_11_ciq': 'MANUAL CHECK',
        'check_12_attainment_dashboard': <result>,
        'overall_automated': 'PASS' if all automated checks pass else 'NEEDS REVIEW',
        'action_needed': <comma-joined list of action_needed strings for any NEEDS REVIEW>
    }
    # one row per EEID
]

with open(filename, 'w', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=rows[0].keys())
    writer.writeheader()
    writer.writerows(rows)

print(f'CSV saved to: {filename}')
```

---

#### Step 5b: HTML KPI Report

Generate an HTML file with the following layout:

**Row 1 — 3 top-level summary tiles:**
- **Total Active** — total employees checked
- **Needs Review** — employees with ≥1 actionable automated issue (excluding C7 org-wide metadata gap)
- **Passing All Automated** — employees with no actionable issues (C4/C11 manual and C7 known gap excluded)

**Row 2 — Risk profile (all employees bucketed by highest risk):**
- **High Priority** (red) — C1, C5, C6, or C12 failing: compensation or attainment at risk
- **Medium Priority** (amber) — C8/C9/C10 failing only: access issues, no comp impact
- **Low / Monitor** (gray) — C7 only: known org-wide metadata gap, no compensation or access impact

Below the KPIs: **Summary table** (check-by-check NEEDS REVIEW / Passed / Manual Check) and **Action Items** detail sections for each failing check.

Write the Python script to disk using the **`write` tool** (NOT a bash heredoc) and then execute it:

```python
import os
from datetime import datetime

SKILL_DIR = os.path.expanduser('~/.snowflake/cortex/skills/industry-onboarding-checks')
AUDITS_DIR = os.path.join(SKILL_DIR, 'audits')
os.makedirs(AUDITS_DIR, exist_ok=True)

timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
run_date = datetime.now().strftime('%B %d, %Y %H:%M')
html_file = os.path.join(AUDITS_DIR, f'report_{timestamp}.html')

# Build summary table rows from check results
def summary_row(label, nr, passed, is_manual, risk=None, is_last=False, action_label=None, action_url=None):
    sep = '' if is_last else 'border-bottom:1px solid #f1f5f9;'
    dim = 'opacity:.7;' if (not is_manual and nr == 0) else ''
    bg = 'background:#fafafa;' if is_manual else ''
    risk_badges = {
        'high':   '<span style="font-size:11px;font-weight:700;padding:2px 8px;border-radius:99px;background:#fee2e2;color:#dc2626;">High</span>',
        'medium': '<span style="font-size:11px;font-weight:700;padding:2px 8px;border-radius:99px;background:#fef9c3;color:#d97706;">Medium</span>',
        'low':    '<span style="font-size:11px;font-weight:700;padding:2px 8px;border-radius:99px;background:#f1f5f9;color:#6b7280;">Low</span>',
    }
    risk_cell = f'<td style="padding:9px 16px;text-align:center;">{risk_badges[risk]}</td>' if risk else '<td style="padding:9px 16px;text-align:center;color:#94a3b8;">\u2014</td>'
    if is_manual:
        nr_cell = '<td style="padding:9px 16px;text-align:center;color:#94a3b8;">\u2014</td>'
        pass_cell = '<td style="padding:9px 16px;text-align:center;color:#94a3b8;">\u2014</td>'
        manual_cell = '<td style="padding:9px 16px;text-align:center;color:#6b7280;font-weight:600;">\u26a0\ufe0f All</td>'
    else:
        nr_color = '#9ca3af' if nr == 0 else '#dc2626'
        nr_text = f'0 \u2705' if nr == 0 else str(nr)
        nr_weight = 'font-weight:700;' if nr > 0 else ''
        nr_cell = f'<td style="padding:9px 16px;text-align:center;{nr_weight}color:{nr_color};">{nr_text}</td>'
        pass_cell = f'<td style="padding:9px 16px;text-align:center;color:#475569;">{passed}</td>'
        manual_cell = '<td style="padding:9px 16px;text-align:center;color:#94a3b8;">\u2014</td>'
    # Take Action cell: show link/text when nr>0 or manual; dash when passing
    show_action = (is_manual or nr > 0)
    if show_action and action_label and action_url:
        action_cell = f'<td style="padding:9px 16px;"><a href="{action_url}" target="_blank" style="font-size:12px;color:#2563eb;font-weight:500;white-space:nowrap;">&rarr; {action_label}</a></td>'
    elif show_action and action_label:
        action_cell = f'<td style="padding:9px 16px;font-size:12px;font-weight:500;color:#475569;white-space:nowrap;">{action_label}</td>'
    else:
        action_cell = '<td style="padding:9px 16px;text-align:center;color:#94a3b8;">\u2014</td>'
    return f'<tr style="{sep}{dim}{bg}"><td style="padding:9px 16px;color:#1e293b;font-weight:500;">{label}</td>{risk_cell}{nr_cell}{pass_cell}{manual_cell}{action_cell}</tr>'

PIGMENT_206 = 'https://pigment.app/w/snowflake/application/9746e078-5805-4dba-b0df-3d4adcb95601/boards/e579b1e3-b2bd-44d7-b864-70c46713b691'
PIGMENT_205 = 'https://pigment.app/w/snowflake/application/7ab9ab3f-584b-4a23-95b1-813cf708b2c1/boards/6eb0b362-57c3-49bf-99fd-59c693b750bc'
LIFT_URL    = 'https://lift.snowflake.com/lift?id=esc_sc_cat_item&table=sc_cat_item&sys_id=62819c911b1f1a505f6111f3b24bcb37&searchTerm=sigma'

summary_rows_html = (
    summary_row('C1 — Employee Record',       <nr_c1>,  <pass_c1>,  False, risk='high',   action_label='Contact In-Year Planning Team') +
    summary_row('C2 — Territories',           <nr_c2>,  <pass_c2>,  False, risk='high',   action_label='Pigment 2.06', action_url=PIGMENT_206) +
    summary_row('C3 — ETM Alignments',        <nr_c3>,  <pass_c3>,  False, risk='high',   action_label='Pigment 2.06', action_url=PIGMENT_206) +
    summary_row('C5 — Targets',               <nr_c5>,  <pass_c5>,  False, risk='high',   action_label='Pigment 2.06', action_url=PIGMENT_206) +
    summary_row('C6 — Baselines',             <nr_c6>,  <pass_c6>,  False, risk='high',   action_label='Pigment 2.06', action_url=PIGMENT_206) +
    summary_row('C7 — Metadata',              <nr_c7>,  <pass_c7>,  False, risk='low',    action_label='Pigment 2.05', action_url=PIGMENT_205) +
    summary_row('C8 — Sigma Access',          <nr_c8>,  <pass_c8>,  False, risk='medium', action_label='Lift Request', action_url=LIFT_URL) +
    summary_row('C9 — Slack: CX-Specialists', <nr_c9>,  <pass_c9>,  False, risk='medium', action_label='Contact Field Ops') +
    summary_row('C10 — Slack: CX-Mgrs',       <nr_c10>, <pass_c10>, False, risk='medium', action_label='Contact Field Ops') +
    summary_row('C11 — CiQ',                  0,        0,           True,               action_label='Contact In-Year Planning Team') +
    summary_row('C12 — Attainment Dashboard', <nr_c12>, <pass_c12>, False, risk='high',   action_label='Contact DAA', is_last=True)
)
# detail_checks: list of dicts with keys: label, nr, rows (list of row dicts), instructions, link_label, link_url
# Each row dict: {'cells': [list of cell values], 'headers': [list of header names]} on first call only
def detail_section(label, nr, headers, rows, instructions, link_label=None, link_url=None, badge_bg='#fee2e2', badge_fg='#dc2626'):
    if nr == 0:
        return ''
    header_cells = ''.join(f'<th style="padding:9px 14px;text-align:left;font-weight:600;color:#64748b;font-size:11px;text-transform:uppercase;letter-spacing:.4px;border-bottom:1px solid #e2e8f0;">{h}</th>' for h in headers)
    row_html = ''
    for i, r in enumerate(rows):
        br = 'border-bottom:1px solid #f1f5f9;' if i < len(rows)-1 else ''
        cells = ''.join(f'<td style="padding:9px 14px;color:#475569;">{c}</td>' for c in r)
        row_html += f'<tr style="{br}">{cells}</tr>'
    link_html = (
        f'<a href="{link_url}" target="_blank" style="font-size:12px;color:#2563eb;font-weight:500;white-space:nowrap;">&rarr; {link_label}</a>'
        if link_label and link_url else
        f'<span style="font-size:12px;font-weight:500;color:#475569;white-space:nowrap;">{link_label}</span>'
        if link_label else ''
    )
    return f'''
    <div style="margin-bottom:28px;">
      <div style="display:flex;align-items:center;gap:10px;margin-bottom:10px;">
        <span style="font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;padding:2px 8px;border-radius:99px;background:{badge_bg};color:{badge_fg};">{nr} Needs Review</span>
        <span style="font-size:14px;font-weight:700;color:#0f172a;">{label}</span>
      </div>
      <div style="background:#fff;border-radius:10px;border:1px solid #e2e8f0;overflow:hidden;">
        <table style="width:100%;border-collapse:collapse;font-size:13px;">
          <thead><tr style="background:#f8fafc;">{header_cells}</tr></thead>
          <tbody>{row_html}</tbody>
        </table>
        <div style="padding:12px 16px;background:#f8fafc;border-top:1px solid #e2e8f0;font-size:12px;color:#64748b;display:flex;justify-content:space-between;align-items:center;">
          <span>{instructions}</span>{link_html}
        </div>
      </div>
    </div>'''

# Populate detail_sections_html from actual check results
detail_sections_html = ''

# C1 — Employee Record (Remediation Location: Contact In-Year Planning Team, no URL)
detail_sections_html += detail_section(
    'C1 — Employee Record', <nr_c1>,
    ['Employee', 'Role', 'Manager', 'Action'],
    <needs_review_rows_c1>,   # e.g. [['Jane Doe (12345)', 'Industry GTM', 'Rinesh Patel', 'Contact Field Ops']]
    'Ops Ready status required before ICM sync.',
    'Contact In-Year Planning Team'  # no URL — renders as plain text
)

# C5 — Targets
detail_sections_html += detail_section(
    'C5 — Targets', <nr_c5>,
    ['Employee', 'Role', 'Missing'],
    <needs_review_rows_c5>,   # e.g. [['Rinesh Patel (5368)', 'Industry GTM Mgmt', 'Secondary INCR target']]
    'Enter CALC_SECONDARY_INCR_CONSUMPTION in Pigment. Baselines (C6) will also need updating after targets are set.',
    'Pigment 2.06', 'https://pigment.app/w/snowflake/application/9746e078-5805-4dba-b0df-3d4adcb95601/boards/e579b1e3-b2bd-44d7-b864-70c46713b691'
)

# C6 — Baselines
detail_sections_html += detail_section(
    'C6 — Baselines', <nr_c6>,
    ['Employee', 'Months Affected'],
    <needs_review_rows_c6>,
    'Resolve C5 first. Baselines auto-populate once targets are set. If still failing, manually enter monthly baselines in Pigment.',
    'Pigment 2.06', 'https://pigment.app/w/snowflake/application/9746e078-5805-4dba-b0df-3d4adcb95601/boards/e579b1e3-b2bd-44d7-b864-70c46713b691'
)

# C7 — Metadata (Low risk badge)
detail_sections_html += detail_section(
    'C7 — Metadata', <nr_c7>,
    ['Employee', 'Role', 'Action'],
    <needs_review_rows_c7>,
    'Add SPECIALIST_GROUP to Pigment 2.05 OPS HC Planning (Specialist Overview). Only a handful of employees typically have this missing.',
    'Pigment 2.05', 'https://pigment.app/w/snowflake/application/7ab9ab3f-584b-4a23-95b1-813cf708b2c1/boards/6eb0b362-57c3-49bf-99fd-59c693b750bc',
    badge_bg='#f1f5f9', badge_fg='#6b7280'
)

# C8 — Sigma Access (Medium risk badge)
detail_sections_html += detail_section(
    'C8 — Sigma Access', <nr_c8>,
    ['Employee', 'Email', 'Action'],
    <needs_review_rows_c8>,
    'Employee not found as an active Sigma user. File a Lift ticket to provision access.',
    'Lift Request', 'https://lift.snowflake.com/lift?id=esc_sc_cat_item&table=sc_cat_item&sys_id=62819c911b1f1a505f6111f3b24bcb37&searchTerm=sigma',
    badge_bg='#fef9c3', badge_fg='#d97706'
)

# C12 — Attainment Dashboard (Remediation Location: Contact DAA, no URL)
detail_sections_html += detail_section(
    'C12 — Attainment Dashboard', <nr_c12>,
    ['Employee', 'Role', 'Manager', 'Resolution Order'],
    <needs_review_rows_c12>,
    '⚡ Fix C1 (Ops Ready) and C7 (Metadata) first; then verify C5/C6 targets and baselines are populated. Attainment data refreshes overnight. For persistent misses, escalate to RevOps.',
    'Contact DAA'  # no URL — renders as plain text
)

# ── KPI computation ──────────────────────────────────────────────────────
# Populate these from actual check results before building the HTML.
# Per-employee dicts: {'name': str, 'c1': 'PASS'|'NEEDS REVIEW', 'c5': ..., ...}
# employee_results = list of dicts, one per active employee
# Each dict has keys: name, eeid, role, c1, c2, c3, c5, c6, c7, c8, c9, c10, c12
# (C4=Industry Alignments removed; C11=CiQ is manual; C13=Transfers is informational)

COMP_CHECKS = {'c1', 'c5', 'c6', 'c12'}    # failing any = High risk (Employee Record, Targets, Baselines, Attainment)
ACCESS_CHECKS = {'c8', 'c9', 'c10'}         # failing any (but not comp) = Medium risk (Sigma, Slack)

high_employees, medium_employees, low_count = [], [], 0
for e in employee_results:
    comp_fail   = any(str(e.get(c, '')).upper() == 'NEEDS REVIEW' for c in COMP_CHECKS)
    access_fail = any(str(e.get(c, '')).upper() == 'NEEDS REVIEW' for c in ACCESS_CHECKS)
    if comp_fail:
        reasons = [c.upper() for c in COMP_CHECKS if str(e.get(c, '')).upper() == 'NEEDS REVIEW']
        high_employees.append(f"{e['name'].split(' (')[0]} ({', '.join(reasons)})")
    elif access_fail:
        medium_employees.append(e['name'].split(' (')[0])
    else:
        low_count += 1  # only C7 metadata gap or all checks passing

total_employees = <total_active_employees>
scope_label = <'Whole Org' or employee_name>
high_count   = len(high_employees)
medium_count = len(medium_employees)
needs_review_count = high_count + medium_count  # actionable issues (C7 metadata gap excluded — low risk)
passing_count      = low_count                  # only C7 known gap — no actionable issues

# ── Transfer Alert HTML ────────────────────────────────────────────────────
# Build from Check 13 results. Set to '' if no transfers found.
# transfers_list = [{'name': str, 'transfer_type': str, 'date': str, 'manager': str}, ...]
if transfers_list:
    items = ''.join(
        f'<div style="font-size:12px;color:#d97706;margin-top:2px;">{t["name"]} \u2014 {t["transfer_type"]} {t["date"]} &middot; Manager: {t["manager"]} &middot; Re-validate C1\u2013C12</div>'
        for t in transfers_list
    )
    transfers_html = f'''
  <div style="max-width:1160px;margin:20px auto 0;">
    <div style="background:#fffbeb;border:1px solid #fcd34d;border-radius:10px;padding:14px 18px;display:flex;align-items:flex-start;gap:12px;">
      <span style="font-size:20px;line-height:1;">&#128260;</span>
      <div>
        <div style="font-size:13px;font-weight:700;color:#92400e;">{len(transfers_list)} Transfer{"s" if len(transfers_list) > 1 else ""} in Last 15 Days</div>
        {items}
      </div>
    </div>
  </div>'''
else:
    transfers_html = ''

high_names_html   = '<ul style="margin:6px 0 0;padding-left:16px;font-size:10px;opacity:.85;">' + ''.join(f'<li>{n}</li>' for n in high_employees) + '</ul>' if high_employees else ''
medium_names_html = '<ul style="margin:6px 0 0;padding-left:16px;font-size:10px;opacity:.85;">' + ''.join(f'<li>{n}</li>' for n in medium_employees) + '</ul>' if medium_employees else ''

# ── Row 1: 3 top-level summary tiles ─────────────────────────────────────
row1_html = f'''
  <div style="display:grid;grid-template-columns:repeat(3,1fr);gap:14px;max-width:1160px;margin:0 auto;">
    <div style="background:#fff;border-radius:12px;padding:22px 20px;box-shadow:0 1px 6px rgba(0,0,0,.1);display:flex;flex-direction:column;gap:4px;">
      <div style="font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#64748b;">Total Active</div>
      <div style="font-size:52px;font-weight:900;line-height:1;color:#0f172a;margin:6px 0 4px;">{total_employees}</div>
      <div style="font-size:13px;font-weight:500;color:#475569;">Industry Principals &amp; Architects</div>
    </div>
    <div style="background:#fff;border-radius:12px;padding:22px 20px;box-shadow:0 1px 6px rgba(0,0,0,.1);display:flex;flex-direction:column;gap:4px;">
      <div style="font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#64748b;">Needs Review</div>
      <div style="font-size:52px;font-weight:900;line-height:1;color:{'#dc2626' if needs_review_count > 0 else '#9ca3af'};margin:6px 0 4px;">{needs_review_count}</div>
      <div style="font-size:13px;font-weight:500;color:#475569;">Employees with &ge;1 actionable issue</div>
      <div style="font-size:10px;opacity:.7;margin-top:2px;">Excludes C7 metadata gap (low risk)</div>
    </div>
    <div style="background:#fff;border-radius:12px;padding:22px 20px;box-shadow:0 1px 6px rgba(0,0,0,.1);display:flex;flex-direction:column;gap:4px;">
      <div style="font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#64748b;">Passing All Automated</div>
      <div style="font-size:52px;font-weight:900;line-height:1;color:#16a34a;margin:6px 0 4px;">{passing_count}</div>
      <div style="font-size:13px;font-weight:500;color:#475569;">No actionable issues detected</div>
      <div style="font-size:10px;opacity:.7;margin-top:2px;">Excludes C4, C11 (manual) and C7 metadata gap</div>
    </div>
  </div>'''

# ── Row 2: Risk profile tiles ─────────────────────────────────────────────
row2_html = f'''
  <div style="max-width:1160px;margin:10px auto 0;">
    <div style="font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.6px;color:#94a3b8;margin-bottom:8px;">By Risk Profile</div>
    <div style="display:grid;grid-template-columns:repeat(3,1fr);gap:14px;">
      <div style="background:#fff5f5;border-radius:12px;padding:20px;border:1px solid #fecaca;display:flex;flex-direction:column;gap:4px;">
        <div style="font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#dc2626;">High Priority</div>
        <div style="font-size:46px;font-weight:900;line-height:1;color:#dc2626;margin:5px 0 3px;">{high_count}</div>
        <div style="font-size:13px;font-weight:500;color:#374151;">Comp or attainment at risk</div>
        <div style="font-size:10px;opacity:.75;margin-top:3px;line-height:1.5;">C1, C5, C6, or C12 failing<br>{high_names_html}</div>
      </div>
      <div style="background:#fffbeb;border-radius:12px;padding:20px;border:1px solid #fcd34d;display:flex;flex-direction:column;gap:4px;">
        <div style="font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#d97706;">Medium Priority</div>
        <div style="font-size:46px;font-weight:900;line-height:1;color:#d97706;margin:5px 0 3px;">{medium_count}</div>
        <div style="font-size:13px;font-weight:500;color:#374151;">Access issues only</div>
        <div style="font-size:10px;opacity:.75;margin-top:3px;line-height:1.5;">C8/C9/C10 — no comp impact<br>{medium_names_html}</div>
      </div>
      <div style="background:#f8fafc;border-radius:12px;padding:20px;border:1px solid #e2e8f0;opacity:.8;display:flex;flex-direction:column;gap:4px;">
        <div style="font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:#6b7280;">Low / Monitor</div>
        <div style="font-size:46px;font-weight:900;line-height:1;color:#9ca3af;margin:5px 0 3px;">{low_count}</div>
        <div style="font-size:13px;font-weight:500;color:#475569;">C7 metadata gap only — or fully passing</div>
        <div style="font-size:10px;opacity:.75;margin-top:3px;line-height:1.5;">SPECIALIST_GROUP missing for some employees<br>Low risk — no comp/access impact</div>
      </div>
    </div>
  </div>'''

html = f'''<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <meta name="snowflake-source" content="cortex-agent-authored"/>
  <title>Industry Onboarding Checks — {scope_label}</title>
  <script type="application/json" id="snowflake-report-metadata">
  {{
    "generated": "{datetime.now().strftime('%Y-%m-%d')}",
    "intent": "Industry onboarding check results — {scope_label}",
    "sections": [
      {{ "id": "kpis", "title": "Check KPIs",
         "producerNotes": "Risk profiles computed from IT.PIGMENT and SNOW_CERTIFIED. Re-run skill to refresh." }}
    ]
  }}
  </script>
  <style>
    :root {{ color-scheme: light; }}
    * {{ box-sizing: border-box; margin: 0; padding: 0; }}
    body {{
      font-family: -apple-system, system-ui, sans-serif;
      background: #f1f5f9;
      color: #1e293b;
      padding: 28px 20px 48px;
      min-height: 100vh;
    }}
    .header {{ max-width: 1160px; margin: 0 auto 24px; }}
    h1 {{
      font-size: 24px; font-weight: 800;
      color: #0f172a; margin-bottom: 5px;
    }}
    .meta {{ font-size: 13px; color: #64748b; }}
    .badge {{
      display: inline-block; padding: 2px 10px; border-radius: 99px;
      font-size: 11px; font-weight: 700; text-transform: uppercase;
      background: #dbeafe; color: #1d4ed8;
      margin-left: 8px; vertical-align: middle;
    }}
  </style>
</head>
<body>
  <div class="header">
    <h1>Industry Onboarding Checks <span class="badge">{scope_label}</span></h1>
    <div class="meta">{total_employees} active employees &nbsp;&middot;&nbsp; Run {run_date}</div>
  </div>
  {row1_html}
  {row2_html}

  <!-- ── Transfer Alert (only rendered if transfers_html is non-empty) ────── -->
  {transfers_html}

  <!-- ── Summary Table ───────────────────────────────────────────── -->
  <div style="max-width:1160px;margin:32px auto 0;">
    <h2 style="font-size:16px;font-weight:700;color:#0f172a;margin-bottom:14px;">Summary</h2>
    <div style="background:#fff;border-radius:10px;border:1px solid #e2e8f0;overflow:hidden;">
      <table style="width:100%;border-collapse:collapse;font-size:13px;">
        <thead>
          <tr style="background:#f8fafc;">
            <th style="padding:9px 16px;text-align:left;font-weight:600;font-size:11px;text-transform:uppercase;letter-spacing:.4px;color:#64748b;border-bottom:1px solid #e2e8f0;">Check</th>
            <th style="padding:9px 16px;text-align:center;font-weight:600;font-size:11px;text-transform:uppercase;letter-spacing:.4px;color:#64748b;border-bottom:1px solid #e2e8f0;">Risk Profile</th>
            <th style="padding:9px 16px;text-align:center;font-weight:600;font-size:11px;text-transform:uppercase;letter-spacing:.4px;color:#64748b;border-bottom:1px solid #e2e8f0;">Needs Review</th>
            <th style="padding:9px 16px;text-align:center;font-weight:600;font-size:11px;text-transform:uppercase;letter-spacing:.4px;color:#64748b;border-bottom:1px solid #e2e8f0;">Passed</th>
            <th style="padding:9px 16px;text-align:center;font-weight:600;font-size:11px;text-transform:uppercase;letter-spacing:.4px;color:#64748b;border-bottom:1px solid #e2e8f0;">Manual Check</th>
            <th style="padding:9px 16px;text-align:left;font-weight:600;font-size:11px;text-transform:uppercase;letter-spacing:.4px;color:#64748b;border-bottom:1px solid #e2e8f0;">Take Action</th>
          </tr>
        </thead>
        <tbody>{summary_rows_html}</tbody>
      </table>
    </div>
  </div>

  <!-- ── Detail Sections ─────────────────────────────────────────── -->
  <div style="max-width:1160px;margin:40px auto 0;">
    <h2 style="font-size:16px;font-weight:700;color:#0f172a;
               margin-bottom:20px;padding-bottom:10px;
               border-bottom:1px solid #e2e8f0;">Action Items</h2>
    {detail_sections_html}

  <!-- ── Manual Checks ──────────────────────────────────────────────── -->
  <div style="max-width:1160px;margin:40px auto 0;">
    <h2 style="font-size:16px;font-weight:700;color:#0f172a;margin-bottom:14px;padding-bottom:10px;border-bottom:1px solid #e2e8f0;">Manual Checks</h2>
    <div style="background:#fafafa;border:1px solid #e2e8f0;border-radius:10px;padding:16px 18px;">
      <div style="font-size:13px;font-weight:700;color:#0f172a;margin-bottom:6px;">C11 &mdash; CiQ (Attainment Payout)</div>
      <div style="font-size:12px;color:#64748b;">Verify each employee has been provisioned in CaptivateIQ. Cannot be automated &mdash; contact the In-Year Planning team. <span style="font-weight:500;">Contact In-Year Planning Team</span></div>
    </div>
  </div>

    <div style="font-size:11px;color:#94a3b8;padding-top:16px;
                border-top:1px solid #e2e8f0;">
      Generated by Cortex Code &nbsp;·&nbsp; Data from IT.PIGMENT, SNOW_CERTIFIED, Sigma, and Slack &nbsp;·&nbsp;
      <a href="https://docs.google.com/spreadsheets/d/1lC4-qnDxQzGzYwlQ5_y6XPON3ilkx-B2-gZgfP4GdTY/edit?gid=0#gid=0"
         target="_blank" style="color:#2563eb;">Process Repo</a>
    </div>
  </div>
</body>
</html>'''

with open(html_file, 'w') as f:
    f.write(html)

print(f'HTML report saved to: {html_file}')
```

Tell the user both paths:
```
CSV  → ~/.snowflake/cortex/skills/industry-onboarding-checks/audits/audit_{timestamp}.csv
HTML → ~/.snowflake/cortex/skills/industry-onboarding-checks/audits/report_{timestamp}.html
```

The HTML report can be opened directly in a browser or published via Snowflake Workspaces.

---

## Notes

- **Industry Principals & Architects only** — these tables are scoped to Industry GTM roles.
- **Active employees only** — every query joins `SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY` with `IS_EMPLOYEE_ACTIVE = TRUE`. Terminated or inactive employees in Pigment (e.g., Adam Kaufman, Tulio Quinones) are excluded automatically. Do not manually re-add them.
- **Corp bonus plan employees** are excluded from Checks 5 and 6 (targets and baselines) — no quotas by design.
- **FY dates are hardcoded for FY2026** (Feb 2026 – Jan 2027) directly in the Check 6 baselines query. Update the month-end dates when the fiscal year rolls.
- **Sigma check (Check 8)** runs in both single-person and whole-org mode. Single-person mode requires the employee's email provided in Step 1. Whole-org mode uses emails fetched in Step 1c from `SNOW_CERTIFIED.EMPLOYEE.D_EMPLOYEE_WORKDAY`.
- **Enrichment columns from D_EMPLOYEE_WORKDAY**: `EMPLOYEE_PREFERRED_NAME`, `EMPLOYEE_BUSINESS_TITLE`, `EMPLOYEE_REGION`, `EMPLOYEE_HIRE_DATE_AT`, `EMPLOYEE_MANAGER_NAME`. Surface these in the final summary for context.
- **Checks 9 and 10** (Slack) are automated via Slack MCP (`mcp_slack-snow_slack_list_channel_members`, `mcp_slack-snow_slack_search_users`). They apply to **Industry Architects only** — skip entirely for `Industry GTM` and `Industry GTM Mgmt` roles. If Slack MCP is unavailable, fall back to manual.
- **Manual checks** (4 and 11) cannot be validated via SQL. Always surface these as `MANUAL CHECK — requires human verification`.
- **Slack checks run in ~10 pages** of 30 members each for `#cx-specialists` (~300 members). Always start Slack collection in Round 1 in parallel with SQL checks so the wait is hidden.
- `QUOTA_START_DATE` is stored as VARCHAR (`MM/DD/YYYY HH:MM:SS TZ`). Use `TO_DATE(LEFT(..., 10), 'MM/DD/YYYY')` for date comparisons.
- `ACTIVE_RECORD` is a BOOLEAN column. Use `ACTIVE_RECORD = TRUE` (not the string `'TRUE'`).
- **Attainment Dashboard** (Check 12) depends on Metadata (Check 7) and Baselines (Check 6) being correct first — a NEEDS REVIEW here is often downstream of upstream gaps.
- Reference: [Process Repo](https://docs.google.com/spreadsheets/d/1lC4-qnDxQzGzYwlQ5_y6XPON3ilkx-B2-gZgfP4GdTY/edit?gid=0#gid=0) | [Validations](https://docs.google.com/spreadsheets/d/1lC4-qnDxQzGzYwlQ5_y6XPON3ilkx-B2-gZgfP4GdTY/edit?gid=829379486#gid=829379486)
