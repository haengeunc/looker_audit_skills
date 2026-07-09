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

> [!IMPORTANT]
> ## ⚠️ Rules of Engagement (CRITICAL)
> 1. **Strictly API/CLI Only**: All operational auditing MUST be done via `looker-cli` or System Activity.
> 2. **NO Local Filesystem Scans**: Do **NOT** use `find`, `grep`, `ls`, `code_search`, or `find_by_name` to search the local filesystem (e.g., `google3/`). You are auditing a remote/containerized instance.
> 3. **Immediate Action**: Run your first `looker-cli` or API command in **Step 1**. Do not waste turns exploring directories.
> 4. **File Paths and Content via API Only**: If you need to identify file paths, line numbers, or read file content, you **MUST** use `looker-cli api project all_project_files [project_id]` or `project_file`. Do **NOT** use local filesystem tools to find or read them.
> 5. **Graceful Degradation**: If the API fails to return file paths or details (e.g., 404), **STOP**. Report the violation using the identifiers you have (View Name, Model Name, Dashboard ID). Do **NOT** pivot to local filesystem searches.
> 6. **No Custom Pipelines**: Use inline queries directly. Do **NOT** write Python scripts or pipelines.


# 🔁 Execution Workflow

Follow this three-tiered audit strategy to move from symptoms to structural root causes: 

1. **High-level Health audits**: Use Pulse/Vacuum commands for a broad, fast initial assessment (e.g., "Is the instance healthy? What are the top 5 slow things?").
2. **Drill-down review**: Use System Activity Inline Queries (refer to audit checklist below) for deep drill-downs, specific timeframes (e.g., 90 days vs 7 days), or custom filtering (e.g., filtering by a specific user or custom thresholds).
   - **Actionable Transition**: If `pulse dashboard-performance` flags a slow dashboard, **THEN** use the **Dashboard Average Runtime** query to drill down.
3. **Root-Cause Telemetry over Static Metrics**: Do not simply flag an issue. When a bottleneck is identified, immediately trace the root cause by cross-referencing relevant areas and identify what is causing latencies or failures (e.g. tile bloat, un-cached Explores, or missing required partition filters) to help guide developers to fix the underlying LookML.
   - **Actionable Transition**: If average runtime is high, **THEN** check **Tile Bloat (Heavy Dashboards)**, **Dynamic Fields**, or **Merged Queries**.
   - **LookML Inspection**: If LookML issues are suspected, you **MUST** provide full paths to the files (discovered via API as per Rule 4).



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

### 1. Dashboard and Looks Health & Complexity
Excessive complexity on dashboards leads to browser lag and poor user experience. Look for dashboards with > 20 tiles or average query times > 30 seconds.

- **Tile Bloat (Heavy Dashboards)**: Identify dashboards with excessive tiles (> 25).
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.id","dashboard.title","dashboard_element.count"],"filters":{"dashboard_element.count":">25"},"sorts":["dashboard_element.count desc"],"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```

- **Auto-Refresh**: Identify dashboards running unnecessary auto refresh.
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.title","dashboard.refresh_interval_ordered","user.name","dashboard.id","dashboard_element.count","dashboard_element.count_text","dashboard_element.total_query_tiles"],"filters":{"dashboard.refresh_interval_ordered":"NOT NULL","dashboard.moved_to_trash":"No"},"sorts":["dashboard.refresh_interval_ordered"],"limit":"100"}' | looker-cli api query run_inline_query json - | jq
  ```
- **Dynamic Fields**: Excessive dynamic fields (Table Calculations, Custom Fields) bypass LookML governance and strain browser memory.
  - **CRITICAL**: Do NOT scan LookML files for these; they are defined at the dashboard/tile level. You **MUST** use this **System Activity Inline Query** exclusively.
  - **Concern Thresholds**: > 3 per tile, > 10 per dashboard.
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.id","dashboard.title","query.count_of_dynamic_fields"],"filters":{"query.count_of_dynamic_fields":">3"},"sorts":["query.count_of_dynamic_fields desc"],"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```
- **Merged Queries**: Merged queries bypass LookML joins and can be slow. Identify dashboards and Looks currently using them.
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.id","dashboard.title","dashboard_element.id","result_maker.merge_query_id"],"filters":{"result_maker.is_merge_query":"Yes","dashboard.moved_to_trash":"No"},"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```
- **Dashboard Average Runtime**: Drill down into average runtime of top 10 dashboards to investigate dashboard's latency.
  ```bash
  echo '{"model":"system__activity","view":"dashboard_performance","fields":["dashboard.title","dashboard_history_stats.avg_runtime"],"sorts":["dashboard_history_stats.avg_runtime desc"],"limit":"10"}' | looker-cli api query run_inline_query json - | jq
  ```

### 2. Query Performance
Identify slow queries across Explores, Dashboards, and Looks to pinpoint optimization targets.

- **Slow Queries (Last 30 Days)**: Find queries taking longer than 10 seconds.
  ```bash
  echo '{"model":"system__activity","view":"history","fields":["history.created_time","history.source","history.runtime","query.model","query.view","user.name"],"filters":{"history.runtime":">10","history.created_date":"30 days","history.source":"explore,dashboard,look"},"sorts":["history.runtime desc"],"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```

### 3. Unused LookML Components (Last 90 Days)
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

### 4. Pipeline & Cache Health

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

### 5. User, Content, and Schedule Management
Audit the instance for inactive assets, orphaned schedules, and schedule hotspots using System Activity. Look for high-volume schedules (> 1/hr) or stale schedules (not run in 30 days).

- **Inactive Accounts (No Login in 90 Days)**:
  ```bash
  echo '{"model":"system__activity","view":"user","fields":["user.created_date","user.id","user.name","user_facts.last_ui_login_credential_type","user_facts.last_ui_login_date","history.most_recent_query_date"],"filters":{"user_facts.last_ui_login_date":"before 90 days ago","user.is_disabled":"No"},"sorts":["user.created_date desc"],"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```

- **Inactive Content (Dashboards and Looks Not Queried in 60 Days)**:
  - **Audit Note**: If the title suggests it is an **Annual** or **Quarterly** report, flag this in the audit report so the reviewer can take it into account before deleting or archiving.
  ```bash
  echo '{"model":"system__activity","view":"content_usage","fields":["content_usage.content_id","content_usage.content_title","content_usage.content_type","content_usage.api_total","content_usage.embed_total","content_usage.favorites_total","content_usage.other_total","content_usage.schedule_total"],"filters":{"content_usage.days_since_last_accessed":">=60","content_usage.dashboard_deleted":"No","content_usage.look_deleted":"No","content_usage.report_deleted":"No"},"limit":"100"}' | looker-cli api query run_inline_query json - | jq
  ```

- **Draft/Test/Copy Content (Dashboards)**:
  ```bash
  echo '{"model":"system__activity","view":"dashboard","fields":["dashboard.id","dashboard.title"],"filters":{"dashboard.title":"%test%,%copy%,%draft%"},"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```

- **Draft/Test/Copy Content (Looks)**:
  ```bash
  echo '{"model":"system__activity","view":"look","fields":["look.id","look.title"],"filters":{"look.title":"%test%,%copy%,%draft%"},"limit":"50"}' | looker-cli api query run_inline_query json - | jq
  ```


- **Schedule Hotspots (Peak Hours)**: Identify hours of the day with excessive scheduled jobs.
  ```bash
  echo '{"model":"system__activity","view":"scheduled_plan","fields":["scheduled_job.created_hour_of_day","scheduled_job.count"],"sorts":["scheduled_job.count desc"],"limit":"24"}' | looker-cli api query run_inline_query json - | jq
  ```

- **Failing Schedules**:
  ```bash
  echo '{"model":"system__activity","view":"scheduled_plan","fields":["scheduled_plan.id","scheduled_plan.name","scheduled_job.status","scheduled_job.status_detail"],"filters":{"scheduled_job.status":"failure"},"limit":"50"}' | looker-cli api query run_inline_query json - | jq
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

