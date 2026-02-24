---
name: Egym
assetId: 8318e89c-1671-4a63-a4d7-43c37e3d5d8d
type: page
---

# Egym User Analysis

```sql egym_users
SELECT
    u.first_name || ' ' || u.last_name as user_name,
    u.email,
    coalesce(activity.total_events, 0) as total_events,
    coalesce(activity.total_sessions, 0) as total_sessions,
    coalesce(activity.active_days, 0) as active_days,
    coalesce(activity.last_seen, null) as last_seen,
    coalesce(agents.active_agents, 0) as active_agents,
    coalesce(agents.inactive_agents, 0) as inactive_agents,
    coalesce(agents.total_agents, 0) as total_agents,
    coalesce(runs.total_runs, 0) as total_runs,
    coalesce(runs.successful_runs, 0) as successful_runs,
    coalesce(runs.error_runs, 0) as error_runs
FROM tahoe_public_user_profiles u
LEFT JOIN (
    SELECT
        pp.property_email as email,
        count(DISTINCT e.id) as total_events,
        count(DISTINCT e.session_id) as total_sessions,
        count(DISTINCT toDate(e.time_stamp)) as active_days,
        max(e.time_stamp) as last_seen
    FROM posthog_person pp
    JOIN posthog_event e ON e.person_id = pp.id
    GROUP BY pp.property_email
) activity ON activity.email = u.email
LEFT JOIN (
    SELECT
        owner_user_id,
        countIf(enabled = true AND archived = false) as active_agents,
        countIf(enabled = false OR archived = true) as inactive_agents,
        count(*) as total_agents
    FROM tahoe_public_agents
    GROUP BY owner_user_id
) agents ON agents.owner_user_id = u.id
LEFT JOIN (
    SELECT
        a.owner_user_id,
        count(DISTINCT r.id) as total_runs,
        countIf(r.status = 'success') as successful_runs,
        countIf(r.status = 'error') as error_runs
    FROM tahoe_public_agents a
    JOIN tahoe_public_runs r ON r.agent_id = a.id
    GROUP BY a.owner_user_id
) runs ON runs.owner_user_id = u.id
WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
ORDER BY total_events DESC, user_name ASC
```

{% big_value
  data="egym_users"
  value="count(*)"
  title="Total Users"
/%}

```sql active_agent_users
SELECT count(DISTINCT u.id) as active_agent_users
FROM tahoe_public_user_profiles u
JOIN tahoe_public_agents a ON a.owner_user_id = u.id
WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
  AND a.enabled = true AND a.archived = false
```

{% big_value
  data="active_agent_users"
  value="sum(active_agent_users)"
  title="Active Agent Users"
/%}

```sql wau
SELECT count(DISTINCT u.id) as weekly_active_users
FROM tahoe_public_user_profiles u
JOIN posthog_person pp ON pp.property_email = u.email
JOIN posthog_event e ON e.person_id = pp.id
WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
  AND e.time_stamp >= (
    SELECT max(e2.time_stamp) - INTERVAL 7 DAY
    FROM posthog_event e2
    JOIN posthog_person pp2 ON pp2.id = e2.person_id
    JOIN tahoe_public_user_profiles u2 ON u2.email = pp2.property_email
    WHERE u2.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
  )
```

{% big_value
  data="wau"
  value="sum(weekly_active_users)"
  title="Weekly Active Users"
/%}

{% big_value
  data="egym_users"
  value="sum(total_events)"
  title="Total Events"
  fmt="num0"
/%}

{% big_value
  data="egym_users"
  value="sum(active_agents)"
  title="Active Agents"
  fmt="num0"
/%}

{% big_value
  data="egym_users"
  value="sum(total_runs)"
  title="Total Runs"
  fmt="num0"
/%}

## User Details

{% table
  data="egym_users"
  subtotals=false
  search=true
  row_shading=true
%}
  {% dimension value="user_name" /%}
  {% dimension value="email" /%}
  {% measure value="sum(total_events)" fmt="num0" sort="desc" /%}
  {% measure value="sum(total_sessions)" fmt="num0" /%}
  {% measure value="sum(active_days)" fmt="num0" /%}
  {% measure value="max(last_seen)" fmt="shortdate" /%}
  {% measure value="sum(active_agents)" fmt="num0" /%}
  {% measure value="sum(inactive_agents)" fmt="num0" /%}
  {% measure value="sum(total_runs)" fmt="num0" /%}
  {% measure value="sum(successful_runs)" fmt="num0" /%}
  {% measure value="sum(error_runs)" fmt="num0" /%}
{% /table %}

## Activity by User

{% bar_chart
  data="egym_users"
  x="user_name"
  y="sum(total_events)"
  y_fmt="num0"
  order="sum(total_events) desc"
  title="Total Events by User"
/%}

## Agents by User

```sql agent_status
SELECT
    u.first_name || ' ' || u.last_name as user_name,
    'Active' as agent_status,
    countIf(a.enabled = true AND a.archived = false) as agent_count
FROM tahoe_public_user_profiles u
JOIN tahoe_public_agents a ON a.owner_user_id = u.id
WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
GROUP BY user_name

UNION ALL

SELECT
    u.first_name || ' ' || u.last_name as user_name,
    'Inactive' as agent_status,
    countIf(a.enabled = false OR a.archived = true) as agent_count
FROM tahoe_public_user_profiles u
JOIN tahoe_public_agents a ON a.owner_user_id = u.id
WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
GROUP BY user_name

ORDER BY user_name
```

{% bar_chart
  data="agent_status"
  x="user_name"
  y="sum(agent_count)"
  series="agent_status"
  y_fmt="num0"
  title="Active vs Inactive Agents by User"
  stacked=true
  chart_options={
    series_colors={
      "Active"="#22c55e"
      "Inactive"="#ef4444"
    }
  }
/%}

## Agents Over Time

```sql agents_over_time
SELECT
    date,
    'Total Agents' as metric,
    cumulative_total_agents as agent_count
FROM (
    SELECT
        toDate(fromUnixTimestamp64Milli(a.created_at)) as date,
        sum(count(*)) OVER (ORDER BY toDate(fromUnixTimestamp64Milli(a.created_at))) as cumulative_total_agents
    FROM tahoe_public_agents a
    JOIN tahoe_public_user_profiles u ON u.id = a.owner_user_id
    WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
    GROUP BY date
)

UNION ALL

SELECT
    date,
    'Active Agents' as metric,
    cumulative_active_agents as agent_count
FROM (
    SELECT
        toDate(fromUnixTimestamp64Milli(a.created_at)) as date,
        sum(sumIf(1, a.enabled = true AND a.archived = false)) OVER (ORDER BY toDate(fromUnixTimestamp64Milli(a.created_at))) as cumulative_active_agents
    FROM tahoe_public_agents a
    JOIN tahoe_public_user_profiles u ON u.id = a.owner_user_id
    WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
    GROUP BY date
)

ORDER BY date, metric
```

{% line_chart
  data="agents_over_time"
  x="date"
  y="sum(agent_count)"
  series="metric"
  y_fmt="num0"
  title="Cumulative Agents Over Time"
  subtitle="Total agents created vs currently active"
  chart_options={
    series_colors={
      "Total Agents"="#6366f1"
      "Active Agents"="#22c55e"
    }
  }
/%}

## Credits Used Over Time

```sql credits_over_time
SELECT
    toDate(fromUnixTimestamp64Milli(c.created_at)) as date,
    sum(c.credits_charged) as daily_credits,
    sum(sum(c.credits_charged)) OVER (ORDER BY toDate(fromUnixTimestamp64Milli(c.created_at))) as cumulative_credits
FROM tahoe_public_credit_usage_events c
JOIN tahoe_public_user_profiles u ON u.id = c.user_id
WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
GROUP BY date
ORDER BY date
```

{% line_chart
  data="credits_over_time"
  x="date"
  y="sum(cumulative_credits)"
  y_fmt="num0"
  title="Cumulative Credits Used"
  subtitle="Running total of credit consumption over time"
/%}

## Agent Runs Over Time

```sql runs_over_time
SELECT
    toDate(fromUnixTimestamp64Milli(r.start_time)) as date,
    count(*) as daily_runs,
    sum(count(*)) OVER (ORDER BY toDate(fromUnixTimestamp64Milli(r.start_time))) as cumulative_runs
FROM tahoe_public_runs r
JOIN tahoe_public_agents a ON a.id = r.agent_id
JOIN tahoe_public_user_profiles u ON u.id = a.owner_user_id
WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
GROUP BY date
ORDER BY date
```

{% line_chart
  data="runs_over_time"
  x="date"
  y="sum(cumulative_runs)"
  y_fmt="num0"
  title="Cumulative Agent Runs"
  subtitle="Running total of agent runs executed over time"
/%}

## Cumulative Cost (USD)

```sql cost_over_time
SELECT
    toDate(fromUnixTimestamp64Milli(c.created_at)) as date,
    sum(sum(c.credits_charged) * 0.01) OVER (ORDER BY toDate(fromUnixTimestamp64Milli(c.created_at))) as cumulative_cost_usd
FROM tahoe_public_credit_usage_events c
JOIN tahoe_public_user_profiles u ON u.id = c.user_id
WHERE u.organization_id = 'org_ofmj0t5ozgrcmqbpiw61'
GROUP BY date
ORDER BY date
```

{% line_chart
  data="cost_over_time"
  x="date"
  y="sum(cumulative_cost_usd)"
  y_fmt="usd2"
  title="Cumulative Cost in USD"
  subtitle="1 credit = $0.01 USD"
/%}
