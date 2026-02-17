# Event Hub Multi-Region Deployment for Dynatrace

This ARM template deploys Azure Event Hub namespaces across multiple Azure regions with standardized configuration for Dynatrace log and event ingestion. The deployment follows Azure Cloud Adoption Framework (CAF) and Well-Architected Framework (WAF) naming conventions.

## Overview

This solution provides:

- **Multi-Region Deployment**: Deploy Event Hub namespaces across multiple Azure regions for high availability and disaster recovery
- **Azure CAF Naming Conventions**: All resources follow Azure CAF naming standards
- **Dynatrace Integration**: Pre-configured for Dynatrace log and event ingestion
- **RBAC Configuration**: Automatic role assignments for Dynatrace service principal
- **Standardized Configuration**: Consistent settings across all regional deployments
- **UI Definition**: Interactive Azure Portal deployment experience

## Architecture

The deployment creates the following resources per region:

1. **Resource Group**: `rg-{workload}-{env}-{region}`
2. **Event Hub Namespace**: `evhns-{workload}-{env}-{region}`
3. **Event Hub**: `evh-logs-ingestion`
4. **Authorization Rules**: Three policies for Listen, Send, and Manage access
5. **RBAC Role Assignments**: 
   - Azure Event Hubs Data Receiver (a638d3c7-ab3a-418d-83e6-5f17a39d4fde)
   - Azure Event Hubs Data Sender (2b629674-e913-4c01-ae53-ef4638d8f975)

## Azure CAF Naming Conventions

This template follows Azure CAF naming conventions:

| Resource Type | Naming Pattern | Example |
|---------------|----------------|---------|
| Resource Group | `rg-{workload}-{env}-{region}` | `rg-dynatrace-prod-eastus` |
| Event Hub Namespace | `evhns-{workload}-{env}-{region}` | `evhns-dynatrace-prod-eastus` |
| Event Hub | `evh-{purpose}` | `evh-logs-ingestion` |

## Prerequisites

1. **Azure Subscription**: Active Azure subscription with appropriate permissions
2. **Dynatrace Service Principal**: A service principal with:
   - **Object ID** (required for RBAC assignments)
   - Appropriate permissions in your Azure AD tenant
3. **Azure CLI** (for retrieving Service Principal Object ID):
   ```bash
   az ad sp show --id <application-id> --query id -o tsv
   ```

### Important: Service Principal Object ID vs App ID

The deployment requires the **Object ID** of the service principal, not the Application (Client) ID:

- **Application (Client) ID**: The unique identifier of the app registration
- **Object ID**: The unique identifier of the service principal object in your tenant

To get the Object ID from the Application ID:

```bash
# Using Azure CLI
az ad sp show --id <application-id> --query id -o tsv

# Or using Azure PowerShell
(Get-AzADServicePrincipal -ApplicationId <application-id>).Id
```

## Deployment Options

### Option 1: Azure Portal (Recommended)

1. Click the "Deploy to Azure" button:

   [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdynatrace-oss%2Fcloud-snippets%2Fmain%2Fazure%2Feventhub-dynatrace-deployment%2Feventhub-deployment.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2Fdynatrace-oss%2Fcloud-snippets%2Fmain%2Fazure%2Feventhub-dynatrace-deployment%2FcreateUiDefinition.json)

2. Fill in the required parameters in the UI
3. Review and create the deployment

### Option 2: Azure CLI

```bash
# Create a parameters file
cat > parameters.json << EOF
{
  "\$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "workloadName": {
      "value": "dynatrace"
    },
    "environment": {
      "value": "prod"
    },
    "regions": {
      "value": ["eastus", "westus", "northeurope"]
    },
    "eventHubSku": {
      "value": "Standard"
    },
    "eventHubCapacity": {
      "value": 2
    },
    "eventHubRetentionDays": {
      "value": 7
    },
    "eventHubPartitionCount": {
      "value": 4
    },
    "dtMonitoringServicePrincipalId": {
      "value": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    },
    "enableAutoInflate": {
      "value": true
    },
    "maximumThroughputUnits": {
      "value": 10
    }
  }
}
EOF

# Deploy at subscription scope
az deployment sub create \
  --name eventhub-dynatrace-deployment \
  --location eastus \
  --template-file eventhub-deployment.json \
  --parameters parameters.json
```

### Option 3: Azure PowerShell

```powershell
# Create parameters hash table
$parameters = @{
    workloadName = "dynatrace"
    environment = "prod"
    regions = @("eastus", "westus", "northeurope")
    eventHubSku = "Standard"
    eventHubCapacity = 2
    eventHubRetentionDays = 7
    eventHubPartitionCount = 4
    dtMonitoringServicePrincipalId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    enableAutoInflate = $true
    maximumThroughputUnits = 10
}

# Deploy at subscription scope
New-AzDeployment `
    -Name "eventhub-dynatrace-deployment" `
    -Location "eastus" `
    -TemplateFile "eventhub-deployment.json" `
    -TemplateParameterObject $parameters
```

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `regions` | array | Array of Azure regions for deployment (e.g., ["eastus", "westus"]) |
| `dtMonitoringServicePrincipalId` | string | Object ID of the Dynatrace service principal |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `workloadName` | string | "dynatrace" | Workload name for resource naming |
| `environment` | string | "prod" | Environment (dev, test, staging, prod) |
| `eventHubSku` | string | "Standard" | Event Hub SKU (Basic, Standard, Premium) |
| `eventHubCapacity` | int | 1 | Throughput units (1-20 for Standard/Basic) |
| `eventHubRetentionDays` | int | 7 | Message retention in days (1-7 for Standard/Basic) |
| `eventHubPartitionCount` | int | 4 | Number of partitions (2-32) |
| `enableAutoInflate` | bool | false | Enable auto-inflate for Standard tier |
| `maximumThroughputUnits` | int | 0 | Max TU when auto-inflate enabled |
| `tags` | object | See template | Resource tags |

## Event Hub SKU Comparison

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Throughput Units | 1-20 | 1-20 | 1-10 (Processing Units) |
| Message Retention | 1-7 days | 1-7 days | 1-90 days |
| Auto-Inflate | No | Yes | No |
| Zone Redundancy | No | No | Yes |
| Price | $ | $$ | $$$ |
| Use Case | Development | Production | Mission-Critical |

## RBAC Roles Assigned

The template automatically assigns the following built-in Azure roles to the Dynatrace service principal on each Event Hub namespace:

1. **Azure Event Hubs Data Receiver** (a638d3c7-ab3a-418d-83e6-5f17a39d4fde)
   - Allows receiving messages from Event Hubs
   - Required for Dynatrace to consume logs and events

2. **Azure Event Hubs Data Sender** (2b629674-e913-4c01-ae53-ef4638d8f975)
   - Allows sending messages to Event Hubs
   - Required if Dynatrace needs to send data to Event Hubs

## Authorization Rules

Three authorization rules are created for each Event Hub:

1. **dynatrace-listen**: Listen-only access
2. **dynatrace-send**: Send-only access
3. **dynatrace-manage**: Full access (Listen, Send, Manage)

Access keys for these rules can be retrieved from the Azure Portal or via Azure CLI:

```bash
# Get connection string for manage rule
az eventhubs eventhub authorization-rule keys list \
  --resource-group rg-dynatrace-prod-eastus \
  --namespace-name evhns-dynatrace-prod-eastus \
  --eventhub-name evh-logs-ingestion \
  --name dynatrace-manage \
  --query primaryConnectionString -o tsv
```

## Outputs

The template returns an array of deployed Event Hub namespaces:

```json
{
  "eventHubNamespaces": [
    {
      "region": "eastus",
      "resourceGroup": "rg-dynatrace-prod-eastus",
      "namespace": "evhns-dynatrace-prod-eastus",
      "eventHub": "evh-logs-ingestion"
    },
    {
      "region": "westus",
      "resourceGroup": "rg-dynatrace-prod-westus",
      "namespace": "evhns-dynatrace-prod-westus",
      "eventHub": "evh-logs-ingestion"
    }
  ]
}
```

## Post-Deployment Steps

1. **Verify Deployment**:
   ```bash
   # List all Event Hub namespaces
   az eventhubs namespace list --query "[?tags.purpose=='dynatrace-log-ingestion']" -o table
   ```

2. **Retrieve Connection Strings**:
   ```bash
   # For each region
   az eventhubs eventhub authorization-rule keys list \
     --resource-group <resource-group-name> \
     --namespace-name <namespace-name> \
     --eventhub-name evh-logs-ingestion \
     --name dynatrace-manage \
     --query primaryConnectionString -o tsv
   ```

3. **Configure Dynatrace**:
   - Add the Event Hub connection strings to your Dynatrace environment
   - Configure log forwarding rules to send logs to the Event Hubs
   - Verify data ingestion in Dynatrace

4. **Set Up Monitoring**:
   - Enable Azure Monitor diagnostic settings for the Event Hub namespaces
   - Configure alerts for throughput, errors, and availability

## Best Practices

1. **Security**:
   - Use managed identities when possible instead of connection strings
   - Rotate authorization rule keys regularly
   - Use Azure Key Vault to store connection strings
   - Enable firewall rules to restrict access to Event Hubs

2. **Performance**:
   - Choose appropriate partition count based on expected throughput
   - Enable auto-inflate for Standard tier to handle traffic spikes
   - Monitor throughput unit usage and adjust as needed

3. **High Availability**:
   - Deploy to at least two regions for disaster recovery
   - Use Premium tier for zone redundancy
   - Implement retry logic in consuming applications

4. **Cost Optimization**:
   - Use Basic tier for development/testing
   - Monitor throughput unit usage to avoid over-provisioning
   - Set appropriate message retention to balance cost and requirements

## Troubleshooting

### Deployment Fails with "InvalidTemplate"

- Verify JSON syntax in parameters file
- Ensure regions array is properly formatted
- Check that service principal Object ID is a valid GUID

### Role Assignment Fails

- Verify the service principal Object ID (not App ID)
- Ensure you have sufficient permissions to assign roles
- Check that the service principal exists in your Azure AD tenant

### Event Hub Not Accessible

- Verify firewall rules allow access
- Check RBAC role assignments are complete
- Ensure network connectivity to Event Hub namespace

## Contributing

Contributions are welcome! Please follow the repository's contribution guidelines.

## License

Copyright 2022 Dynatrace LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

## Support

For issues and questions:
- GitHub Issues: [cloud-snippets issues](https://github.com/dynatrace-oss/cloud-snippets/issues)
- Documentation: [Azure Event Hubs documentation](https://docs.microsoft.com/azure/event-hubs/)
