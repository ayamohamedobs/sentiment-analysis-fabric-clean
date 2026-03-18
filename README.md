# Survey Sentiment Analysis Agent

AI-powered survey analysis using **Azure AI Foundry Agent** with **Azure AI Language** and **Microsoft Fabric** integration. Analyze survey data from local Excel/CSV files or directly from a Fabric semantic model — all through a Streamlit web interface that delivers comprehensive sentiment analysis, key themes, and actionable recommendations.

> [!IMPORTANT]
> **DISCLAIMER:** This is a proof-of-concept (POC) sample application provided for demonstration and educational purposes only. This code is provided "AS IS" without warranty of any kind. Microsoft makes no warranties, express or implied, with respect to this sample code and disclaims all implied warranties including, without limitation, any implied warranties of merchantability, fitness for a particular purpose, or non-infringement. The entire risk arising out of the use or performance of the sample code remains with you. In no event shall Microsoft, its authors, or anyone else involved in the creation, production, or delivery of the code be liable for any damages whatsoever (including, without limitation, damages for loss of business profits, business interruption, loss of business information, or other pecuniary loss) arising out of the use of or inability to use the sample code, even if Microsoft has been advised of the possibility of such damages.
>
> **Use this code at your own risk.** This is not a production-ready solution and should not be used in production environments without proper review, testing, and modifications to meet your specific requirements.

## Features

- 📊 **Multi-column analysis** — Select 1-2 columns from Excel/CSV files
- 🔗 **Microsoft Fabric integration** — Query survey data directly from Fabric semantic models via the built-in Fabric Data Agent tool
- 🤖 **AI-powered insights** — GPT-4o agent with Azure Language integration
- 💬 **Interactive chat** — Ask questions and get detailed analysis
- 📈 **Comprehensive metrics** — Sentiment distribution, themes, key phrases, entities
- 🔒 **Zero API keys** — All auth via managed identity
- ⚡ **Batch processing** — Analyzes up to 100 responses per file

## Architecture

The solution supports two data sources — local file upload and Microsoft Fabric semantic models:

```
                    ┌──────────────────────────────────────────────────────────────┐
                    │                   Azure AI Foundry Agent (GPT-4o)            │
                    │                   - Orchestrates the workflow               │
                    │                   - Calls Language tools for NLP            │
                    │                   - Calls Fabric tool for data retrieval    │
                    │                   - Generates structured analysis           │
                    └────────┬─────────────────────┬──────────────────────────────┘
                             │                     │
                    ┌────────▼────────┐    ┌───────▼────────────┐
                    │  Azure AI       │    │  Microsoft Fabric  │
                    │  Language SDK   │    │  Data Agent        │
                    │  - Sentiment    │    │  (FabricTool)      │
                    │  - Key Phrases  │    │  - Translates NL   │
                    │  - NER & PII    │    │    queries to DAX  │
                    │  - Language Det │    │  - Returns data    │
                    └─────────────────┘    │    from semantic    │
                                           │    model            │
                                           └────────────────────┘
```

### Data Flow

| Step | Component | Action |
|------|-----------|--------|
| 1 | **User** | Uploads a file OR types a natural language query for Fabric |
| 2 | **Streamlit UI** | Parses file / sends query to the Foundry Agent |
| 3 | **Foundry Agent** | For Fabric queries: calls the `fabric_dataagent` tool to retrieve data |
| 4 | **Fabric Data Agent** | Translates query to DAX, fetches rows from the semantic model |
| 5 | **Foundry Agent** | Calls Language tools (sentiment, key phrases, entities) on the data |
| 6 | **Language Service** | Returns NLP analysis results |
| 7 | **Foundry Agent** | Compiles 5-section structured analysis report |

### Tool Architecture: Dual-Mode Design

The agent supports two modes for calling Azure AI Language services:

**SDK Mode (Current)** — Function tools executed locally
- Agent calls Language analysis functions (`analyze_sentiment`, `extract_key_phrases`, etc.)
- Streamlit app intercepts tool calls and executes them using Azure Language SDK
- Uses managed identity for authentication
- **Status**: ✅ Working

**MCP Mode (Future)** — Direct MCP server integration
- Agent calls Language services via MCP protocol
- Requires Azure AI Agent Service to support Streamable HTTP transport
- **Status**: ⏸️ Blocked by transport incompatibility (Agent Service uses SSE, Language MCP requires POST)

Mode is controlled by `LANGUAGE_TOOL_MODE` environment variable (defaults to `sdk`).

## Project Structure

```
sentiment-analysis/
├── infra/
│   ├── main.bicep            # Infrastructure-as-Code (subscription scope)
│   ├── resources.bicep       # All Azure resources (AI Services, Language, etc.)
│   └── resources.bicepparam  # Parameter values
├── src/
│   ├── app.py               # Streamlit web UI + file handling + Fabric queries
│   ├── create_agent.py      # Agent provisioning (Language + Fabric tools)
│   ├── language_tools.py    # Language SDK function tool implementations
│   └── test_sdk.py          # End-to-end test for SDK mode
├── .env.example             # Environment variable template
├── agent_config.json        # Auto-generated agent config (ID, endpoint, mode)
├── azure.yaml               # Azure Developer CLI config
├── deploy.ps1               # Automated deployment script
└── README.md                # This file
```

## Infrastructure

Provisioned by `infra/resources.bicep` — all resources use **managed identity**:

| Resource | Type | Purpose |
|---|---|---|
| **Azure AI Services** | `Microsoft.CognitiveServices/accounts` (AIServices) | Foundry host + project container |
| **Foundry Project** | `accounts/projects` | `sentiment-analysis` project |
| **GPT-4o** | `accounts/deployments` | Agent's LLM (50K TPM capacity) |
| **Azure AI Language** | `accounts` (TextAnalytics) | NLP APIs for sentiment, NER, key phrases, PII |
| **Log Analytics Workspace** | `Microsoft.OperationalInsights/workspaces` | Centralized logging and monitoring backend |
| **Application Insights** | `Microsoft.Insights/components` | Agent telemetry, request tracking, and performance monitoring |
| **Fabric Connection** | `accounts/projects/connections` | Connection to Microsoft Fabric Data Agent (optional) |

### Role Assignments

- AI Services → Language Service: `Cognitive Services User`
- Foundry Project → Language Service: `Cognitive Services User`

### Monitoring & Telemetry

The application integrates with **Azure Application Insights** to provide:
- **Request Tracing**: Track agent message processing end-to-end
- **Tool Execution Metrics**: Monitor Language SDK tool calls (sentiment analysis, key phrase extraction, etc.)
- **File Analysis Tracking**: Monitor survey file uploads and processing
- **Performance Insights**: Response times, error rates, and usage patterns

Telemetry is automatically sent when `APPLICATIONINSIGHTS_CONNECTION_STRING` is configured.

## Azure AI Language Capabilities

The agent has access to these Language SDK functions:

| Function | API | Description |
|---|---|---|
| `analyze_sentiment` | Sentiment Analysis | Document & sentence-level sentiment with confidence scores |
| `extract_key_phrases` | Key Phrase Extraction | Identify main topics and themes |
| `recognize_entities` | Named Entity Recognition | Detect people, places, organizations, dates |
| `detect_language` | Language Detection | Identify the language of text |
| `recognize_pii_entities` | PII Detection | Detect personal data (email, phone, SSN, etc.) |

All functions support batching up to 10 documents per call for efficiency.

## Prerequisites

**Required:**
- Azure subscription with appropriate credits (GPT-4o deployment required)
- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) (`az`)
- [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (`azd`)
- Python 3.10 or later

**Install prerequisites on Windows:**
```powershell
winget install Microsoft.AzureCLI
winget install Microsoft.Azd
winget install Python.Python.3.11
```

## Quick Start

### Option A: Automated Deployment (Recommended)

```powershell
# 1. Clone the repository
git clone https://github.com/katkostro/sentiment-analysis.git
cd sentiment-analysis

# 2. Authenticate to Azure
az login

# 3. Run automated deployment script
./deploy.ps1

# 4. Start the UI
streamlit run src/app.py
```

The script will:
- Provision all Azure infrastructure
- Create the `.env` file automatically
- Install Python dependencies
- Create the AI agent

### Option B: Manual Step-by-Step

```powershell
# 1. Clone and navigate to repository
git clone https://github.com/katkostro/sentiment-analysis.git
cd sentiment-analysis

# 2. Authenticate to Azure
az login

# 3. Deploy infrastructure (will prompt for environment name and region)
azd up

# 4. Create environment variables file
azd env get-values | Out-File -FilePath .env -Encoding utf8

# 5. Install Python dependencies
pip install -r src/requirements.txt

# 6. Create the agent
python src/create_agent.py

# 7. Start the Streamlit UI
streamlit run src/app.py
```

**Infrastructure deployed:**
- Azure AI Services + Foundry Project
- GPT-4o deployment (50K TPM capacity)
- Azure AI Language service
- Log Analytics Workspace + Application Insights
- RBAC role assignments for managed identity

Open http://localhost:8501 in your browser.

## Usage

### Option 1: Analyzing Local Files

1. **Upload File**
   - Select **"Local File"** in the sidebar
   - Click "Browse files" and select an Excel (.xlsx, .xls) or CSV file
   - File is parsed automatically

2. **Select Columns**
   - Choose 1-2 columns to analyze
   - App auto-selects the most likely response column
   - Preview shows sample values from each selected column

3. **Analyze**
   - Click "Analyse File"
   - Agent processes up to 100 responses (first 100 if file is larger)
   - Results are presented in a structured format:
     1. **Customer Sentiment Overview** — Executive summary with key insights
     2. **Where Sentiment Breaks Down** — Table showing sentiment percentages by theme (Technical Support Quality, Communication, Tools, Documentation, Product Features, etc.)
     3. **Key Drivers of Negative Sentiment** — Top 5 recurring issues with their share of negative impact
     4. **Key Drivers of Positive Sentiment** — Top strengths customers consistently praise
     5. **Insight-Driven Recommendations** — Numbered, prioritized recommendations with reasoning and specific actions

### Multi-Column Analysis

When analyzing 2 columns, the agent will:
- Label responses with `[ColumnName]` prefix
- Identify patterns in each column separately
- Highlight differences in sentiment or themes between columns
- Note if certain columns have more negative feedback

### Option 2: Querying Microsoft Fabric

When a Fabric connection is configured, the sidebar shows a **"Fabric Semantic Model"** option:

1. **Select Data Source**
   - Choose **"Fabric Semantic Model"** in the sidebar

2. **Enter Query**
   - Type a natural language query in the text area
   - Examples:
     - "Get all survey responses"
     - "Show responses where satisfaction rating is below 3"
     - "Get the first 20 responses from the Comments column"
     - "What tables and columns are available?"

3. **Query & Analyze**
   - Click "Query & Analyze"
   - The Foundry Agent calls the Fabric Data Agent to retrieve data
   - Then runs Language analysis tools on the results
   - Presents the same structured 5-section report

### Chat Interface

After analysis (from either data source), you can:
- Ask follow-up questions about the results
- Request specific breakdowns (e.g., "show only negative responses")
- Get clarifications on themes or recommendations

## File Format Support

### Supported Formats

- **Excel**: `.xlsx` (Office Open XML, preferred)
- **Excel**: `.xls` (Legacy format, requires `xlrd`)
- **CSV**: UTF-8, Latin-1, or CP1252 encoding
- **HTML**: Tables in HTML files

### File Handling

- **DRM Detection**: Detects password-protected or DRM-encrypted Excel files
- **Column Detection**: Auto-selects columns with longest text (skips ID columns like "ID", "Code", "Date")
- **Encoding Fallback**: Tries UTF-8 → Latin-1 → CP1252 for CSV files
- **Size Limit**: Processes first 100 responses to avoid rate limits and long runtimes

## Authentication

All authentication uses **Azure Managed Identity** (`DefaultAzureCredential`):

- **No API keys** — `disableLocalAuth: true` on all cognitive services
- **Service-to-service** — RBAC role assignments grant permissions
- **Local development** — Falls back to Azure CLI or VS Code credentials

When running locally, ensure you're logged in:
```powershell
az login
```

## Error Handling

### Rate Limiting

- GPT-4o deployment capacity: 50K TPM (tokens per minute)
- App retries with exponential backoff on rate limit errors
- Parses retry-after from API responses
- Shows progress toasts in UI

### Run Management

- Cancels active runs before posting new messages (prevents "run is active" errors)
- 5-minute timeout for agent runs
- Polls run status with progressive backoff

### File Errors

- DRM-protected files: Shows user guidance to save unprotected copy
- Column detection failure: Shows all columns for manual selection
- Empty columns: Warns user before analysis

## Known Issues

### MCP Transport Incompatibility

**Issue**: Azure AI Agent Service uses SSE (Server-Sent Events) transport for MCP, but Azure Language MCP server requires Streamable HTTP (POST-only).

**Impact**: MCP mode (`LANGUAGE_TOOL_MODE=mcp`) is currently non-functional.

**Workaround**: SDK mode (`LANGUAGE_TOOL_MODE=sdk`) executes Language tools locally via the Python SDK.

**Resolution**: Waiting for Agent Service to add Streamable HTTP support (preview feature).

## Configuration

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `AZURE_AI_SERVICES_ENDPOINT` | Yes | AI Services endpoint (includes Foundry) |
| `AZURE_LANGUAGE_ENDPOINT` | Yes | Language service endpoint |
| `FOUNDRY_PROJECT_NAME` | No | Project name (default: `sentiment-analysis`) |
| `GPT_DEPLOYMENT_NAME` | No | GPT deployment name (default: `gpt-4o`) |
| `LANGUAGE_TOOL_MODE` | No | Tool mode: `sdk` or `mcp` (default: `sdk`) |
| `FABRIC_CONNECTION_NAME` | No | Foundry project connection name for Microsoft Fabric Data Agent |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Application Insights connection string for telemetry |

### Microsoft Fabric Setup

To enable the Fabric Semantic Model data source:

1. **Deploy a Fabric Data Agent** in your Microsoft Fabric workspace
   - Create a semantic model (e.g., `rsa-survey`) with your survey data
   - Create and publish a Data Agent that connects to the semantic model

2. **Create a Fabric connection in the Foundry portal**
   - Go to [AI Foundry](https://ai.azure.com) → your project → **Management Center** → **Connected Resources**
   - Click **+ New connection** → select **Microsoft Fabric**
   - Enter your Fabric **Workspace ID** and **Artifact ID** (the Data Agent)
   - Set authentication to **User identity** (On-Behalf-Of)
   - Save and note the connection name

3. **Set the connection name in `.env`**
   ```
   FABRIC_CONNECTION_NAME="your-connection-name"
   ```

4. **Recreate the agent** to pick up the Fabric tool
   ```powershell
   python src/create_agent.py
   ```

> **Note:** Your Fabric workspace and Foundry project must be in the same Entra ID tenant for identity passthrough to work.

### Agent Configuration

`agent_config.json` is created by `create_agent.py` and contains:

```json
{
  "agent_id": "asst_...",
  "agent_name": "sentiment-analysis-agent",
  "endpoint": "https://...",
  "model": "gpt-4o",
  "tool_mode": "sdk"
}
```

## Testing

### Test SDK Mode

```powershell
python src/test_sdk.py
```

This runs an end-to-end test:
1. Creates a thread
2. Sends a test message
3. Waits for agent to call SDK tools
4. Executes tools locally
5. Returns results

Expected output: Sentiment analysis + key phrases for test text.

## Cost Considerations

### Azure Resources

- **Azure AI Services**: AI Services SKU
- **GPT-4o Deployment**: ~$0.03 per 1K input tokens, ~$0.06 per 1K output tokens
- **Language Service**: Standard tier, ~$2 per 1K text records

### Optimization Tips

- Process files in batches (app limits to 100 responses)
- Use multi-column analysis sparingly (doubles API calls)
- Monitor GPT-4o capacity (50K TPM deployed by default)
- Consider lower GPT capacity if budget-constrained (edit `infra/resources.bicep`)

## Development

### Project Stack

- **Frontend**: Streamlit 1.32+
- **Agent SDK**: `azure-ai-agents` 1.2.0b5
- **Language SDK**: `azure-ai-textanalytics` 5.3+
- **Auth**: `azure-identity` 1.15+
- **IaC**: Bicep + Azure Developer CLI

### Adding New Language Capabilities

1. Add function definition to `language_tools.py`:
   ```python
   TOOL_DEFINITIONS.append({
       "type": "function",
       "function": {
           "name": "your_function",
           "description": "...",
           "parameters": { ... }
       }
   })
   ```

2. Implement function in `TOOL_DISPATCH` dict

3. Update agent instructions in `create_agent.py`

4. Recreate agent: `python src/create_agent.py`

## Troubleshooting

### "No assistant found with id 'asst_...'"

**Cause**: Agent was deleted or `agent_config.json` is stale.

**Fix**: Recreate agent:
```powershell
python src/create_agent.py
```
Then restart Streamlit app.

### "Rate limit exceeded"

**Cause**: GPT-4o capacity (50K TPM) exceeded.

**Fix**: 
- App retries automatically with backoff
- Wait for rate limit window to reset
- Or increase capacity in `infra/resources.bicep` (edit `gptCapacity`)

### File Upload Issues

**Cause**: DRM protection, unsupported format, or encoding issues.

**Fix**:
- For DRM files: Save unprotected copy in Excel
- For encoding: Convert CSV to UTF-8
- For unsupported formats: Convert to .xlsx or .csv

### Agent Not Responding

**Cause**: Active run blocking the thread.

**Fix**: App auto-cancels active runs. If issue persists:
```powershell
# Create new conversation
# Click "🗑️ New Conversation" in sidebar
```

## License

MIT

## Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Test changes with `src/test_sdk.py`
4. Submit a pull request

## Support

For issues or questions:
- File an issue on GitHub
- Check Azure AI Foundry documentation
- Review Azure Language service docs
