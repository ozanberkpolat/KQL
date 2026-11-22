# KQL

A library of Kusto (KQL) queries for day-to-day Azure operations: Azure Resource Graph (ARG) inventories,
Log Analytics reports, Microsoft Defender Advanced Hunting and Microsoft Sentinel / firewall log analysis.
Every query is generic: parameters sit in `let` statements at the top with placeholder values, and each file
starts with a header (title, purpose, source tables, parameters to edit, notes and traps).

The ARG posture and hardening checks (findings such as "storage accounts allowing shared key" or "Key Vaults without purge
protection") are NOT repeated here. They live in [azposture](https://github.com/ozanberkpolat/azposture).
The posture inventories in this repo (`storage-account-posture.kql`, `keyvault-posture.kql`) show the full
configuration instead of a single finding.

## Layout

| Folder | Runs in | Content |
|---|---|---|
| `ResourceGraph/<Area>/` | Azure Resource Graph | Compute, Networking, PrivateEndpoint, Storage, Backup-ASR, UpdateManager, Monitoring, Governance-Tags, Defender, Cost, AVD, RBAC, LogicApps, KeyVault-AppService |
| `LogAnalytics/<Area>/` | Log Analytics workspace | Heartbeat, VMInsights, AzureBackup, TrafficAnalytics, StorageLogs, AzureActivity, SQL, Audit |
| `Defender/` | Defender portal Advanced Hunting, and Log Analytics for the exported tables | vulnerable software, software evidence paths, App Control (WDAC) hunting, recommendation prioritisation and state changes |
| `Sentinel/` | Sentinel / Log Analytics | FortiGate syslog parsing (generic field extraction), Azure network change register |

## How to run

Each file holds one question. Files with several numbered queries (`--- Query 2`) contain alternates or a
family of related lists: run them one at a time, the later ones are commented out.

### Azure Resource Graph

```bash
az extension add --name resource-graph          # once
az login
az graph query -q "$(cat ResourceGraph/Compute/vm-inventory.kql)" --first 1000 -o table
```

`az graph query` returns at most 1000 rows per call. Page with `--skip 1000`, or pass `--management-groups <id>`
(or `--subscriptions`) to widen the scope, the default is the subscriptions of the current context.
Comment lines (`//`) are fine inside the file.

```powershell
Connect-AzAccount
$q = Get-Content .\ResourceGraph\Compute\vm-inventory.kql -Raw
Search-AzGraph -Query $q -First 1000                       # current subscriptions
Search-AzGraph -Query $q -UseTenantScope -First 1000       # whole tenant, needs read access on the scopes
```

The Azure portal "Resource Graph Explorer" also works, paste the file as is.

### Log Analytics

```bash
# 1. find the workspace GUID (customerId), not the resource id
az monitor log-analytics workspace list --query "[].{name:name,id:customerId,rg:resourceGroup}" -o table
# 2. run, ALWAYS with --timespan
az monitor log-analytics query -w <customerId> --analytics-query "$(cat LogAnalytics/Heartbeat/heartbeat-last-seen-os.kql)" --timespan P30D -o table
```

Or open the workspace in the Azure portal, Logs, and paste the file. The `az monitor log-analytics` command needs
`az extension add --name log-analytics`. Reader on the workspace (Log Analytics Reader) is enough for queries.
`ResourceGraph/Monitoring/log-analytics-workspaces.kql` lists the workspace GUIDs.

### Defender Advanced Hunting

Paste the files from `Defender/` that use `Device*` tables into security.microsoft.com, Hunting, Advanced hunting.
Those tables are not in Log Analytics unless you stream them. `SecurityRecommendation`, `SecureScores` and
`SecurityIncident` are Log Analytics tables (continuous export / Sentinel).

## Traps

Hard-won lessons collected from the queries. They are also repeated in the file headers where they apply.

### Running queries

- `az monitor log-analytics query` returns 0 rows without `--timespan` and does not tell you why.
  Always pass `--timespan P30D` (or an interval) that covers the window of your query. The `ago()` /
  `datetime()` filter inside the query does not replace it.
- An empty result is a finding to investigate, not an answer. Check that the table is fed (diagnostic setting,
  agent, DCR), that the window is right and that the name filter matches before reporting "nothing found".
- `az graph query` needs the `resource-graph` extension, returns 1000 rows at most per call and
  only covers the subscriptions of the current context unless you set a scope.
- On Windows, `az.cmd` mangles parentheses and special characters in `--analytics-query` and `-q` strings.
  Put the query in a file and read it (`$(cat f.kql)`, `Get-Content -Raw`), or use PowerShell
  `Search-AzGraph` / `Invoke-AzOperationalInsightsQuery`. On Git Bash set `MSYS_NO_PATHCONV=1` for resource ids.
- Search jobs create `*_SRCH` tables. In them the original event time is `_OriginalTimeGenerated`,
  `TimeGenerated` is the time the search job wrote the row. Filtering a Sentinel `_SRCH` table on
  `TimeGenerated` gives wrong windows.
- `AzureActivity` interactive retention is limited (90 days by default). Older data needs a search job
  (a write operation that creates a table) or an archive restore.

### Azure Resource Graph dialect

- Only a subset of KQL exists. `join` flavours: `innerunique`, `inner`, `leftouter`, `fullouter`.
  `leftanti` / `leftsemi` do not exist: use `leftouter` and `where isnull(rightKey)`.
- At most 3 `join` and 3 `mv-expand` per query (join and union legs counted together). When a comparison needs
  more, run the lists separately and join client-side, as in `Governance-Tags/vm-tag-compliance-lists.kql`.
- `mv-expand` returns at most 128 array items per record unless you add `limit N` (max 2000).
  Subnets, security rules and route lists of large objects are cut off silently without it.
- `countif()` works in native Resource Graph queries, but NOT in Log Analytics cross-queries
  through `arg("")` (also unsupported there: `avg()`, `sumif()`, `percentile()`, `arg_max()`). When in doubt use `summarize count()` after a `where`.
- `let` works, but a table reference such as `resources` can be used as the right side of a join only once.
- Type names: match resource types with `=~` (case-insensitive). ARM returns mixed casing
  (`microsoft.storage/storageAccounts` vs `storageaccounts`) and a plain `==` misses rows. Lower-case
  resource ids (`tolower(id)`) before joining or comparing client-side.
- Many useful resources are NOT in `resources`: backup and ASR items are in `recoveryservicesresources`,
  patch data in `patchassessmentresources` / `patchinstallationresources`, maintenance assignments in
  `maintenanceresources`, security assessments in `securityresources`, role assignments in
  `authorizationresources`, policy in `policyresources`, guest configuration in `guestconfigurationresources`,
  DCR associations in `insightsresources`, alerts in `alertsmanagementresources`, resource groups and
  subscriptions in `resourcecontainers`.
- Unset properties are `null`, and null often means the permissive default. `allowSharedKeyAccess` null on a
  storage account means shared key IS allowed. `enablePurgeProtection` null means off. Test with `isnull()`
  explicitly, see `Storage/storage-account-posture.kql`.
- `properties.extended.instanceView.*` (power state, OS name) is a preview feature, empty for VMs that never reported.
- Update Manager history in ARG (`patchinstallationresources`) keeps about 30 days. A monthly report needs
  a daily snapshot, or a backfill from `AzureActivity` (`installPatches` operations, 90 days in the workspace).
- The assessment tables also contain a child type `.../softwarepatches` (one row per patch). Filter the
  parent type exactly, or counts explode.
- Defender for Cloud assessments refresh about every 24 hours, a fix shows up the next day.
  The assessed resource is `properties.resourceDetails.Id` (capital `I`). `additionalData` field names
  differ between recommendations and scanners, look at one row first.
- ARG alert instances and many history tables are short-lived (about 30 days).

### Log Analytics tables

- `InsightsMetrics` `vm.azm.ms/diskSizeMB` is not always megabytes: on some mounts the value is in other
  units (KiB or bytes), and an unfixed monthly report showed disk use of roughly minus 104,857,500 %.
  `LogicalDisk` queries must scale values above about 10,000,000 and be sanity-checked (used percent between 0 and 100).
  See `VMInsights/vminsights-disk-monthly.kql`.
- VM Insights records AVAILABLE memory, not used memory. Used = total - available, and the peak used value
  comes from the MINIMUM available sample. CPU `UtilizationPercentage` is per VM, CPU count is in the
  `vm.azm.ms/totalCpus` tag.
- Business-hours reports must convert UTC to local time before filtering hours and before bucketing months
  (`datetime_add("hour", offset, TimeGenerated)`). A fixed offset ignores daylight saving.
- `AddonAzureBackupJobs`: a job writes a row per status change. Keep the final one with
  `summarize arg_max(TimeGenerated, *) by JobUniqueId`, otherwise every job is counted twice or more.
  Transferred data is `DataTransferredInMB`. The tables only exist when the vault diagnostic setting uses
  resource-specific mode, not `AzureDiagnostics`. The join key `BackupItemUniqueId` differs in casing
  between `CoreAzureBackup` and `AddonAzureBackupJobs`: lower-case both sides.
- `Heartbeat` proves the agent is alive, not that the data collection rule delivers. Check `InsightsMetrics`
  and `Syslog` / `Event` too. Use `union isfuzzy=true` so a missing table does not fail the whole query.
- `StorageBlobLogs` / `StorageFileLogs` hold nothing for a storage account without the diagnostic setting:
  no rows means "not logged", not "not used". `CallerIpAddress` includes the port, strip it before counting clients.
- Traffic Analytics: the new schema is `NTANetAnalytics` / `NTAIpDetails`, the old one is
  `AzureNetworkAnalytics_CL` with `_s` / `_d` suffixed columns. Flow logs are aggregated, so results
  lag by the processing interval. Check ingestion first (`TrafficAnalytics/traffic-analytics-ingestion-health.kql`).
- `AzureActivity.OperationNameValue` and `_ResourceId` are UPPER CASE in many rows, compare with `=~` / `tolower()`.
  ARM writes a Start / Accept / Success trio per operation, filter on `Success` to avoid triple counting.
- `SecurityIncident` has a row for every incident update, `summarize arg_max(TimeGenerated, *) by IncidentNumber` first.
- `AzureDiagnostics` columns carry type suffixes (`action_id_s`, `client_ip_s`, `DestPort_d` is a double).

### Firewall (FortiGate) syslog

- Keys are prefixed `FTNTFGT` (`FTNTFGTaction`, `FTNTFGTpolicyid`); `src`, `dst`, `dpt`, `proto` are plain.
  Anchor every `extract()` on a leading space or `\b`, and put a trailing space in `has "src=192.0.2.10 "`
  filters, otherwise `src=192.0.2.1` also matches `192.0.2.10`.
- Filter with `has` and a time window before any `extract()` / `parse`, Syslog is large.
- A policy without traffic logging leaves no rows. Absence of a flow in the log is not proof that no
  traffic was permitted.

### Defender Advanced Hunting

- The `Device*` tables are only in the Defender portal (30 days), not in your workspace.
- `DeviceTvmSoftwareEvidenceBeta.DiskPaths` is an array: `mv-expand` it. The device name is in `DeviceInfo`
  (`summarize arg_max(Timestamp, DeviceName) by DeviceId`).
- App Control (WDAC) policy tooling (`ConvertFrom-CIPolicy`, `New-CIPolicy`) only runs in Windows PowerShell 5.1,
  not in pwsh 7.

### Related operational notes

- A VM can hold at most 25 Run Command resources. Delete old ones or new calls are rejected.
- Azure Backup restore of a VM turns boot diagnostics off. Re-enable it after the restore.
- An `install-patches` call that times out has usually still installed the patches. Run `assess-patches` and look again before retrying.
- One diagnostic setting category cannot be sent to the same workspace by two separate settings.

## Index of queries

### Defender

| File | What it answers |
|---|---|
| [advanced-hunting-listening-ports-and-java.kql](Defender/advanced-hunting-listening-ports-and-java.kql) | Listening ports and Java runtimes on selected devices |
| [advanced-hunting-onboarding-health.kql](Defender/advanced-hunting-onboarding-health.kql) | Server and Linux device onboarding and sensor health in Defender for Endpoint |
| [advanced-hunting-process-version-lookup.kql](Defender/advanced-hunting-process-version-lookup.kql) | Which version of a program is running on a device (process events) |
| [advanced-hunting-software-evidence-paths.kql](Defender/advanced-hunting-software-evidence-paths.kql) | Disk paths of a software package on each device (OpenSSL, log4j, any library) |
| [advanced-hunting-software-inventory.kql](Defender/advanced-hunting-software-inventory.kql) | Where is a piece of software installed (device list per version) |
| [advanced-hunting-vulnerable-software.kql](Defender/advanced-hunting-vulnerable-software.kql) | Vulnerable software on devices (Defender Vulnerability Management): summary and per device |
| [app-control-hunting.kql](Defender/app-control-hunting.kql) | App Control for Business (WDAC) block and audit events in Advanced Hunting |
| [recommendation-state-changes.kql](Defender/recommendation-state-changes.kql) | Defender for Cloud recommendation state changes per VM (Healthy to Unhealthy and back) |
| [secure-score-weekly.kql](Defender/secure-score-weekly.kql) | Secure Score per subscription per week |
| [security-incidents-rollup.kql](Defender/security-incidents-rollup.kql) | Security incidents roll-up by provider and product (status and ownership) |
| [update-recommendations-prioritisation.kql](Defender/update-recommendations-prioritisation.kql) | Prioritise Defender for Cloud "Update <software>" recommendations by CVSS, EPSS and exposure |

### LogAnalytics/Audit

| File | What it answers |
|---|---|
| [diagnostic-settings-deleted.kql](LogAnalytics/Audit/diagnostic-settings-deleted.kql) | Diagnostic settings deleted (who silenced a log source) |
| [storage-key-and-sas-usage.kql](LogAnalytics/Audit/storage-key-and-sas-usage.kql) | Storage accounts still being used with account keys or SAS (all accounts in the workspace) |
| [workspace-table-changes.kql](LogAnalytics/Audit/workspace-table-changes.kql) | Changes to Log Analytics tables and workspaces (retention, plan, search jobs) |

### LogAnalytics/AzureActivity

| File | What it answers |
|---|---|
| [runcommand-callers.kql](LogAnalytics/AzureActivity/runcommand-callers.kql) | Who invoked Run Command on which VM |
| [updatemanager-installpatches-backfill.kql](LogAnalytics/AzureActivity/updatemanager-installpatches-backfill.kql) | Update Manager runs older than the Resource Graph window, rebuilt from AzureActivity |
| [vm-activity-timeline.kql](LogAnalytics/AzureActivity/vm-activity-timeline.kql) | Activity log timeline of one VM |

### LogAnalytics/AzureBackup

| File | What it answers |
|---|---|
| [backup-data-transferred.kql](LogAnalytics/AzureBackup/backup-data-transferred.kql) | Backup data transferred per item (full vs incremental) and job status |
| [backup-job-history.kql](LogAnalytics/AzureBackup/backup-job-history.kql) | Backup job history for items matching a name (duration, status, data transferred) |
| [backup-jobs-monthly-detail.kql](LogAnalytics/AzureBackup/backup-jobs-monthly-detail.kql) | Azure Backup job detail for last month (one row per job with host and instance) |
| [backup-jobs-monthly-summary.kql](LogAnalytics/AzureBackup/backup-jobs-monthly-summary.kql) | Azure Backup job success summary for last month (VM and database jobs) |
| [backup-status-by-vault.kql](LogAnalytics/AzureBackup/backup-status-by-vault.kql) | Backup jobs per vault and operation: completed, failed, other |
| [backup-weekly-failures-by-item.kql](LogAnalytics/AzureBackup/backup-weekly-failures-by-item.kql) | Weekly backup job counts and failures per protected item |

### LogAnalytics/Heartbeat

| File | What it answers |
|---|---|
| [heartbeat-gaps.kql](LogAnalytics/Heartbeat/heartbeat-gaps.kql) | Heartbeat continuity of one VM in 5 minute bins |
| [heartbeat-last-seen-os.kql](LogAnalytics/Heartbeat/heartbeat-last-seen-os.kql) | Heartbeat: latest OS name, OS type and major/minor version per VM |
| [running-services-by-computer.kql](LogAnalytics/Heartbeat/running-services-by-computer.kql) | Which database / messaging / mail services run on which machine (ConfigurationData) |
| [software-inventory-configurationchange.kql](LogAnalytics/Heartbeat/software-inventory-configurationchange.kql) | Installed software and change history of a VM (ConfigurationChange / ConfigurationData) |
| [vm-reporting-check.kql](LogAnalytics/Heartbeat/vm-reporting-check.kql) | Is a VM (or a group of VMs) reporting to the workspace? Record counts per table |

### LogAnalytics/SQL

| File | What it answers |
|---|---|
| [sql-mi-login-audit.kql](LogAnalytics/SQL/sql-mi-login-audit.kql) | SQL Managed Instance login audit (failed or successful logins by principal and client IP) |

### LogAnalytics/StorageLogs

| File | What it answers |
|---|---|
| [storage-access-by-auth-type.kql](LogAnalytics/StorageLogs/storage-access-by-auth-type.kql) | Storage account access by authentication type (blob and file) |
| [storage-caller-ips.kql](LogAnalytics/StorageLogs/storage-caller-ips.kql) | Caller IPs, user agents and containers for a storage account |
| [storage-failed-requests.kql](LogAnalytics/StorageLogs/storage-failed-requests.kql) | Failed and denied storage requests (HTTP 403 and other errors) |
| [storage-write-operations.kql](LogAnalytics/StorageLogs/storage-write-operations.kql) | Who writes to a storage account (create container, put blob, put block, append) |

### LogAnalytics/TrafficAnalytics

| File | What it answers |
|---|---|
| [denied-and-high-port-flows.kql](LogAnalytics/TrafficAnalytics/denied-and-high-port-flows.kql) | Denied flows and RPC / high-port traffic between two address ranges |
| [east-west-flow-matrix.kql](LogAnalytics/TrafficAnalytics/east-west-flow-matrix.kql) | Allowed and denied flow volume between two sets of prefixes (east-west matrix) |
| [flows-into-named-ranges.kql](LogAnalytics/TrafficAnalytics/flows-into-named-ranges.kql) | Flows into named subnets (for example the private endpoint subnets), grouped by source |
| [flows-to-azure-storage.kql](LogAnalytics/TrafficAnalytics/flows-to-azure-storage.kql) | Which VMs and subnets talk to Azure Storage public IPs |
| [traffic-analytics-ingestion-health.kql](LogAnalytics/TrafficAnalytics/traffic-analytics-ingestion-health.kql) | Is Traffic Analytics data arriving? Records per hour and per table |
| [traffic-to-destination.kql](LogAnalytics/TrafficAnalytics/traffic-to-destination.kql) | Flows to or from an IP / subnet (Traffic Analytics, NTANetAnalytics) |

### LogAnalytics/VMInsights

| File | What it answers |
|---|---|
| [aks-container-insights-sizing.kql](LogAnalytics/VMInsights/aks-container-insights-sizing.kql) | AKS node CPU and memory sizing from Container Insights (average and P95) |
| [free-disk-space.kql](LogAnalytics/VMInsights/free-disk-space.kql) | Free disk space of one VM (InsightsMetrics, with Perf fallback) |
| [vminsights-cpu-monthly.kql](LogAnalytics/VMInsights/vminsights-cpu-monthly.kql) | VM Insights CPU utilisation per VM per month (business hours) |
| [vminsights-disk-monthly.kql](LogAnalytics/VMInsights/vminsights-disk-monthly.kql) | VM Insights logical disk usage per mount per month (with the KiB/MB unit fix) |
| [vminsights-memory-monthly.kql](LogAnalytics/VMInsights/vminsights-memory-monthly.kql) | VM Insights memory usage per VM per month (AvailableMB to used %) |

### ResourceGraph/AVD

| File | What it answers |
|---|---|
| [avd-resources.kql](ResourceGraph/AVD/avd-resources.kql) | Azure Virtual Desktop host pools, application groups and workspaces |
| [avd-scaling-plans.kql](ResourceGraph/AVD/avd-scaling-plans.kql) | AVD scaling plans with time zone, schedules and host pool references |
| [devtest-auto-shutdown-schedules.kql](ResourceGraph/AVD/devtest-auto-shutdown-schedules.kql) | DevTest Labs / VM auto-shutdown schedules |

### ResourceGraph/Backup-ASR

| File | What it answers |
|---|---|
| [asr-health-summary.kql](ResourceGraph/Backup-ASR/asr-health-summary.kql) | ASR replication health summary and test-failover coverage |
| [asr-replicated-items.kql](ResourceGraph/Backup-ASR/asr-replicated-items.kql) | ASR replicated items: health, state, RPO, failover and test-failover status |
| [backup-items-by-vault.kql](ResourceGraph/Backup-ASR/backup-items-by-vault.kql) | Backup item counts per vault, workload type and management type |
| [backup-protected-items.kql](ResourceGraph/Backup-ASR/backup-protected-items.kql) | Azure Backup protected items with state, policy and last backup result |
| [recovery-vault-settings.kql](ResourceGraph/Backup-ASR/recovery-vault-settings.kql) | Recovery Services vault settings (public access, private endpoints, redundancy, immutability) |

### ResourceGraph/Compute

| File | What it answers |
|---|---|
| [disk-encryption-sets.kql](ResourceGraph/Compute/disk-encryption-sets.kql) | Disk encryption sets and their Key Vault |
| [disk-encryption-summary.kql](ResourceGraph/Compute/disk-encryption-summary.kql) | Managed disk encryption type and network access summary |
| [disks-per-vm.kql](ResourceGraph/Compute/disks-per-vm.kql) | Disk count and total size per VM |
| [sql-vm-licensing.kql](ResourceGraph/Compute/sql-vm-licensing.kql) | SQL Server on Azure VM: image, SKU, management mode and licence type |
| [vm-inventory.kql](ResourceGraph/Compute/vm-inventory.kql) | VM inventory with size, OS, image, zone and protection tags |
| [vm-os-inventory.kql](ResourceGraph/Compute/vm-os-inventory.kql) | VM OS name and version from the instance view |
| [vm-security-profile.kql](ResourceGraph/Compute/vm-security-profile.kql) | VM security profile summary (security type, encryption at host, Secure Boot, vTPM) |
| [vm-size-summary.kql](ResourceGraph/Compute/vm-size-summary.kql) | VM count per size |
| [vm-zones-and-availability.kql](ResourceGraph/Compute/vm-zones-and-availability.kql) | VM zone, availability set, security type and hyper-v generation |

### ResourceGraph/Cost

| File | What it answers |
|---|---|
| [vm-running-inventory-for-reservations.kql](ResourceGraph/Cost/vm-running-inventory-for-reservations.kql) | VM inventory for reservation coverage (size, power state, priority, region) |
| [vmss-inventory-for-reservations.kql](ResourceGraph/Cost/vmss-inventory-for-reservations.kql) | Virtual machine scale sets (Uniform) with size, capacity and priority |

### ResourceGraph/Defender

| File | What it answers |
|---|---|
| [defender-edr-assessment-causes.kql](ResourceGraph/Defender/defender-edr-assessment-causes.kql) | EDR / endpoint protection assessment: counts by status code and cause |
| [defender-plans-pricing.kql](ResourceGraph/Defender/defender-plans-pricing.kql) | Defender for Cloud plans per subscription (tier, sub-plan, extensions) |
| [defender-unhealthy-assessments.kql](ResourceGraph/Defender/defender-unhealthy-assessments.kql) | Unhealthy Defender for Cloud assessments filtered by display name |
| [defender-update-recommendations-per-vm.kql](ResourceGraph/Defender/defender-update-recommendations-per-vm.kql) | "Update <software>" recommendations per VM with severity, package type, scanner and CVEs |
| [defender-update-recommendations-summary.kql](ResourceGraph/Defender/defender-update-recommendations-summary.kql) | "Update <software>" recommendations: unhealthy count per recommendation |
| [mde-coverage-per-vm.kql](ResourceGraph/Defender/mde-coverage-per-vm.kql) | Microsoft Defender for Endpoint coverage per VM (extension, EDR and antivirus assessments) |

### ResourceGraph/Governance-Tags

| File | What it answers |
|---|---|
| [guest-configuration-compliance.kql](ResourceGraph/Governance-Tags/guest-configuration-compliance.kql) | Machine Configuration (guest configuration) compliance per configuration and per VM |
| [policy-assignments.kql](ResourceGraph/Governance-Tags/policy-assignments.kql) | Policy assignments with scope level, enforcement mode and initiative-or-definition |
| [policy-exemptions.kql](ResourceGraph/Governance-Tags/policy-exemptions.kql) | Policy exemptions with category, expiry and scope (optionally only VM-level exemptions) |
| [resource-group-tags.kql](ResourceGraph/Governance-Tags/resource-group-tags.kql) | Resource groups with location and tag count |
| [subscriptions-and-management-groups.kql](ResourceGraph/Governance-Tags/subscriptions-and-management-groups.kql) | Subscriptions with their management group ancestors |
| [tag-coverage.kql](ResourceGraph/Governance-Tags/tag-coverage.kql) | Tag coverage per subscription (how many resources carry the required tags) |
| [vm-tag-compliance-lists.kql](ResourceGraph/Governance-Tags/vm-tag-compliance-lists.kql) | VM tags versus reality (Backup, ASR, Maintenance): the four lists |

### ResourceGraph/KeyVault-AppService

| File | What it answers |
|---|---|
| [appgw-and-appservice-certificates.kql](ResourceGraph/KeyVault-AppService/appgw-and-appservice-certificates.kql) | Certificates that come from Key Vault: Application Gateway SSL certs and App Service certificates |
| [appservice-inventory.kql](ResourceGraph/KeyVault-AppService/appservice-inventory.kql) | App Service / Function App inventory (state, identity, HTTPS, public access, private endpoints, VNet integration) |
| [appservice-vnet-integration.kql](ResourceGraph/KeyVault-AppService/appservice-vnet-integration.kql) | Web apps by integration subnet (which apps use which VNet subnet) |
| [keyvault-posture.kql](ResourceGraph/KeyVault-AppService/keyvault-posture.kql) | Key Vault posture inventory (RBAC vs access policies, purge protection, network, private endpoints) |

### ResourceGraph/LogicApps

| File | What it answers |
|---|---|
| [action-group-logic-app-receivers.kql](ResourceGraph/LogicApps/action-group-logic-app-receivers.kql) | Action groups that call a Logic App, and alert rules wired to them |
| [logic-app-connections.kql](ResourceGraph/LogicApps/logic-app-connections.kql) | API connections (Logic Apps) with API name and connection status |
| [logic-apps-by-state.kql](ResourceGraph/LogicApps/logic-apps-by-state.kql) | Logic Apps (Consumption) with state, created and changed dates |

### ResourceGraph/Monitoring

| File | What it answers |
|---|---|
| [alert-rules-referencing-workspace.kql](ResourceGraph/Monitoring/alert-rules-referencing-workspace.kql) | Alert rules, data collection rules and smart detectors that reference a Log Analytics workspace |
| [alert-rules.kql](ResourceGraph/Monitoring/alert-rules.kql) | Alert rules (metric, log, activity log) with enabled state |
| [dcr-associations.kql](ResourceGraph/Monitoring/dcr-associations.kql) | Which data collection rules are associated with which VMs |
| [dcr-inventory.kql](ResourceGraph/Monitoring/dcr-inventory.kql) | Data collection rules with OS kind, syslog / perf counter / extension data source counts |
| [fired-alerts.kql](ResourceGraph/Monitoring/fired-alerts.kql) | Fired alerts (Azure Monitor alerts) in the last N hours |
| [log-analytics-workspaces.kql](ResourceGraph/Monitoring/log-analytics-workspaces.kql) | Log Analytics workspaces visible to the sign-in (name, RG, subscription, workspace id) |
| [vm-diagnostics-extension.kql](ResourceGraph/Monitoring/vm-diagnostics-extension.kql) | VM diagnostics extensions (Windows Azure Diagnostics / Linux Diagnostic) with version and state |

### ResourceGraph/Networking

| File | What it answers |
|---|---|
| [expressroute-circuits.kql](ResourceGraph/Networking/expressroute-circuits.kql) | ExpressRoute circuits with provisioning state, bandwidth and SKU |
| [network-topology-extract.kql](ResourceGraph/Networking/network-topology-extract.kql) | Network topology extract (subscriptions, RGs, VNets, subnets, peerings, NICs, private endpoints) |
| [nic-private-ip-lookup.kql](ResourceGraph/Networking/nic-private-ip-lookup.kql) | Which NIC / VM / subnet owns a private IP (or a whole range) |
| [nics-in-subnet.kql](ResourceGraph/Networking/nics-in-subnet.kql) | All NICs in one subnet (name fragment) with their VMs and IPs |
| [nsg-flowlog-summary.kql](ResourceGraph/Networking/nsg-flowlog-summary.kql) | Flow log counts by target type, enabled state and identity; and by subscription and region |
| [nsg-flowlogs.kql](ResourceGraph/Networking/nsg-flowlogs.kql) | NSG and VNet flow logs with target, version, retention and Traffic Analytics status |
| [nsg-rule-counts.kql](ResourceGraph/Networking/nsg-rule-counts.kql) | NSGs with attachment counts and allow/deny rule counts per direction |
| [private-dns-zones.kql](ResourceGraph/Networking/private-dns-zones.kql) | Private DNS zones with VNet link and record counts |
| [route-tables.kql](ResourceGraph/Networking/route-tables.kql) | Route tables with every route, next hop and BGP propagation setting |
| [subnets-inventory.kql](ResourceGraph/Networking/subnets-inventory.kql) | All VNet subnets with NSG, route table, delegation, private endpoint and NIC counts |
| [subscription-vnet-overview.kql](ResourceGraph/Networking/subscription-vnet-overview.kql) | VNets with subscription name, address space, subnet and peering counts |
| [vnet-address-spaces.kql](ResourceGraph/Networking/vnet-address-spaces.kql) | Virtual networks with address space, DNS servers and peering count |
| [vnet-gateways-and-connections.kql](ResourceGraph/Networking/vnet-gateways-and-connections.kql) | VPN / ExpressRoute gateways and their connections |
| [vnet-peerings.kql](ResourceGraph/Networking/vnet-peerings.kql) | VNet peerings with state and gateway/forwarding flags |

### ResourceGraph/PrivateEndpoint

| File | What it answers |
|---|---|
| [private-endpoint-subnets.kql](ResourceGraph/PrivateEndpoint/private-endpoint-subnets.kql) | Subnets that hold private endpoints: NSG, route table, network policy, endpoint count |
| [private-endpoints-by-target-type.kql](ResourceGraph/PrivateEndpoint/private-endpoints-by-target-type.kql) | Private endpoints pointing at one resource type (storage, Key Vault, SQL, ...) |
| [private-endpoints-inventory.kql](ResourceGraph/PrivateEndpoint/private-endpoints-inventory.kql) | Private endpoints with target resource, sub-resource, subnet, VNet and private IP |

### ResourceGraph/RBAC

| File | What it answers |
|---|---|
| [role-assignments.kql](ResourceGraph/RBAC/role-assignments.kql) | Role assignments for a principal or on a scope |
| [role-definition-names.kql](ResourceGraph/RBAC/role-definition-names.kql) | Resolve role definition GUIDs to role names |

### ResourceGraph/Storage

| File | What it answers |
|---|---|
| [storage-account-posture.kql](ResourceGraph/Storage/storage-account-posture.kql) | Storage account posture inventory (shared key, public network, TLS, HTTPS, replication, private endpoints) |

### ResourceGraph/UpdateManager

| File | What it answers |
|---|---|
| [maintenance-assignments.kql](ResourceGraph/UpdateManager/maintenance-assignments.kql) | Which maintenance configuration is assigned to which VM, and how many VMs per configuration |
| [maintenance-configurations.kql](ResourceGraph/UpdateManager/maintenance-configurations.kql) | Maintenance configurations (guest patching schedules) |
| [patch-assessment-per-vm.kql](ResourceGraph/UpdateManager/patch-assessment-per-vm.kql) | Update Manager assessment result per VM (pending patch counts, reboot pending) |
| [patch-installation-failures.kql](ResourceGraph/UpdateManager/patch-installation-failures.kql) | Update Manager runs that did not succeed (recent) |
| [patch-installation-results.kql](ResourceGraph/UpdateManager/patch-installation-results.kql) | Update Manager installation results (run history) per VM |
| [pending-patches.kql](ResourceGraph/UpdateManager/pending-patches.kql) | Pending software patches per VM (count, and the full patch list) |
| [vm-patch-mode.kql](ResourceGraph/UpdateManager/vm-patch-mode.kql) | VM patch mode and assessment mode (Linux and Windows) |

### Sentinel

| File | What it answers |
|---|---|
| [fortigate-config-changes.kql](Sentinel/fortigate-config-changes.kql) | FortiGate configuration changes (who changed which policy, address, web filter, SSL setting) |
| [fortigate-denied-and-web-blocks.kql](Sentinel/fortigate-denied-and-web-blocks.kql) | FortiGate blocked traffic and web filter blocks from a source range |
| [fortigate-log-source-coverage.kql](Sentinel/fortigate-log-source-coverage.kql) | Which firewall / syslog sources are sending, and for how many days |
| [fortigate-policy-hits.kql](Sentinel/fortigate-policy-hits.kql) | FortiGate sessions per policy with allowed / denied split (optionally between two ranges) |
| [fortigate-segment-matrix.kql](Sentinel/fortigate-segment-matrix.kql) | FortiGate traffic matrix between named network segments, per day |
| [fortigate-traffic-extract.kql](Sentinel/fortigate-traffic-extract.kql) | FortiGate forward traffic from Syslog: extract source, destination, port, protocol, action, policy |
| [fortigate-vpn-tunnel-stats.kql](Sentinel/fortigate-vpn-tunnel-stats.kql) | FortiGate IPsec tunnel traffic statistics (bytes in and out per interval) |
| [network-change-register.kql](Sentinel/network-change-register.kql) | Azure network change register (succeeded MICROSOFT.NETWORK write and delete operations) |

## License

MIT, see [LICENSE](LICENSE).
