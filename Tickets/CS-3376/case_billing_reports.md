
1. **Report selection template**  
   [selectreport.html:1413](/home/projectds/projects/compass/usac/usac/templates/usac/mat-admin/selectreport.html:1413) contains links to:
   ```text
   /view-report/DISPATCH_MODULE_BILLING
   /view-report/DISPATCH_MODULE_BILLING_ACCOUNTING
   ```

2. **Report page routing**  
   [urls.py](/home/projectds/projects/compass/usac/usac/urls.py) maps `/view-report/<report_name>` to the report view. It also maps `/report/type/<report_id>/` to `generate_report()` for generating output.

3. **Report implementation mapping**  
   [reports_managment.py:2597](/home/projectds/projects/compass/usac/usac/code/business/reports_managment.py:2597) maps those report IDs to:
   ```text
   payments/dispatch_module_billing.sql
   payments/dispatch_module_billing_accounting.sql
   ```

4. **Connection to case management**  
   The SQL queries read `Case`, `CaseService`, `CaseVendor`, and `ServiceProvider`, including case statuses, dispatch details, service fees, and mileage:
   
   - [dispatch_module_billing.sql](/home/projectds/projects/compass/usac/data/reports/_queries/payments/dispatch_module_billing.sql)
   - [dispatch_module_billing_accounting.sql](/home/projectds/projects/compass/usac/data/reports/_queries/payments/dispatch_module_billing_accounting.sql)



   **No—many reports in `selectreport.html` concern cases.** I highlighted those two specifically because you asked about **billing after case completion**.

Case-related reports include:

| Area | Reports |
|---|---|
| General case details | Cases (Basic), Cases (Detailed), Cases by State, Cases by Client |
| Dispatch activity | Cases 60min Dispatch, Dispatch Events, Digital Dispatch Status Update |
| Case billing | Dispatch Module Billing, Dispatch Module Billing Accounting, and their Roadway-specific versions |
| Payment-related cases | CashCall/OnDemand Cases, Credit Card Link Cases |
| Specialized cases | ODCC Incident Cases, Cases with RepairPal Locations, External Tow to Pepboys, Internal Pepboys API Case Usage |
| Customer feedback | Case Survey reports |

There are also claims and service-provider payment reports, which relate to different parts of the case/payment workflow.

