# Technical Exercise: Support Data Scenario

## 1. First response time by team

```sql
SELECT
    a.team,
    AVG(t.first_response_minutes) AS avg_first_response_minutes
FROM tickets t
JOIN agents a ON t.agent_id = a.agent_id
WHERE t.closed_at >= CURRENT_TIMESTAMP - INTERVAL '30 days'
GROUP BY a.team;
```

## 2. Agents with above-average reopen rates

```sql
WITH agent_rates AS (
    SELECT
        a.agent_id,
        a.name,
        a.team,
        AVG(CASE WHEN t.reopened_count > 0 THEN 1.0 ELSE 0.0 END) AS reopen_rate
    FROM agents a
    JOIN tickets t ON a.agent_id = t.agent_id
    GROUP BY a.agent_id, a.name, a.team
),
team_rates AS (
    SELECT
        *,
        AVG(reopen_rate) OVER (PARTITION BY team) AS team_avg
    FROM agent_rates
)
SELECT
    agent_id,
    name,
    team,
    reopen_rate
FROM team_rates
WHERE reopen_rate > team_avg;
```

## 3. CSAT trend by category

```sql
SELECT
    DATE_TRUNC('month', c.submitted_at) AS month,
    t.category,
    AVG(c.score) AS avg_csat
FROM csat_responses c
JOIN tickets t ON c.ticket_id = t.ticket_id
WHERE c.submitted_at >= CURRENT_DATE - INTERVAL '3 months'
GROUP BY
    DATE_TRUNC('month', c.submitted_at),
    t.category
ORDER BY month, t.category;
```

## 4. Digging in

I would start with `tickets` and compare Billing with previous months, especially first response time, reopen rate, priority and status.

I would also want issue type/subcategory, ticket tags, resolution time, escalations, macros used and CSAT comments. This would help identify if the drop is related to one specific issue instead of the whole Billing category.

## 5. Testing a theory

**Theory:** Billing CSAT dropped because more tickets were being reopened.

```sql
SELECT
    CASE
        WHEN t.reopened_count > 0 THEN 'Reopened'
        ELSE 'Not Reopened'
    END AS reopen_status,
    COUNT(*) AS tickets,
    AVG(c.score) AS avg_csat
FROM tickets t
JOIN csat_responses c ON t.ticket_id = c.ticket_id
WHERE t.category = 'Billing'
  AND c.submitted_at >= CURRENT_DATE - INTERVAL '3 months'
GROUP BY
    CASE
        WHEN t.reopened_count > 0 THEN 'Reopened'
        ELSE 'Not Reopened'
    END;
```

I would compare the CSAT of reopened vs. non-reopened tickets. If reopened tickets have much lower CSAT, I would then investigate what issue is causing those reopens.

## 6. What I'd actually do

If most reopens came from one specific issue type, I would first review some tickets to confirm the issue.

I have worked with similar situations as a Team Lead. When we had a technical issue that increased the queue or affected CSAT, we treated it as an incident. We identified the trend, created a predefined macro with the correct information and tags, and tracked those tickets separately.

If we were waiting for a Tech/Product squad, the macro explained the situation and set expectations with the customer. If we already had the solution, agents could resolve it directly using the macro.

For this case, I would do the same: update the macro, add a specific tag, define when agents should resolve or escalate, and monitor the queue, reopen rate and CSAT during the following week.

## 7. Reporting up and coaching down

**To my manager:**

Billing CSAT dropped mainly because reopened tickets increased around one specific issue type. We are updating the macro and escalation process and will monitor reopen rate and CSAT to see if it improves.

**In an agent 1:1:**

I would review a specific ticket with the agent and explain what could be clearer in the first response. I would focus on setting better expectations and next steps, then review the results again after using the updated process.
