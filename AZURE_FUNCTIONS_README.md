# Deploy Panther MCP Server To Azure Functions

This guide deploys the existing Panther FastMCP server to Azure Functions as a self-hosted MCP custom handler. This is different from the Azure Functions MCP extension tool-trigger model used by `octo_mcp`; this repo already has a FastMCP server, so Azure Functions runs it with the `streamable-http` transport.

The deployment scaffold is:

| File | Purpose |
|------|---------|
| `host.json` | Configures the Azure Functions MCP custom handler |
| `local.settings.example.json` | Template for local Functions settings and Panther credentials |
| `requirements.txt` | Lets Azure remote build install this package and its dependencies |
| `.funcignore` | Excludes local-only files from deployment |

## Prerequisites

Install:

- Python 3.12
- Azure Functions Core Tools v4.5.0 or later
- Azure CLI
- uv
- Node.js
- Podman or Docker for local Azurite storage

Check the tools:

```powershell
python --version
func --version
az --version
uv --version
node --version
```

## Panther Settings

Create a Panther API token and keep these values ready:

```powershell
$env:PANTHER_INSTANCE_URL = "https://YOUR-PANTHER-INSTANCE.domain"
$env:PANTHER_API_TOKEN = "YOUR-PANTHER-API-TOKEN"
```

The URL must include `https://`.

## Local Setup

Create local settings:

```powershell
Copy-Item local.settings.example.json local.settings.json
```

Edit `local.settings.json` and replace:

- `PANTHER_INSTANCE_URL`
- `PANTHER_API_TOKEN`

Create the environment and install dependencies:

```powershell
uv sync --group dev
```

If `uv sync` fails on Windows while building `cffi`, install Microsoft C++ Build Tools or run only the server dependencies from a Linux shell/container. Azure remote build runs on Linux and should use prebuilt wheels.

## Run Local Storage

Azure Functions needs local storage. This example uses Azurite through Podman:

```powershell
podman machine start
```

```powershell
podman run -d --name azurite `
  -p 10000:10000 `
  -p 10001:10001 `
  -p 10002:10002 `
  mcr.microsoft.com/azure-storage/azurite
```

If the container already exists but is stopped:

```powershell
podman start azurite
```

Docker works too:

```powershell
docker run -d --name azurite `
  -p 10000:10000 `
  -p 10001:10001 `
  -p 10002:10002 `
  mcr.microsoft.com/azure-storage/azurite
```

## Run Locally With Azure Functions

Start the Functions host:

```powershell
uv run func start
```

The local MCP endpoint is:

```text
http://localhost:7071/runtime/webhooks/mcp
```

If you want to test the FastMCP server without Azure Functions, run:

```powershell
uv run python -m mcp_panther --transport streamable-http --host 127.0.0.1 --port 3000
```

The direct FastMCP endpoint is:

```text
http://127.0.0.1:3000/mcp
```

## Test Locally

Start MCP Inspector:

```powershell
npx @modelcontextprotocol/inspector
```

Open the Inspector URL printed in the terminal, usually:

```text
http://127.0.0.1:6274
```

For the Azure Functions local host, connect to:

```text
http://localhost:7071/runtime/webhooks/mcp
```

For the direct FastMCP process, connect to:

```text
http://127.0.0.1:3000/mcp
```

Click `List Tools`. You should see the Panther tools from the main README.

## Create Azure Resources With az

Sign in:

```powershell
az login
```

Set deployment values:

```powershell
$RESOURCE_GROUP = "rg-mcp-panther"
$LOCATION = "centralus"
$STORAGE_ACCOUNT = "stmcppanther$((Get-Random -Maximum 99999999).ToString('00000000'))"
$FUNCTION_APP_NAME = "func-mcp-panther-$(Get-Random)"
$ZIP_FILE = "mcp-panther-azure.zip"
```

Create the resource group:

```powershell
az group create `
  --name $RESOURCE_GROUP `
  --location $LOCATION
```

Register the provider used by Flex Consumption:

```powershell
az provider register --namespace Microsoft.App
```

Check registration:

```powershell
az provider show `
  --namespace Microsoft.App `
  --query "registrationState" `
  --output tsv
```

Create storage:

```powershell
az storage account create `
  --name $STORAGE_ACCOUNT `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --sku Standard_LRS
```

Create a Linux Python Function App on Flex Consumption:

```powershell
az functionapp create `
  --name $FUNCTION_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --storage-account $STORAGE_ACCOUNT `
  --flexconsumption-location $LOCATION `
  --runtime python `
  --runtime-version 3.12 `
  --functions-version 4
```

Add required app settings:

```powershell
az functionapp config appsettings set `
  --name $FUNCTION_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --settings `
    "AzureWebJobsFeatureFlags=EnableMcpCustomHandlerPreview" `
    "PANTHER_INSTANCE_URL=$env:PANTHER_INSTANCE_URL" `
    "PANTHER_API_TOKEN=$env:PANTHER_API_TOKEN"
```

## Deploy

Create the deployment zip:

```powershell
Remove-Item $ZIP_FILE -Force -ErrorAction SilentlyContinue

$exclude = @(
  ".venv",
  ".git",
  ".github",
  ".pytest_cache",
  ".ruff_cache",
  "htmlcov",
  "local.settings.json",
  "__pycache__",
  $ZIP_FILE
)

Get-ChildItem -Force |
  Where-Object { $exclude -notcontains $_.Name } |
  Compress-Archive -DestinationPath $ZIP_FILE -Force
```

Confirm the zip contains the Azure Functions files:

```powershell
tar -tf $ZIP_FILE | Select-String "host.json|requirements.txt|src/mcp_panther/server.py"
```

Deploy with remote build:

```powershell
az functionapp deployment source config-zip `
  --src $ZIP_FILE `
  --name $FUNCTION_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --build-remote true
```

Restart after deployment:

```powershell
az functionapp restart `
  --name $FUNCTION_APP_NAME `
  --resource-group $RESOURCE_GROUP
```

## Test In Azure

Get the Function App hostname:

```powershell
$HOSTNAME = az functionapp show `
  --name $FUNCTION_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --query "defaultHostName" `
  --output tsv
```

The remote MCP endpoint is:

```powershell
"https://$HOSTNAME/runtime/webhooks/mcp"
```

Start MCP Inspector:

```powershell
npx @modelcontextprotocol/inspector
```

Connect to:

```text
https://<function-app-hostname>/runtime/webhooks/mcp
```

Click `List Tools` and run a low-risk read-only Panther tool such as `get_permissions`.

This scaffold sets the Functions host authorization level to anonymous because the self-hosted MCP custom-handler preview expects MCP clients to reach the MCP endpoint directly. For production, put the Function App behind the authentication and network controls your organization requires before exposing it broadly.

## Useful PowerShell Commands

Stream logs:

```powershell
az webapp log tail `
  --name $FUNCTION_APP_NAME `
  --resource-group $RESOURCE_GROUP
```

List app settings:

```powershell
az functionapp config appsettings list `
  --name $FUNCTION_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --output table
```

Update Panther credentials:

```powershell
az functionapp config appsettings set `
  --name $FUNCTION_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --settings `
    "PANTHER_INSTANCE_URL=$env:PANTHER_INSTANCE_URL" `
    "PANTHER_API_TOKEN=$env:PANTHER_API_TOKEN"
```

Restart:

```powershell
az functionapp restart `
  --name $FUNCTION_APP_NAME `
  --resource-group $RESOURCE_GROUP
```

Delete the resource group when done:

```powershell
az group delete `
  --name $RESOURCE_GROUP `
  --yes
```

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `func: command not found` | Install Azure Functions Core Tools v4.5.0 or later |
| Local storage error | Start Azurite and confirm `AzureWebJobsStorage` is `UseDevelopmentStorage=true` |
| Local host starts but Inspector cannot connect | Use `http://localhost:7071/runtime/webhooks/mcp` for Functions or `http://127.0.0.1:3000/mcp` for direct FastMCP |
| Remote tools do not appear | Confirm `AzureWebJobsFeatureFlags=EnableMcpCustomHandlerPreview` is set |
| Custom handler fails to start | Confirm `host.json` passes `--transport streamable-http` and uses the same port as `customHandler.port` |
| Panther calls fail | Confirm `PANTHER_INSTANCE_URL` includes `https://` and the token has the required Panther permissions |
| Deployment succeeds but imports fail | Confirm `requirements.txt` is included and remote build is enabled |

## References

- [Host servers built with MCP SDKs on Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/scenario-host-mcp-server-sdks)
- [Azure Functions custom handlers](https://learn.microsoft.com/en-us/azure/azure-functions/functions-custom-handlers)
- [Azure Functions MCP tutorial](https://learn.microsoft.com/en-us/azure/azure-functions/functions-mcp-tutorial)
