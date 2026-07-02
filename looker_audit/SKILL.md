---
name: looker-audit
description: >-
  Audits a Looker instance for performance, governance, and cleanup opportunities.
  Use when analyzing dashboard performance, identifying unused LookML components,
  checking for excessive table calculations, or auditing datagroup health.
  Don't use for general Looker onboarding or creating new dashboards.
---

# Looker Audit & Health Check

This skill guides you through auditing a Looker instance using `looker-cli` and System Activity to identify performance bottlenecks, governance gaps, and cleanup opportunities.

## ⚡ Quick Health Audits (Pulse Commands)
Before running detailed custom queries, you can use pre-packaged `looker-cli health` commands for a rapid overview of instance health. These are often faster than inline queries for a quick pulse check.

- **Dashboard Performance**: Find dashboards with queries slower than 30 seconds in the last 7 days.
  ```bash
  looker-cli health pulse dashboard-performance
  ```
- **Dashboard Errors**: Check for dashboards with erroring queries in the last 7 days.
  ```bash
  looker-cli health pulse dashboard-errors
  ```
- **Slow Explores**: Identify the slowest explores in the past 7 days.
  ```bash
  looker-cli health pulse explore-performance
  ```
- **Schedule Failures**: Quick check for failing schedules.
  ```bash
  looker-cli health pulse schedule-failures
  ```
- **Database Connections**: Verify connection status and query volume.
  ```bash
  looker-cli health pulse db-connections
  ```

> [!TIP]
> **Audit Strategy**:
> *   Use **Pulse/Vacuum commands** for a broad, fast initial assessment (e.g., "Is the instance healthy? What are the top 5 slow things?").
> *   Use **System Activity Inline Queries** (Checklist below) when you need deep drill-downs, specific timeframes (e.g., 90 days vs 7 days), or custom filtering (e.g., filtering by a specific user or custom thresholds).

## 📋 Audit Checklist



### 1. Dashboard Health & Complexity
Excessive complexity on dashboards leads to browser lag and poor user experience.

- **Auto-Refresh**: Identify dashboards running unnecessary constant queries.
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.id","dashboard.title","dashboard.refresh_interval"],"filters":{"dashboard.refresh_interval":"-NULL"},"limit":"100"}' | looker-cli api query run_inline_query json - | jq
  ```
- **Dynamic Fields**: Excessive dynamic fields (Table Calculations, Custom Fields) bypass LookML governance and strain browser memory.
  - **Concern Thresholds**: > 3 per tile, > 10 per dashboard.
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.id","dashboard.title","query.count_of_dynamic_fields"],"filters":{"query.count_of_dynamic_fields":">3"},"sorts":["query.count_of_dynamic_fields desc"],"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```

### 2. Unused LookML Components (Last 90 Days)
Clean up debt by removing components that aren't being used.

- **Unused Fields**:
  ```bash
  echo '{"model":"system__activity","view":"field_usage","fields":["field_usage.model","field_usage.explore","field_usage.field"],"filters":{"field_usage.times_used":"0"},"limit":"100"}' | looker-cli api query run_inline_query json - | jq
  ```
- **Unused/Low Usage Explores**:
  ```bash
  echo '{"model":"system__activity","view":"history","fields":["query.model","query.view","history.query_run_count"],"filters":{"history.created_date":"90 days"},"sorts":["history.query_run_count asc"],"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```
- **Unused Joins**: Deduce by finding Views with 0 field usage in an Explore.
  ```bash
  echo '{"model":"system__activity","view":"field_usage","fields":["field_usage.explore","field_usage.view","field_usage.times_used"],"filters":{"field_usage.times_used":"0"},"limit":"100"}' | looker-cli api query run_inline_query json - | jq
  ```

> [!NOTE]
> **Alternative: CLI Vacuum**: You can also use `looker-cli health vacuum explores` or `looker-cli health vacuum models` to identify unused components directly via the CLI. This can be more convenient than custom inline queries, though it may take longer to execute on large instances.

### 3. Pipeline & Cache Health

Ensure data is moving and caching efficiently.

- **Failed Datagroups**:
  ```bash
  looker-cli api datagroup all_datagroups | jq '.[] | select(.trigger_error != null) | {name: .name, model: .model_name, error: .trigger_error}'
  ```
- **Failed PDTs**:
  ```bash
  echo '{"model":"system__activity","view":"pdt_event_log","fields":["pdt_event_log.view_name","pdt_event_log.model_name","pdt_event_log.action","pdt_event_log.error_reason","pdt_event_log.created_time"],"filters":{"pdt_event_log.action":"%error%","pdt_event_log.created_date":"24 hours"},"sorts":["pdt_event_log.created_time desc"]}' | looker-cli api query run_inline_query json - | jq
  ```
### 4. LookML Code Quality & Anti-Patterns
Scan the local LookML repository files for anti-patterns and code smells. Run these commands from the root of the LookML project repository.

- **Wildcard in Includes**: Overly broad includes (e.g., `include: "*.view"`) can slow down validation and clutter the model. Scoped includes (e.g., `include: "/views/**/*.view"`) are preferred.
  ```bash
  grep -rn 'include:\s*"*\*\.view' --include="*.model.lkml" .
  ```

- **Missing Primary Keys**: Views without a primary key can cause incorrect symmetric aggregates and join issues. Every view intended for use as a "one" in a join must have a primary key.
  ```bash
  # List view files that do NOT contain a primary key definition
  find . -name "*.view.lkml" -exec grep -L "primary_key: yes" {} +
  ```

- **Hardcoded Database References**: Avoid hardcoding table or column names. Always use LookML substitution syntax `${TABLE}.column` or `${view_name.field_name}` to allow Looker to handle scoping and aliases correctly.
  ```bash
  # Find sql parameters with dots (likely table.column) but no substitution syntax
  grep -rn 'sql:' --include="*.lkml" . | grep -v '\${' | grep '\.'
  ```

- **Using Number Type for IDs**: Dimension IDs should usually be `type: string` to prevent accidental aggregation (e.g., summing IDs).
  ```bash
  grep -rn "type:\s*number" --include="*.view.lkml" . | grep -i "id"
  ```

- **Repeated Drill Fields (Use Sets)**: Avoid repeating the same list of fields in `drill_fields` across multiple dimensions. Define a `set` and reference it using `drill_fields: [set_name*]`.
  - Scan for identical arrays in `drill_fields` parameters manually or check for lack of `set` usage in views with many drill fields.

### 5. User, Content, and Schedule Management
Audit the instance for inactive assets, orphaned schedules, and schedule hotspots using System Activity.

- **Inactive Accounts (No Login in 90 Days)**:
  ```bash
  echo '{"model":"system__activity","view":"user","fields":["user.id","user.name","user.email","user.last_login_date"],"filters":{"user.last_login_date":"before 90 days ago"},"limit":"100"}' | looker-cli api query run_inline_query json - | jq
  ```

- **Inactive Content (Dashboards Not Queried in 90 Days)**:
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.id","dashboard.title","dashboard.created_date"],"filters":{"history.query_run_count":"0","dashboard.created_date":"before 90 days ago"},"limit":"100"}' | looker-cli api query run_inline_query json - | jq
  ```

- **Schedule Hotspots (Peak Hours)**: Identify hours of the day with excessive scheduled jobs.
  ```bash
  echo '{"model":"system__activity","view":"scheduled_job","fields":["scheduled_job.created_hour_of_day","scheduled_job.count"],"sorts":["scheduled_job.count desc"],"limit":"24"}' | looker-cli api query run_inline_query json - | jq
  ```

- **Failing Schedules**:
  ```bash
  echo '{"model":"system__activity","view":"scheduled_job","fields":["scheduled_job.id","scheduled_plan.name","scheduled_job.status","scheduled_job.status_detail"],"filters":{"scheduled_job.status":"failure"},"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```

- **Orphaned Schedules (Owner is Disabled)**:
  ```bash
  echo '{"model":"system__activity","view":"scheduled_plan","fields":["scheduled_plan.id","scheduled_plan.name","user.name"],"filters":{"user.is_disabled":"Yes"},"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```

> [!NOTE]
> **Definition-Based Analysis via CLI**: You can also use `looker-cli api scheduledplan all_scheduled_plans` to inspect schedule definitions directly. This is useful for analyzing `crontab` patterns to detect hotspots or checking `user_id` to identify potentially orphaned schedules (cross-reference with disabled users). However, to check **historical failures or execution history**, you MUST use the System Activity `scheduled_job` queries above.

## 🛡️ Best Practices & Gotchas




- **Piping JSON**: Always use `echo '...' | looker-cli ... json -` to pass inline JSON to `run_inline_query`. Do NOT pass the JSON string directly as an argument, or you will get a "file name too long" error.
- **Filter Wildcards**: Use `%` (SQL style) instead of `*` for string filters in `run_inline_query` on System Activity (e.g., `%error%`).
- **Timeframes**: The `history` table is time-bound. A field or explore showing as "unused" in 90 days might still be used for annual reporting. Verify before deleting.
- **Hidden Fields**: `hidden: yes` fields might still be used in joins or calculations. Do not delete without checking dependencies.
- **Unused Fields**: fields identified as not used might still be used in other explores via hub & spoke model. Check if the fields are being referenced via `extends` or `refinements` in spoke repositories.

## 📚 References

- **Looker Performance Audit Guide**: [performance_audit_guide.md](references/performance_audit_guide.md)

