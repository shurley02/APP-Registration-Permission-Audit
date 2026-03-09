# Invoke-AppPermissionAudit

A PowerShell script that audits Entra ID app registrations against actual Microsoft Graph API usage logged in Azure Log Analytics, then generates color-coded HTML reports highlighting over-privileged and never-used permissions.

---

## Executive Summary

### What It Does

This script answers a simple question: *does each app registration in your tenant only have the permissions it actually uses?*

It pulls the list of granted permissions from the Graph API, queries your Log Analytics workspace for observed API calls over a configurable lookback window, correlates the two, and produces an HTML report identifying permissions that have never been exercised. The result is an actionable least-privilege audit you can run on a single app, a list of apps, or your entire tenant at once.

### Licensing Requirements

| Requirement | Details |
|---|---|
| **Entra ID** | P1 or P2 license required to enable `MicrosoftGraphActivityLogs` in Diagnostic Settings |
| **Log Analytics Workspace** | Any Azure subscription — no premium tier required |
| **App Registration** | Free — one auditor app registration with two API permissions |

> **Note:** A free Azure account is sufficient for the Log Analytics workspace. The only hard licensing requirement is Entra ID P1 or higher, which is included in Microsoft 365 E3/E5, EMS E3/E5, or available as a standalone add-on (~$6/user/month).

### Data & Cost Considerations

Log Analytics pricing is based on data ingestion volume. `MicrosoftGraphActivityLogs` are the primary cost driver.

| Log Type | Typical Volume | Estimated Cost |
|---|---|---|
| MicrosoftGraphActivityLogs | 0.5–5 GB/day (varies by tenant activity) | ~$2.30/GB ingested (Pay-As-You-Go) |
| AuditLogs / SignInLogs | 0.1–1 GB/day | ~$2.30/GB ingested |
| **Retention** | First 31 days free | $0.10/GB/month after 31 days |

For most small-to-mid-sized tenants, total monthly Log Analytics cost for this solution runs **$20–$100/month**. Large enterprise tenants with high API call volume should evaluate a Commitment Tier (100 GB/day+) for significant per-GB discounts.

There is no additional cost to run this script itself — it only reads from existing Log Analytics data.

---

## Technical Setup

### Prerequisites

- PowerShell 5.1 or 7+
- An Azure subscription with a Log Analytics workspace
- Entra ID P1 or higher

---

### Step 1 — Enable Diagnostic Logging in Entra ID

1. Sign in to the [Azure Portal](https://portal.azure.com) and navigate to **Microsoft Entra ID → Monitoring → Diagnostic settings**.
2. Click **+ Add diagnostic setting**.
3. Give it a name (e.g., `EntraLogs-to-LogAnalytics`).
4. Under **Logs**, check at minimum:
   - `MicrosoftGraphActivityLogs` *(required)*
   - `AuditLogs` *(recommended)*
   - `SignInLogs` *(recommended)*
5. Under **Destination**, check **Send to Log Analytics workspace** and select your workspace.
6. Click **Save**.

> **Allow 1–2 hours** for logs to begin appearing in the workspace after initial setup.

To verify, open your Log Analytics workspace, go to **Logs**, and run:

```kql
MicrosoftGraphActivityLogs
| take 10
```

If results appear, logging is working.

---

### Step 2 — Create the Auditor App Registration

This is the identity the script uses to read app registrations and query Log Analytics. It needs two API permissions and one RBAC role.

#### 2a. Register the App

1. Navigate to **Microsoft Entra ID → App registrations → + New registration**.
2. Name it something like `apipermission-auditor`.
3. Leave the redirect URI blank and click **Register**.
4. Note the **Application (client) ID** and **Directory (tenant) ID** from the Overview page.

#### 2b. Add API Permissions

Navigate to **API permissions → + Add a permission**.

Add the following **Application permissions** (not Delegated):

| API | Permission | Purpose |
|---|---|---|
| Microsoft Graph | `Application.Read.All` | Read app registrations and service principals |
| Log Analytics API | `Data.Read` | Query MicrosoftGraphActivityLogs |

For **Log Analytics API**, click **APIs my organization uses**, search for `Log Analytics`, and select `Data.Read`.

After adding both permissions, click **Grant admin consent for [your tenant]** and confirm.

#### 2c. Assign the RBAC Role

The app also needs read access to the Log Analytics workspace itself.

1. Navigate to your **Log Analytics workspace → Access control (IAM) → + Add role assignment**.
2. Select the **Log Analytics Reader** role.
3. Assign it to your `apipermission-auditor` app registration.
4. Click **Save**.

#### 2d. Create a Client Secret

1. Back in the app registration, go to **Certificates & secrets → + New client secret**.
2. Set a description and expiration, then click **Add**.
3. **Copy the secret value immediately** — it will not be shown again.

---

### Step 3 — Configure the Script

Open `Invoke-AppPermissionAudit.ps1` and update the four variables at the top of the `param` block:

```powershell
[string]$TenantId     = "your-tenant-id",
[string]$ClientId     = "your-auditor-app-client-id",
[string]$ClientSecret = "your-client-secret-value",
[string]$WorkspaceId  = "your-log-analytics-workspace-id",
```

Your **Workspace ID** is found in the Log Analytics workspace under **Settings → Properties**.

---

## Usage

```powershell
# Audit a single app
.\Invoke-AppPermissionAudit.ps1 -TargetAppId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Audit multiple apps by ID
.\Invoke-AppPermissionAudit.ps1 -AppIdList "id1","id2","id3"

# Audit from a CSV file (requires an AppId column header)
.\Invoke-AppPermissionAudit.ps1 -InputFile "apps.csv"

# Audit from a plain text file (one App ID per line)
.\Invoke-AppPermissionAudit.ps1 -InputFile "appids.txt"

# Auto-discover and audit all admin-managed app registrations in the tenant
.\Invoke-AppPermissionAudit.ps1 -DiscoverAll

# Adjust the lookback window (default: 90 days)
.\Invoke-AppPermissionAudit.ps1 -DiscoverAll -LookbackDays 30

# Set a custom output folder
.\Invoke-AppPermissionAudit.ps1 -DiscoverAll -OutputFolder "/Users/me/Desktop/AuditReports"

# Generate only the summary report (skip individual per-app reports)
.\Invoke-AppPermissionAudit.ps1 -DiscoverAll -SkipIndividualReports
```

Input modes can be combined — they deduplicate automatically.

---

## Output

Running against multiple apps produces a folder of HTML files:

- **`AuditSummary.html`** — tenant-wide roll-up: all apps ranked by unused permission count, overall risk posture, and a cross-app table of every never-used permission.
- **`AppAudit_<AppName>.html`** — individual per-app report showing each permission, call count, last used date, failed call count, and least-privilege recommendations.

Reports open automatically in the default browser when the script completes.

---

## Auto-Discovery Behavior

When `-DiscoverAll` is used, the script calls `GET /applications` to enumerate all app registrations in the tenant, then excludes:

- Well-known Microsoft first-party apps (Graph, Exchange, Teams, SharePoint, etc.)
- Apps with no permissions configured

What remains is every app registration that an admin has created or managed and assigned at least one API permission to.

> **Performance note:** Auto-discovery works well for small-to-medium tenants. On tenants with thousands of app registrations, expect longer runtimes as each app requires individual Graph and Log Analytics API calls. A batched/parallel mode for large-scale tenants is a planned enhancement.

---

## Security Notes

- Store the client secret in a secrets manager (Azure Key Vault, a CI/CD secret store, etc.) rather than hardcoding it in the script for production use.
- Rotate the client secret on a regular schedule and whenever personnel with access to it change.
- The auditor app only requires **read** permissions — it cannot modify any app registrations or data.

---

## License

MIT
