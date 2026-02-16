# Open Systems Logs API Solution for Microsoft Sentinel

This repository contains the Open Systems data connector solution for Microsoft Sentinel. It allows you to ingest logs from Open Systems Logs API into your Sentinel workspace for security monitoring, threat detection, and incident investigation.

The solution deploys a single data connector that supports multiple log types including Secure Web Gateway, Identity Service, Firewall, Zero Trust Network Access, Secure E-Mail Gateway, and Intrusion Detection System logs.

## What's Included

- **Data Connector** - A single Microsoft Sentinel data connector for all Open Systems log types
- **ASIM Parsers** - ASIM-compliant parsers for Authentication, Network Session (Firewall), and Web Session (Secure Web Gateway) schemas
- **Custom Tables** - Pre-defined table schemas for storing Open Systems events
- **Data Collection Rules** - Transformation rules that parse and normalize incoming data
- **Container App** - Azure Container App running a log processor to process and forward logs
- **Data Collection Endpoint** - Endpoint for receiving log data

## Repository Structure

```
armTemplate/
├── customUiDefinition.json    # Custom UI definition for deployment wizard
├── solution.json              # ARM template for the solution
└── README.md                  # This documentation
```

## Data Tables

The solution creates the following custom tables in your Log Analytics workspace:

- `OpenSystemsProxyLogs_CL` - Secure Web Gateway logs
- `OpenSystemsAuthenticationLogs_CL` - Identity Service logs
- `OpenSystemsFirewallLogs_CL` - Firewall logs
- `OpenSystemsZtnaLogs_CL` - Zero Trust Network Access logs
- `OpenSystemsMailLogs_CL` - Secure E-Mail Gateway logs
- `OpenSystemsZeekLogs_CL` - Intrusion Detection System logs

## Prerequisites

Before you start, you'll need:

1. An existing Log Analytics workspace with Microsoft Sentinel enabled
2. **Azure Service Principal** - An Azure service principal that the log processor will use for authentication when ingesting logs into your Log Analytics workspace (the ARM template will automatically assign the required Monitoring Metrics Publisher role)
3. **Permissions** - The user deploying this solution must have rights to assign Azure permissions in the target resource group
4. Open Systems Logs API credentials and configuration provided by Open Systems:
   - **Logs API Endpoint**: The endpoint URL for your Open Systems account
   - **Logs API Connection String**: The connection string for the Logs API
   - **Available Log Types**: List of log types available for your account

### Obtaining Open Systems Credentials

Open Systems will provide you with the required Logs API credentials and configuration based on your account setup. Contact your Open Systems technical account management team to obtain:

- The Logs API endpoint URL (typically in the format `logs-your-namespace.servicebus.windows.net:9093`)
- The Logs API connection string with appropriate permissions
- List of available log types for your account

## Installation

### Selecting Log Streams

The solution supports selective log stream ingestion. Log stream selection happens during ARM template deployment.

Available Log Streams:
- **Secure Web Gateway Logs** - Web traffic filtering and security events
- **Identity Service Logs** - User authentication and identity management events
- **Firewall Logs** - Network firewall activity and security events
- **Zero Trust Network Access Logs** - ZTNA access and session events
- **Secure E-Mail Gateway Logs** - Email security and filtering events
- **Intrusion Detection System Logs** - IDS alerts and network intrusion events

### Quick Deploy

**One-Click Deploy (Recommended):**

Click the button below to deploy directly to Azure Portal with the full configuration UI:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fopen-ch%2Fopen-sentinel%2Fmain%2Fsolution.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2Fopen-ch%2Fopen-sentinel%2Fmain%2FcustomUiDefinition.json)

This will open Azure Portal with a guided deployment form where you can:
- Select your existing workspace
- Select or create an Azure service principal for log ingestion (this will be used by the log processor for authentication)
- Choose which log streams to enable with checkboxes
- Configure container resources (CPU, memory) and autoscaling
- Provide your Open Systems Logs API credentials

---

**Manual Deploy via CLI**

If you prefer command-line deployment:

```bash
az deployment group create \
  --resource-group <your-resource-group> \
  --template-file solution.json \
  --parameters \
    workspace='<your-workspace>' \
    location='<region>' \
    principalId='<service-principal-object-id>' \
    principalClientId='<service-principal-client-id>' \
    principalSecret='<service-principal-secret>' \
    openSystemsLogsAPIEndpoint='<your-endpoint>' \
    openSystemsLogsAPIConnectionString='<your-connection-string>' \
    secureWebGateway=true \
    identity=true \
    firewall=true \
    zeroTrustNetworkAccess=true \
    secureE-MailGateway=false \
    intrusionDetectionSystem=false \
    containerCpu='1.0' \
    containerMemory=2 \
    enableAutoScale=true \
    collectContainerLogs=false
```

Replace the parameter values with your actual configuration.

## Configuration

After deployment, the data connector will be available in Microsoft Sentinel. You may need to:

1. Navigate to **Microsoft Sentinel** > **Data connectors**
2. Find the "Open Systems Data Connector"
3. Open the connector and verify the configuration
4. Check that data is being ingested into the expected tables

### Monitoring Data Ingestion

Monitor data ingestion in Sentinel under **Logs** with queries like:
```kql
OpenSystemsProxyLogs_CL
| take 10
```

## Costs

The Open Systems data connector solution for Microsoft Sentinel is free to use. However, it comes with costs Azure Container App costs to run the processor and with Sentinel storage costs depending on the ingested data volume.

## Troubleshooting

If you experience issues with log ingestion, check the console logs of the log processor (Container App) for detailed error information:

1. Navigate to the Azure Portal
2. Go to **Container Apps** in your resource group
3. Find the container app named `opensystems-sentinel-connector`
4. Go to **Replicas** in the container app menu
5. Select the running replica instance
6. Click on **Console** to access the container's console logs
7. Check the console output for any error messages or connection issues

Alternatively, if log collection is enabled, you can also check under **Monitoring** > **Logs** in the container app for additional diagnostic information.

## Support

For support with this solution:
- Contact your Open Systems technical account management team.
