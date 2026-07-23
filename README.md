# IT Service Desk Analytics — SQL Portfolio Project

## Overview

This project analyzes 6 months of IT service desk ticket data to answer real operational questions a service desk manager would actually ask: are we keeping up with demand, which teams are missing SLA targets, where is the backlog building up, and who's performing well.

The dataset (~1,500 tickets across 5 teams and 15 agents) was built to reflect realistic patterns — uneven weekly volume, SLA targets that vary by ticket type and priority, and teams with different breach tendencies — so the analysis surfaces genuine, non-obvious findings rather than a flat, uniform dataset.

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

**Finding:** Weekly volume trended upward over the period, from roughly the high-teens/40s early on to a sustained 55–75 tickets/week by the final month. The 4-week rolling average smooths out spike weeks (system rollouts) and slow weeks (holidays) to reveal a consistent underlying growth trend rather than random noise.

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

**Finding:** Breach rates vary sharply by team — **Hardware Support (64.6%)** and **Network Operations (63.7%)** breach far more often than **Security & Access (44.5%)** and **Helpdesk Tier 1 (46.1%)**. Hardware and Network teams also run the longest overdue (3.4–4.3 days past target on average), pointing to a resourcing or process gap specific to those two teams rather than a company-wide SLA problem.

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

**Finding:** Resolution time scales predictably with priority within each ticket type (e.g., Access Requests: 1.5 days at Urgent vs. 6.1 days at Low) — confirming prioritization logic is generally working as intended. This baseline is useful for spotting future anomalies (e.g., an Urgent ticket type suddenly taking as long as a Low one).

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

**Finding:** Of the open tickets currently in the queue, nearly all are already past their SLA deadline — some by over a month. This is a clear, actionable red flag: the backlog isn't just large, it's stale, suggesting open tickets are being deprioritized rather than actively worked.

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

**Finding:** **Security & Access** is the team most consistently over capacity — appearing repeatedly at the top of over-capacity weeks, at times handling nearly 3x its weekly capacity (23 tickets against a capacity of 8). This directly explains why it also shows a high SLA breach rate: the team is structurally under-resourced relative to its incoming demand, not underperforming.

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

**Finding:** Performance varies meaningfully within teams, not just across them — e.g., on Helpdesk Tier 1, the top agent resolved 74 tickets within SLA vs. 42 for the third-ranked agent. This is useful both for recognition and for identifying who might mentor lower-performing teammates.

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

**Finding:** This is the headline concern of the whole analysis. SLA compliance rose slightly through Q1 (47.7% → 50.4%), then declined steadily from April onward, dropping to **36.7% by July** — a fall of nearly 14 points from its peak. Read alongside Q1 (rising volume) and Q5 (capacity strain), this tells a clear story: **incoming demand has outpaced team capacity, and compliance is deteriorating as a direct result.**

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

**Finding:** Software Issues are consistently the largest single category month over month, with Access Requests notably spiking in March (72, up from 32–36 in prior months) — likely tied to a specific event (e.g., a system migration or new hire batch) worth investigating with the actual team.

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
