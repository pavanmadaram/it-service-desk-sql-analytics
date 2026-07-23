# IT Service Desk Analytics — SQL Portfolio Project

## Overview

This project analyzes 6 months of IT service desk ticket data to answer real operational questions a service desk manager would actually ask: are we keeping up with demand, which teams are missing SLA targets, where is the backlog building up, and who's performing well.

The dataset (~1,500 tickets across 5 teams and 15 agents) was built to reflect realistic patterns — uneven weekly volume, SLA targets that vary by ticket type and priority, and teams with different breach tendencies — so the analysis surfaces genuine, non-obvious findings rather than a flat, uniform dataset.

All query outputs below are actual results from running these queries in MySQL Workbench against the dataset in this repo.

**Tech used:** MySQL (CTEs, window functions, conditional aggregation, date logic, pivoting)

---

## Schema

**`teams`** — team_id, team_name, capacity_per_week
**`agents`** — agent_id, agent_name, team_id
**`tickets`** — ticket_id, ticket_type, priority, team_id, agent_id, submitted_date, sla_target_days, resolved_date, status

---

## Business Questions & Findings

### 1. Weekly Ticket Volume Trend

**Question:** What's our weekly ticket volume trend over the last 6 months, and is it growing or shrinking?

```sql
WITH weekly_counts AS (
    SELECT 
        DATE_FORMAT(submitted_date, '%Y-%u') AS week_label,
        COUNT(*) AS ticket_count
    FROM tickets
    WHERE submitted_date >= CURDATE() - INTERVAL 6 MONTH
    GROUP BY week_label
)
SELECT *,
       ROUND(AVG(ticket_count) OVER (ORDER BY week_label ROWS BETWEEN 3 PRECEDING AND CURRENT ROW), 2) AS rolling_4wk_avg
FROM weekly_counts
ORDER BY week_label;
```

**Output (sample):**

| week_label | ticket_count | rolling_4wk_avg |
|---|---|---|
| 2026-04 | 8 | 8.00 |
| 2026-05 | 59 | 33.50 |
| 2026-06 | 53 | 40.00 |
| 2026-07 | 65 | 46.25 |
| 2026-08 | 59 | 59.00 |
| 2026-09 | 74 | 62.75 |
| 2026-10 | 56 | 63.50 |
| 2026-11 | 58 | 61.75 |
| 2026-12 | 72 | 65.00 |
| 2026-13 | 83 | 67.25 |
| ... | ... | ... |

**Finding:** After a partial first week, volume quickly stabilizes in the 50s–80s per week, and the 4-week rolling average climbs steadily from ~34 to the mid-60s and beyond — a clear, sustained growth trend rather than random noise.

---

### 2. SLA Breach Rate by Team

**Question:** Which teams are breaching SLA most often, and by how many days on average when they do?

```sql
WITH resolved_ticket_sla AS (
    SELECT 
        t.team_id, tm.team_name,
        DATEDIFF(t.resolved_date, t.submitted_date) AS time_taken,
        t.sla_target_days
    FROM tickets AS t
    JOIN teams AS tm ON t.team_id = tm.team_id
    WHERE t.status = 'Resolved'
)
SELECT 
    team_name, 
    COUNT(*) AS total_resolved_tickets, 
    SUM(CASE WHEN time_taken > sla_target_days THEN 1 ELSE 0 END) AS breach_count,
    ROUND(SUM(CASE WHEN time_taken > sla_target_days THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS breach_rate,
    ROUND(AVG(CASE WHEN time_taken > sla_target_days THEN time_taken - sla_target_days END), 2) AS avg_days_overdue
FROM resolved_ticket_sla
GROUP BY team_name
ORDER BY breach_rate DESC;
```

**Output:**

| team_name | total_resolved_tickets | breach_count | breach_rate | avg_days_overdue |
|---|---|---|---|---|
| Hardware Support | 195 | 148 | 75.90 | 3.38 |
| Network Operations | 146 | 102 | 69.86 | 4.37 |
| Application Support | 244 | 145 | 59.43 | 4.06 |
| Helpdesk Tier 1 | 319 | 179 | 56.11 | 2.22 |
| Security & Access | 274 | 133 | 48.54 | 3.77 |

**Finding:** **Hardware Support** breaches SLA on over 3 out of every 4 resolved tickets (75.9%) — by far the worst rate in the org — with **Network Operations** close behind at 69.9% and also running the longest overdue on average (4.37 days past target). Security & Access has the *lowest* breach rate on resolved tickets (48.5%), which is worth reading alongside Q5 below — it suggests this team isn't resolving badly, it's simply not resolving *enough* of what comes in (see backlog and capacity findings).

---

### 3. Turnaround Time by Ticket Type & Priority

**Question:** What's the average resolution time for each ticket type, broken down by priority level?

```sql
WITH resolution_times AS (
    SELECT 
        ticket_type, priority, 
        DATEDIFF(resolved_date, submitted_date) AS resolution_days
    FROM tickets
    WHERE status = 'Resolved'
)
SELECT 
    ticket_type, priority, 
    ROUND(AVG(resolution_days), 2) AS avg_resolution_days, 
    COUNT(*) AS ticket_count
FROM resolution_times
GROUP BY ticket_type, priority
ORDER BY ticket_type, priority;
```

**Output (sample):**

| ticket_type | priority | avg_resolution_days | ticket_count |
|---|---|---|---|
| Access Request | High | 2.46 | 35 |
| Access Request | Low | 6.62 | 69 |
| Access Request | Medium | 3.07 | 107 |
| Access Request | Urgent | 2.10 | 10 |
| Account Setup | High | 2.74 | 27 |
| Account Setup | Low | 7.45 | 31 |
| Account Setup | Medium | 3.74 | 65 |
| Account Setup | Urgent | 1.80 | 10 |
| Hardware Issue | High | 3.14 | 57 |
| Hardware Issue | Low | 8.50 | 88 |
| Hardware Issue | Medium | 6.36 | 137 |
| Hardware Issue | Urgent | 1.53 | 17 |
| ... | ... | ... | ... |

**Finding:** Resolution time scales as expected with priority within each ticket type — Urgent tickets consistently resolve fastest, Low the slowest. Notably, **Hardware Issue (Medium)** takes 6.36 days on average across 137 tickets — a large volume moving slowly, and a likely contributor to Hardware Support's high breach rate in Q2.

---

### 4. Current Backlog — Overdue Open Tickets

**Question:** Of all currently open tickets, which are already past their SLA deadline, and how overdue are they?

```sql
WITH open_ticket_aging AS (
    SELECT 
        t.ticket_id, t.ticket_type, t.priority, tm.team_name,
        DATEDIFF(CURDATE(), t.submitted_date) AS days_open,
        t.sla_target_days,
        DATEDIFF(CURDATE(), t.submitted_date) - t.sla_target_days AS days_overdue
    FROM tickets AS t
    JOIN teams AS tm ON t.team_id = tm.team_id
    WHERE t.status = 'In Progress'
)
SELECT * FROM open_ticket_aging
WHERE days_overdue > 0
ORDER BY days_overdue DESC;
```

**Output (top rows):**

| ticket_id | ticket_type | priority | team_name | days_open | sla_target_days | days_overdue |
|---|---|---|---|---|---|---|
| 1337 | Network Issue | High | Network Operations | 38 | 2 | 36 |
| 1346 | Software Issue | Medium | Helpdesk Tier 1 | 38 | 3 | 35 |
| 1381 | Network Issue | Medium | Helpdesk Tier 1 | 37 | 3 | 34 |
| 1358 | Account Setup | Medium | Security & Access | 37 | 3 | 34 |
| 1369 | Software Issue | High | Application Support | 36 | 2 | 34 |
| 1345 | Network Issue | Medium | Network Operations | 36 | 3 | 33 |
| 1348 | Hardware Issue | Medium | Helpdesk Tier 1 | 37 | 4 | 33 |
| ... | ... | ... | ... | ... | ... | ... |

**Finding:** Every open ticket in this list is overdue by **30+ days** against SLA targets of just 2–5 days — this isn't a mild backlog, it's tickets sitting untouched for roughly 10x their expected resolution window. The backlog is stale, not just large, suggesting open tickets are being deprioritized rather than actively worked once they age past a certain point.

---

### 5. Team Workload vs. Capacity

**Question:** Which teams are consistently over capacity, and in which weeks?

```sql
WITH team_tickets AS (
    SELECT t.ticket_id, tm.team_name, t.submitted_date, tm.capacity_per_week
    FROM tickets AS t
    JOIN teams AS tm ON t.team_id = tm.team_id
),
weekly_team_volume AS (
    SELECT team_name, DATE_FORMAT(submitted_date, '%Y-%u') AS week_label, 
           COUNT(*) AS ticket_count, capacity_per_week
    FROM team_tickets
    GROUP BY team_name, week_label, capacity_per_week
)
SELECT *, ticket_count - capacity_per_week AS over_capacity_by
FROM weekly_team_volume
WHERE ticket_count > capacity_per_week
ORDER BY over_capacity_by DESC;
```

**Output (top rows):**

| team_name | week_label | ticket_count | capacity_per_week | over_capacity_by |
|---|---|---|---|---|
| Security & Access | 2026-13 | 23 | 8 | 15 |
| Security & Access | 2026-12 | 19 | 8 | 11 |
| Security & Access | 2026-11 | 18 | 8 | 10 |
| Security & Access | 2026-07 | 17 | 8 | 9 |
| Security & Access | 2026-16 | 17 | 8 | 9 |
| Security & Access | 2026-04 | 16 | 8 | 8 |
| Security & Access | 2026-09 | 16 | 8 | 8 |
| ... | ... | ... | ... | ... |

**Finding:** **Security & Access dominates every one of the top over-capacity weeks** — at its worst (week 2026-13), the team received **23 tickets against a capacity of just 8**, nearly 3x its sustainable load. This directly explains the Q2 finding: the team isn't breaching SLA on what it resolves because it's structurally too small for its incoming demand — the strain shows up as backlog and unresolved volume instead of late resolutions.

---

### 6. Top Performing Agents per Team

**Question:** Who are the top 3 performing agents per team, based on tickets resolved within SLA?

```sql
WITH resolved_tickets AS (
    SELECT t.ticket_id, tm.team_name, a.agent_name,
           DATEDIFF(t.resolved_date, t.submitted_date) AS days_taken, t.sla_target_days
    FROM tickets AS t
    JOIN teams AS tm ON t.team_id = tm.team_id
    JOIN agents AS a ON t.agent_id = a.agent_id
    WHERE t.status = 'Resolved'
),
agent_sla_performance AS (
    SELECT team_name, agent_name, 
           SUM(CASE WHEN days_taken <= sla_target_days THEN 1 ELSE 0 END) AS tickets_within_sla
    FROM resolved_tickets
    GROUP BY team_name, agent_name
),
ranked_agents AS (
    SELECT *, DENSE_RANK() OVER (PARTITION BY team_name ORDER BY tickets_within_sla DESC) AS agent_rank
    FROM agent_sla_performance
)
SELECT * FROM ranked_agents WHERE agent_rank <= 3
ORDER BY team_name, agent_rank;
```

**Output:**

| team_name | agent_name | tickets_within_sla | agent_rank |
|---|---|---|---|
| Application Support | Farhan Khan | 37 | 1 |
| Application Support | Priya Menon | 32 | 2 |
| Application Support | Neha Verma | 30 | 3 |
| Hardware Support | Divya Nair | 30 | 1 |
| Hardware Support | Karan Mehta | 17 | 2 |
| Helpdesk Tier 1 | Sneha Iyer | 64 | 1 |
| Helpdesk Tier 1 | Ravi Kumar | 42 | 2 |
| Helpdesk Tier 1 | Amit Shah | 34 | 3 |
| Network Operations | Vikram Singh | 13 | 1 |
| Network Operations | Ananya Rao | 12 | 2 |
| Network Operations | Rohit Sharma | 11 | 3 |
| Security & Access | Meera Pillai | 72 | 1 |
| Security & Access | Arjun Nair | 69 | 2 |

**Finding:** Performance gaps within teams can be large — on Helpdesk Tier 1, the top agent (Sneha Iyer, 64 tickets within SLA) nearly doubles the third-ranked agent (Amit Shah, 34). Hardware Support only shows 2 agents total, meaning its already-low headcount is a compounding factor in that team's high breach rate from Q2.

---

### 7. Month-over-Month SLA Compliance Trend

**Question:** Is our SLA compliance rate improving or declining month over month?

```sql
WITH resolved_ticket_sla AS (
    SELECT t.team_id, tm.team_name, t.resolved_date,
           DATEDIFF(t.resolved_date, t.submitted_date) AS time_taken, t.sla_target_days
    FROM tickets AS t
    JOIN teams AS tm ON t.team_id = tm.team_id
    WHERE t.status = 'Resolved'
),
monthly_compliance AS (
    SELECT DATE_FORMAT(resolved_date, '%Y-%m') AS month_label,
           ROUND(SUM(CASE WHEN time_taken <= sla_target_days THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS compliance_rate
    FROM resolved_ticket_sla
    GROUP BY month_label
)
SELECT *, 
       LAG(compliance_rate) OVER (ORDER BY month_label) AS previous_month_rate, 
       compliance_rate - LAG(compliance_rate) OVER (ORDER BY month_label) AS change_in_percentage
FROM monthly_compliance
ORDER BY month_label;
```

**Output:**

| month_label | compliance_rate | previous_month_rate | change_in_percentage |
|---|---|---|---|
| 2026-01 | 41.67 | NULL | NULL |
| 2026-02 | 40.65 | 41.67 | -1.02 |
| 2026-03 | 41.30 | 40.65 | 0.65 |
| 2026-04 | 39.90 | 41.30 | -1.40 |
| 2026-05 | 43.10 | 39.90 | 3.20 |
| 2026-06 | 36.93 | 43.10 | -6.17 |
| 2026-07 | 26.53 | 36.93 | -10.40 |

**Finding:** This is the headline concern of the whole analysis. Compliance hovered in a relatively stable 40–43% range through May, then **collapsed in the final two months** — dropping 6.2 points in June and a further 10.4 points in July, ending at just **26.53%**, down 15 points from its January starting level. The decline isn't just happening, it's *accelerating*. Read alongside Q1 (rising volume) and Q5 (capacity strain concentrated in specific teams), the story is clear: **incoming demand has outpaced team capacity, and the gap is widening month over month, not stabilizing.**

---

### 8. Ticket Type Mix Over Time

**Question:** Has the mix of ticket types shifted over the past 6 months?

```sql
WITH monthly_type_counts AS (
    SELECT DATE_FORMAT(submitted_date, '%Y-%m') AS month_label, ticket_type, COUNT(*) AS ticket_count
    FROM tickets
    GROUP BY month_label, ticket_type
)
SELECT 
    month_label,
    COALESCE(MAX(CASE WHEN ticket_type = 'Hardware Issue' THEN ticket_count END), 0) AS hardware_issue,
    COALESCE(MAX(CASE WHEN ticket_type = 'Software Issue' THEN ticket_count END), 0) AS software_issue,
    COALESCE(MAX(CASE WHEN ticket_type = 'Access Request' THEN ticket_count END), 0) AS access_request,
    COALESCE(MAX(CASE WHEN ticket_type = 'Network Issue' THEN ticket_count END), 0) AS network_issue,
    COALESCE(MAX(CASE WHEN ticket_type = 'Account Setup' THEN ticket_count END), 0) AS account_setup
FROM monthly_type_counts
GROUP BY month_label
ORDER BY month_label;
```

**Output:**

| month_label | hardware_issue | software_issue | access_request | network_issue | account_setup |
|---|---|---|---|---|---|
| 2026-01 | 55 | 64 | 32 | 39 | 29 |
| 2026-02 | 66 | 71 | 36 | 49 | 29 |
| 2026-03 | 63 | 83 | 72 | 38 | 34 |
| 2026-04 | 58 | 82 | 46 | 32 | 28 |
| 2026-05 | 65 | 68 | 38 | 34 | 22 |
| 2026-06 | 66 | 68 | 45 | 35 | 20 |
| 2026-07 | 12 | 8 | 4 | 8 | 4 |

**Finding:** Software Issues are consistently the largest category month over month. **Access Requests spike sharply in March (72, roughly double the surrounding months)** — likely tied to a specific event (e.g., a system migration or a new hire batch) worth investigating with the actual team. July's low totals reflect a partial month of data, not a real drop in demand.

---

## Key Assumptions Made

Several business questions were ambiguous by design (matching real stakeholder requests) — the assumptions made are stated explicitly rather than silently baked into the queries:

- **SLA breach** = actual resolution time *strictly greater than* target (equal to target = met, not breached).
- **Workload/capacity (Q5)** counts *all* submitted tickets, including cancelled ones, since triage still consumes capacity regardless of outcome.
- **Monthly SLA compliance (Q7)** is grouped by `resolved_date`, not `submitted_date` — compliance reflects the month the outcome was determined, not the month the ticket arrived.
- **Ticket type mix (Q8)** is grouped by `submitted_date` — this question is about intake patterns, not resolution performance.

---

## What This Project Demonstrates

- CTEs and multi-step query composition
- Window functions (rolling averages, `LAG()`, `DENSE_RANK()`)
- Conditional aggregation (`CASE WHEN` inside `SUM`/`AVG`)
- Date/time logic (business-relevant SLA and aging calculations)
- Pivoting with `MAX(CASE WHEN...)`
- Explicit handling of ambiguous business logic — stating assumptions rather than guessing silently
- Cross-question synthesis — connecting findings across separate queries (volume, capacity, and compliance) into one coherent narrative
