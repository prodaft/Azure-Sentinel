# PRODAFTUstaIoC-Backfill

On-demand backfill playbook for the **PRODAFT USTA - IoC Threat Intelligence** solution.

The import playbooks only poll **forward** from a per-feed watermark. This playbook loads
history: it pages a chosen USTA IoC feed over a historical window, maps each record to a
STIX 2.1 indicator, and uploads it to Microsoft Sentinel Threat Intelligence via the same
**Upload STIX Objects** action used by the import playbooks. Indicators are written under the
same per-feed `SourceSystem` as the importer (`PRODAFT USTA - Malicious URLs`, `… - Malware
Hashes`, or `… - Phishing Sites`), so backfilled history unifies with the incremental data and
is picked up by that feed's watermark. Re-uploads are idempotent (deterministic STIX ids), so
running it more than once is safe.

## Parameters

| Parameter | Description |
|---|---|
| `UstaBaseUrl` | PRODAFT USTA API base URL (default `https://usta.prodaft.com`). |
| `UstaApiKey` | PRODAFT USTA long-lived API key (secured). |
| `WorkspaceID` | Log Analytics workspace ID (GUID) of the Sentinel workspace. |
| `Feed` | Which feed to backfill: `malicious-urls`, `malware-hashes`, or `phishing-sites`. |
| `BackfillDays` | Number of days of history to load (default 30). |
| `ValidityDays` | Validity window applied to phishing indicators from their created date, since the feed has no expiry (default 7). |

## Deploy — from the portal

1. **Microsoft Sentinel → Content hub → PRODAFT USTA - IoC Threat Intelligence → Manage → Playbook templates**, select **PRODAFT USTA - IoC Backfill**, choose **Create playbook**, and supply `UstaApiKey` and `WorkspaceID`. (Or **Automation → Create → Playbook**, then deploy this `azuredeploy.json`.)
2. The playbook is created with a **system-assigned managed identity** automatically.
3. **Grant the role** (required for the upload): open the **Log Analytics workspace → Access control (IAM) → Add → Add role assignment** → Role **Microsoft Sentinel Contributor** → **Members: Managed identity** → pick this Logic App by name → **Review + assign**.
4. Open the Logic App → **Run Trigger → manual**, once **per feed** (set the `Feed` parameter each time). Since it is manually triggered, it does nothing until you run it.

## Deploy — via Azure CLI (run from this folder)

```bash
# ---- configuration ----
SUB="<subscription-id>"
RG="<resource-group>"                 # resource group of the Sentinel workspace
WS="<workspace-name>"                  # Log Analytics workspace name
WORKSPACE_ID="<workspace-guid>"        # workspace ID (GUID), from workspace Overview
USTA_API_KEY="<usta-api-key>"
PLAYBOOK="PRODAFTUstaIoC-Backfill"

az account set --subscription "$SUB"

# 1. Deploy the playbook and capture its managed-identity principalId
PRINCIPAL_ID=$(az deployment group create \
  --resource-group "$RG" \
  --template-file azuredeploy.json \
  --parameters PlaybookName="$PLAYBOOK" \
               UstaApiKey="$USTA_API_KEY" \
               WorkspaceID="$WORKSPACE_ID" \
  --query properties.outputs.playbookPrincipalId.value -o tsv)

# 2. Grant that identity 'Microsoft Sentinel Contributor' on the workspace
az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Microsoft Sentinel Contributor" \
  --scope "/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.OperationalInsights/workspaces/$WS"

# 3. Run the backfill once PER FEED (repeat with Feed=malware-hashes and Feed=phishing-sites)
az rest --method POST \
  --url "https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Logic/workflows/$PLAYBOOK/triggers/manual/run?api-version=2016-10-01"
```

> **RBAC propagation:** the role assignment in step 2 can take a minute to take effect. If a
> run shows 403 on the Upload STIX Objects action, wait a moment and run the trigger again.
>
> The `Feed` parameter is set at deploy time. To backfill a different feed, redeploy (step 1)
> with `--parameters ... Feed=<feed>` and run the trigger again.

Verify results:

```kql
ThreatIntelIndicators
| where SourceSystem startswith "PRODAFT USTA"
| sort by TimeGenerated desc
| take 20
```

> **Note on phishing history:** phishing indicators use a `ValidityDays` window measured from
> their `created` date, so backfilled phishing sites older than `ValidityDays` land already
> expired (not Active) — expected, since old phishing sites are typically dead. Malicious-URL
> and malware-hash indicators carry the feed's own `valid_until` and remain active.
