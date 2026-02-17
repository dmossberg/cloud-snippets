# Event Hub Deployment Quick Reference

## Get Service Principal Object ID

```bash
# From Application ID
az ad sp show --id <application-id> --query id -o tsv

# Or using PowerShell
(Get-AzADServicePrincipal -ApplicationId <application-id>).Id
```

## Deploy Template

### Azure Portal
Use the "Deploy to Azure" button in README.md

### Azure CLI
```bash
az deployment sub create \
  --name eventhub-dynatrace \
  --location eastus \
  --template-file eventhub-deployment.json \
  --parameters parameters.example.json
```

### Azure PowerShell
```powershell
New-AzDeployment `
  -Name "eventhub-dynatrace" `
  -Location "eastus" `
  -TemplateFile "eventhub-deployment.json" `
  -TemplateParameterFile "parameters.example.json"
```

## Retrieve Connection Strings

```bash
# List all Event Hub namespaces created by this template
az eventhubs namespace list \
  --query "[?tags.purpose=='dynatrace-log-ingestion'].{Name:name, Location:location, ResourceGroup:resourceGroup}" \
  -o table

# Get connection string for a specific Event Hub
az eventhubs eventhub authorization-rule keys list \
  --resource-group rg-dynatrace-prod-eastus \
  --namespace-name evhns-dynatrace-prod-eastus \
  --eventhub-name evh-logs-ingestion \
  --name dynatrace-manage \
  --query primaryConnectionString -o tsv
```

## Verify RBAC Assignments

```bash
# List role assignments for a namespace
az role assignment list \
  --scope /subscriptions/<subscription-id>/resourceGroups/rg-dynatrace-prod-eastus/providers/Microsoft.EventHub/namespaces/evhns-dynatrace-prod-eastus \
  --query "[].{Role:roleDefinitionName, Principal:principalName}" \
  -o table
```

## Monitor Event Hub

```bash
# Get namespace metrics
az monitor metrics list \
  --resource /subscriptions/<subscription-id>/resourceGroups/rg-dynatrace-prod-eastus/providers/Microsoft.EventHub/namespaces/evhns-dynatrace-prod-eastus \
  --metric IncomingMessages,OutgoingMessages,ThrottledRequests \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T23:59:59Z
```

## Delete Resources

```bash
# Delete a specific resource group
az group delete --name rg-dynatrace-prod-eastus --yes --no-wait

# Delete all resource groups created by this template
az group list --tag purpose=dynatrace-log-ingestion --query "[].name" -o tsv | \
  xargs -I {} az group delete --name {} --yes --no-wait
```

## Common Issues

### Issue: "InvalidResourceReference"
**Solution**: Ensure resource groups are created before deploying Event Hubs. The template handles this with `dependsOn`.

### Issue: "AuthorizationFailed" for role assignments
**Solution**: Verify you have `User Access Administrator` or `Owner` role on the subscription.

### Issue: "PrincipalNotFound"
**Solution**: Double-check the service principal Object ID (not App ID). Use `az ad sp show --id <app-id>` to get it.

## Resource Naming Pattern

| Resource | Pattern | Example |
|----------|---------|---------|
| Resource Group | `rg-{workload}-{env}-{region}` | `rg-dynatrace-prod-eastus` |
| Event Hub Namespace | `evhns-{workload}-{env}-{region}` | `evhns-dynatrace-prod-eastus` |
| Event Hub | `evh-{purpose}` | `evh-logs-ingestion` |

## Azure CAF Abbreviations Reference

- `rg` - Resource Group
- `evhns` - Event Hub Namespace
- `evh` - Event Hub
- `kv` - Key Vault
- `st` - Storage Account
- `la` - Log Analytics
