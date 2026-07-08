---
name: looker-audit
description: >-
  Audits a Looker instance for performance, governance, and cleanup opportunities.
  Use when analyzing dashboard performance, identifying unused LookML components,
  checking for excessive table calculations, or auditing datagroup health.
  Don't use for general Looker onboarding or creating new dashboards.
---

# Looker Audit & Health Check

This skill guides you through auditing a Looker instance using `looker-cli` and System Activity to identify performance bottlenecks, governance gaps, and cleanup opportunities. It flags the violation of best practices and suggests optimal fixes e.g. code snippets or step-by-step guide. 

**IMPORTANT**: When you suggest code snippets or step-by-step guide to update in LookML, you MUST include the exact URL pointing to the specific file and line number(s) of the violation. Construct the full link carefully using the file path, repository URL. Append the line suffix (e.g. #L16 or #L16-L19) to the URL based on the line numbers discovered in the tool output.

> [IMPORTANT]
> ## ⚠️ Rules of Engagement (CRITICAL)
> 1. **System Activity First**: For all operational metrics (Performance, Errors, Usage, Dynamic Fields, Datagroup/PDT operational status), you **MUST** query System Activity or use `looker-cli health` commands.
> 2. **No Operational File Scans**: Do **NOT** read LookML files to deduce operational state or verify usage history.
> 3. **Skip Filesystem Grounding**: You are **ALREADY** in the correct environment. Do **NOT** search the filesystem (e.g., `find_by_name`, `ls -R`, or `grep_search`) to locate Looker configuration files, projects, or credentials. Assume `looker-cli` is pre-authenticated and ready.
> 4. **Immediate Action**: Begin your audit **immediately** using `looker-cli` or API queries. Do not waste turns exploring directory structures or searching for "Looker" keywords in the workspace.


# 🔁 Execution Workflow

Follow this three-tiered audit strategy to move from symptoms to structural root causes: 

1. **High-level Health audits**: Use Pulse/Vacuum commands for a broad, fast initial assessment (e.g., "Is the instance healthy? What are the top 5 slow things?").
2. **Drill-down review**: Use System Activity Inline Queries (refer to audit checklist below) for deep drill-downs, specific timeframes (e.g., 90 days vs 7 days), or custom filtering (e.g., filtering by a specific user or custom thresholds).
3. **Root-Cause Telemetry over Static Metrics**: Do not simply flag an issue. When a bottleneck is identified, immediately trace the root cause by cross-referencing relevant areas and identify what is causing latencies or failures (e.g. tile bloat, un-cached Explores, or missing required partition filters) to help guide developers to fix the underlying LookML.



## ⚡ High-level Health Audits
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
- **Database Connections**: Verify connection status and query volume. Look for high latency (> 5 seconds), large result sets (> 1M rows), or stale tests.
  ```bash
  looker-cli health pulse db-connections
  ```


## 📋 Audit Checklist

### 1. Dashboard Health & Complexity
Excessive complexity on dashboards leads to browser lag and poor user experience. Look for dashboards with > 20 tiles or average query times > 30 seconds.

- **Auto-Refresh**: Identify dashboards running unnecessary constant queries.
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.id","dashboard.title","dashboard.refresh_interval"],"filters":{"dashboard.refresh_interval":"-NULL"},"limit":"100"}' | looker-cli api query run_inline_query json - | jq
  ```
- **Dynamic Fields**: Excessive dynamic fields (Table Calculations, Custom Fields) bypass LookML governance and strain browser memory.
  - **CRITICAL**: Do NOT scan LookML files for these; they are defined at the dashboard/tile level. You **MUST** use this **System Activity Inline Query** exclusively.
  - **Concern Thresholds**: > 3 per tile, > 10 per dashboard.
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.id","dashboard.title","query.count_of_dynamic_fields"],"filters":{"query.count_of_dynamic_fields":">3"},"sorts":["query.count_of_dynamic_fields desc"],"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```
- **Merged Queries**: Merged queries bypass LookML joins and can be slow. Identify usage in the last 30 days.
  ```bash
  echo '{"model":"system__activity","view":"history","fields":["history.created_time","user.name","result_maker.merge_query_id","dashboard.title"],"filters":{"result_maker.is_merge_query":"Yes","history.created_date":"30 days"},"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```
- **Dashboard Average Runtime**: Drill down into average runtime beyond the Pulse command.
  ```bash
  echo '{"model":"system__activity","view":"dashboard_performance","fields":["dashboard.title","dashboard_history_stats.avg_runtime"],"sorts":["dashboard_history_stats.avg_runtime desc"],"limit":"10"}' | looker-cli api query run_inline_query json - | jq
  ```

### 2. Unused LookML Components (Last 90 Days)
Clean up debt by removing components that aren't being used. Consider Explores/Fields with 0 usage, or Models with very low usage (< 10 queries).

- **Workflow**: Rely **exclusively** on System Activity for identifying usage. Do **NOT** scan LookML files to verify usage history.

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

Ensure data is moving and caching efficiently. Check for stale data (models where max build time is older than DB's max update time).

- **Failed Datagroups**:
  - **Workflow**: First, check the operational status via the API. If failures are found, **then** inspect the LookML definition to suggest fixes. Do NOT scan LookML files to find failures.
  ```bash
  looker-cli api datagroup all_datagroups | jq '.[] | select(.trigger_error != null) | {name: .name, model: .model_name, error: .trigger_error}'
  ```
- **Failed PDTs**:
  - **Workflow**: Check the **operational status** via System Activity first. Only inspect LookML definitions when specific failures are identified to suggest fixes (e.g., syntax errors in SQL).
  ```bash
  echo '{"model":"system__activity","view":"pdt_event_log","fields":["pdt_event_log.view_name","pdt_event_log.model_name","pdt_event_log.action","pdt_event_log.error_reason","pdt_event_log.created_time"],"filters":{"pdt_event_log.action":"%error%","pdt_event_log.created_date":"24 hours"},"sorts":["pdt_event_log.created_time desc"]}' | looker-cli api query run_inline_query json - | jq
  ```
- **Cache Efficiency (Cache Hits vs Live Queries)**: Check the ratio of cache hits. Target > 50% for standard dashboards.
  ```bash
  echo '{"model":"system__activity","view":"history","fields":["history.result_source","history.query_run_count"],"pivots":["history.result_source"],"filters":{"history.result_source":"-NULL","history.created_date":"30 days"},"sorts":["history.result_source"],"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```

### 4. User, Content, and Schedule Management
Audit the instance for inactive assets, orphaned schedules, and schedule hotspots using System Activity. Look for high-volume schedules (> 1/hr) or stale schedules (not run in 30 days).

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
- **Holistic Review**: You must analyze the instance or repositories as a whole to provide a holistic review.
- **Anti-Hallucination Guardrail**: NEVER hallucinate or assume data. Always run the query first and report only the actual results. If no data matches, report that explicitly.

## 📚 References

- **Looker Performance Audit Guide**: [performance_audit_guide.md](references/performance_audit_guide.md)

