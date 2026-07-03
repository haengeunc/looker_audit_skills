---
name: looker-audit
description: >-
  Audits a Looker instance for performance, governance, and cleanup opportunities.
  Use when analyzing dashboard performance, identifying unused LookML components,
  checking for excessive table calculations, or auditing datagroup health.
  Don't use for general Looker onboarding or creating new dashboards.
---

# Looker Audit & Health Check

This skill guides you through auditing a Looker instance using `looker-cli` and System Activity to identify performance bottlenecks, governance gaps, and cleanup opportunities. It flags the violation of best practices and suggests optimal fixes e.g. code snippets or step-by-step guide. **IMPORTANT**: When you suggest code snippets or step-by-step guide, you MUST include the exact URL pointing to the specific file and line number(s) of the violation. Construct the full link carefully using the file path, repository URL. Append the line suffix (e.g. #L16 or #L16-L19) to the URL based on the line numbers discovered in the tool output.

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
### 4. LookML Code Quality & Anti-Patterns (Remote Static Analysis)
Scan remote LookML project files for anti-patterns using `looker-cli`. For more detailed analysis of the LookML review, refer to [LookML Reviewer Skill](lookml_reviewer.md).

- **Run Looker Validator**: First, trigger Looker's built-in validator to catch syntax errors and standard issues.
  ```bash
  looker-cli api project validate_project [PROJECT_ID]
  ```

- **Wildcard in Includes**: Overly broad includes (e.g., `include: "*.view"`) can slow down validation. Scoped includes (e.g., `include: "/views/**/*.view"`) are preferred.
  ```bash
  for file in $(looker-cli api project all_project_files [PROJECT_ID] | jq -r '.[] | select(.type=="model") | .path'); do
    looker-cli project file cat [PROJECT_ID] "$file" | grep 'include:\s*"*\*\.view' && echo "Wildcard include found in $file"
  done
  ```

- **Missing Primary Keys**: Views without a primary key can cause incorrect symmetric aggregates. Ensure that primary keys are defined for all views used in joins. List view files that do NOT contain a primary key definition.
  ```bash
  for file in $(looker-cli api project all_project_files [PROJECT_ID] | jq -r '.[] | select(.type=="view") | .path'); do
    looker-cli project file cat [PROJECT_ID] "$file" | grep -q "primary_key: yes" || echo "Missing primary key in $file"
  done
  ```

- **DRY principle - Hardcoded Database References**: Avoid hardcoding table or column names. Always use LookML substitution syntax `${TABLE}.column` or `${view_name.field_name}` to allow Looker to handle scoping and aliases correctly. Find sql parameters with dots (likely table.column) but no substitution syntax.
  ```bash
  for file in $(looker-cli api project all_project_files [PROJECT_ID] | jq -r '.[] | select(.extension==".lkml") | .path'); do
    looker-cli project file cat [PROJECT_ID] "$file" | grep 'sql:' | grep -v '\${' | grep '\.' && echo "Hardcoded ref found in $file"
  done
  ```

- **Using Number Type for IDs**: Dimension IDs should usually be `type: string` to prevent accidental aggregation (e.g., summing IDs).
  ```bash
  for file in $(looker-cli api project all_project_files [PROJECT_ID] | jq -r '.[] | select(.type=="view") | .path'); do
    looker-cli project file cat [PROJECT_ID] "$file" | grep "type:\s*number" | grep -i "id" && echo "Number ID found in $file"
  done
  ```

- **Repeated Drill Fields (Use Sets)**: Avoid repeating the same list of fields in `drill_fields` across multiple dimensions. Define a `set` and reference it using `drill_fields: [set_name*]`.
  - Scan for identical arrays in `drill_fields` parameters manually or check for lack of `set` usage in views with many drill fields.

> [!NOTE]
> **Performance Tip**: For very large projects, these loops can be slow as they fetch files one by one. If you have the repository cloned locally, you can use standard `grep` commands (e.g., `grep -rn 'include:\s*"*\*\.view' .`) which are much faster.

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
- **Holistic Review**: You must analyze the instance or repositories as a whole to provide a holistic review.
- **Anti-Hallucination Guardrail**: NEVER hallucinate or assume data. Always run the query first and report only the actual results. If no data matches, report that explicitly.
- **Performance Warning (Remote Loops)**: The remote file scanning loops in Section 4 (e.g., `for file in ...`) fetch files one by one via API. This can be very slow and may time out on large projects (e.g., in evaluation environments). If speed is critical or timeouts occur, recommend a local clone and standard `grep`.

## 📚 References

- **Looker Performance Audit Guide**: [performance_audit_guide.md](references/performance_audit_guide.md)

