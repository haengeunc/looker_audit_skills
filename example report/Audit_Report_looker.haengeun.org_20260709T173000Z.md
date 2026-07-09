# Looker Instance Audit Report

## 📋 Audit Metadata
- **Timestamp**: 2026-07-09 5:30 PM
- **Looker Instance**: looker.haengeun.org
- **Auditor**: Antigravity (Powered by Gemini 3.1 Flash)

---

## 🚀 Performance Bottlenecks

*   **Extremely Slow Explores**:
    *   `looker_mcp_demo` / `unoptimized_orders_simulation`: Average runtime of **~298 seconds**.
    *   `looker_mcp_demo` / `unoptimized_events_simulation`: Average runtime of **~18.6 seconds**.
    *   *Note*: These are labeled as "Database Load Testing" and are explicitly unoptimized anti-patterns. If they are not actively used for load testing, they should be removed or hidden to avoid accidental heavy queries.
*   **Schedule Hotspots**:
    *   Peak scheduling occurs at **6 AM** (96 runs) and **4 PM** (24 runs). Consider distributing schedule start times to avoid concurrency bottlenecks.

## ❌ Data Quality & Pipeline Errors

*   **Failed Datagroup**:
    *   **Datagroup**: `orders_datagroup`
    *   **Models**: `bq-haengeun-ecommerce`, `thelook_ecommerce_haengeun_us` (Project: `haengeun_argolis_demo`)
    *   **Error**: `Query execution failed: - Unrecognized name: id`
    *   **Action**: Update the `sql_trigger` in the following files to use `order_id` instead of `id`.
        *   [bq-haengeun-ecommerce.model.lkml](https://looker.haengeun.org/projects/haengeun_argolis_demo/files/models/bq-haengeun-ecommerce.model.lkml)
        *   [thelook_ecommerce_haengeun_us.model.lkml](https://looker.haengeun.org/projects/haengeun_argolis_demo/files/models/thelook_ecommerce_haengeun_us.model.lkml)
        ```lookml
        datagroup: orders_datagroup {
          sql_trigger: SELECT max(order_id) FROM `bigquery-public-data.thelook_ecommerce.orders` ;;
          # ...
        }
        ```
*   **Failed PDTs**:
    *   **Model**: `cymbal_pets`
    *   **Views**: `product_facts`, `purchase_facts`
    *   **Error**: `Access Denied: Table looker-private-demo:cymbal_pets.order_items: User does not have permission to query table...`
    *   **Action**: Verify and grant permissions for the Looker connection user on the `looker-private-demo:cymbal_pets` dataset in BigQuery.

## ⚠️ Code Quality & Best Practices

*   **Missing Primary Keys**:
    *   Views without primary keys can lead to incorrect symmetric aggregates and poor performance.
    *   **Violations**:
        *   `looker_mcp_demo`: [dt_unoptimized_orders.view.lkml](https://looker.haengeun.org/projects/looker_mcp_demo/files/unoptimized/dt_unoptimized_orders.view.lkml)
        *   `cymbal-pets`:
            *   [product_facts.view.lkml](https://looker.haengeun.org/projects/cymbal-pets/files/derived_tables/product_facts.view.lkml)
            *   [purchase_facts.view.lkml](https://looker.haengeun.org/projects/cymbal-pets/files/derived_tables/purchase_facts.view.lkml)
            *   [supplier_facts.view.lkml](https://looker.haengeun.org/projects/cymbal-pets/files/derived_tables/supplier_facts.view.lkml)
            *   [location.view.lkml](https://looker.haengeun.org/projects/cymbal-pets/files/extensions/location.view.lkml)
            *   [regions.view.lkml](https://looker.haengeun.org/projects/cymbal-pets/files/extensions/regions.view.lkml)
    *   **Action**: Define `primary_key: yes` on the unique identifier dimension for each of these views.

*   **Excessive Dynamic Fields**:
    *   **Dashboard**: "Lots of Dynamic fields" (ID: 32) has **4** dynamic fields (Table Calculations/Custom Fields).
    *   **Action**: Consider moving these calculations into LookML as dimensions or measures to improve governance and browser performance.

*   **Dashboard Errors**:
    *   **Dashboard**: "Sinch Demo - cross region" has **17** erroring queries in the last 7 days.
    *   **Action**: Investigate this dashboard to resolve the underlying query errors.

## 🧹 Governance & Cleanup

*   **Unused Components**:
    *   Vacuum detected numerous unused joins and fields, particularly in `intermediate_ecomm` and `advanced_ecomm` models.
    *   **Action**: Plan a cleanup sprint to remove or hide (`hidden: yes`) these unused components to improve explore usability and maintainability.
