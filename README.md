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
- 💬 **Interactive chat** — Ask follow-up questions and get detailed analysis
- 📈 **Comprehensive metrics** — Sentiment distribution, themes, key phrases, entities
- 🔒 **Zero API keys** — All auth via managed identity
- ⚡ **Batch processing** — Analyzes up to 100 responses per file

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Getting Started](#getting-started)
3. [Adding Microsoft Fabric (Optional)](#adding-microsoft-fabric-optional)
4. [Usage](#usage)
5. [Architecture](#architecture)
6. [Infrastructure](#infrastructure)
7. [Configuration Reference](#configuration-reference)
8. [Development](#development)
9. [Troubleshooting](#troubleshooting)
10. [Cost Considerations](#cost-considerations)

---

## 1. Prerequisites

- Azure subscription with appropriate credits
- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) (`az`)
- [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (`azd`)
- Python 3.10+

```powershell
# Install on Windows
winget install Microsoft.AzureCLI
winget install Microsoft.Azd
winget install Python.Python.3.11
```

---

## 2. Getting Started

### Option A: Automated (Recommended)

```powershell
git clone https://github.com/katkostro/sentiment-analysis-fabric.git
cd sentiment-analysis-fabric
az login
./deploy.ps1
streamlit run src/app.py
```

The script provisions all Azure infrastructure, creates the `.env` file, installs Python dependencies, and creates the AI agent.

### Option B: Manual Step-by-Step

```powershell
# 1. Clone and authenticate
git clone https://github.com/katkostro/sentiment-analysis-fabric.git
cd sentiment-analysis-fabric
az login

# 2. Deploy infrastructure (prompts for environment name and region)
azd up

# 3. Create .env file from deployed resources
azd env get-values | Out-File -FilePath .env -Encoding utf8

# 4. Install dependencies
pip install -r src/requirements.txt

# 5. Create the AI agent
python src/create_agent.py

# 6. Launch the UI
streamlit run src/app.py
```

Open http://localhost:8501 in your browser.

---

## 3. Adding Microsoft Fabric (Optional)

Fabric integration lets you query survey data directly from a Fabric semantic model using natural language. Skip this section if you only need local file analysis.

### Step 1 — Create Fabric artifacts in the Fabric portal

1. Go to [Microsoft Fabric](https://app.fabric.microsoft.com)
2. Open or create a **workspace**
3. **Import your survey data** into a Lakehouse or Data Warehouse
4. **Create a semantic model** from the data:
   - **+ New** → **Semantic model** → select the survey table(s) → Publish
5. **Create a Data Agent**:
   - **+ New** → **Data Agent** → select your semantic model → test with sample queries → **Publish**
6. **Note your IDs** from the Fabric URL:
   - **Workspace ID** — the GUID after `/workspaces/`
   - **Artifact ID** — the GUID after `/dataagents/`

### Step 2 — Deploy with Fabric enabled

```powershell
azd env set AZURE_ENABLE_FABRIC true
azd env set AZURE_FABRIC_WORKSPACE_ID "<workspace-id>"
azd env set AZURE_FABRIC_ARTIFACT_ID "<artifact-id>"
azd env set AZURE_FABRIC_ADMIN_EMAIL "your-email@example.com"
azd up
```

This provisions a **Fabric capacity** (F2 SKU) and a **Foundry connection** linking the project to your Fabric Data Agent.

### Step 3 — Regenerate config and recreate the agent

```powershell
azd env get-values | Out-File -FilePath .env -Encoding utf8
python src/create_agent.py
```

> **Note:** Your Fabric workspace and Foundry project must be in the same Entra ID tenant for identity passthrough to work.

---

## 4. Usage

### Analyzing Local Files

1. Select **"Local File"** in the sidebar
2. Upload an Excel (`.xlsx`, `.xls`) or CSV file
3. Choose 1-2 columns to analyze (app auto-selects the most likely response column)
4. Click **"Analyse File"**

The agent processes up to 100 responses and returns a structured report:

| Section | Content |
|---------|---------|
| **Customer Sentiment Overview** | Executive summary with key insights |
| **Where Sentiment Breaks Down** | Sentiment percentages by theme |
| **Key Drivers of Negative Sentiment** | Top 5 recurring issues |
| **Key Drivers of Positive Sentiment** | Top strengths customers praise |
| **Insight-Driven Recommendations** | Prioritized actions with reasoning |

**Multi-column analysis:** When 2 columns are selected, the agent labels responses with `[ColumnName]`, analyzes each column separately, and highlights differences in sentiment between them.

### Querying Microsoft Fabric

*(Requires Fabric setup from [Section 3](#adding-microsoft-fabric-optional))*

1. Select **"Fabric Semantic Model"** in the sidebar
2. Type a natural language query, e.g.:
   - *"Get all survey responses"*
   - *"Show responses where satisfaction rating is below 3"*
   - *"What tables and columns are available?"*
3. Click **"Query & Analyze"**

The Foundry Agent calls the Fabric Data Agent to retrieve data, runs Language analysis tools, and returns the same structured report.

### Chat

After any analysis, use the chat to ask follow-up questions, request specific breakdowns, or get clarifications.

### Supported File Formats

| Format | Notes |
|--------|-------|
| `.xlsx` | Office Open XML (preferred) |
| `.xls` | Legacy Excel (requires `xlrd`) |
| `.csv` | Auto-detects UTF-8, Latin-1, or CP1252 encoding |
| `.html` | Tables in HTML files |

- DRM-protected files are detected with user guidance to save an unprotected copy
- First 100 responses are processed per analysis

---

## 5. Architecture

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
| 1 | **User** | Uploads a file OR types a natural language query |
| 2 | **Streamlit UI** | Parses file / sends query to the Foundry Agent |
| 3 | **Foundry Agent** | For Fabric queries: calls `fabric_dataagent` to retrieve data |
| 4 | **Foundry Agent** | Calls Language tools (sentiment, key phrases, entities) on the data |
| 5 | **Language Service** | Returns NLP analysis results |
| 6 | **Foundry Agent** | Compiles structured 5-section analysis report |

### Language Tools

| Function | Description |
|----------|-------------|
| `analyze_sentiment` | Document & sentence-level sentiment with confidence scores |
| `extract_key_phrases` | Identify main topics and themes |
| `recognize_entities` | Detect people, places, organizations, dates |
| `detect_language` | Identify the language of text |
| `recognize_pii_entities` | Detect personal data (email, phone, SSN, etc.) |

All functions support batching up to 10 documents per call.

### Tool Modes

| Mode | Status | Description |
|------|--------|-------------|
| **SDK** (default) | ✅ Working | Language tools executed locally via Python SDK |
| **MCP** (future) | ⏸️ Blocked | Waiting for Agent Service to support Streamable HTTP transport |

Set via `LANGUAGE_TOOL_MODE` environment variable.

---

## 6. Infrastructure

All resources are provisioned by Bicep (`infra/resources.bicep`) using **managed identity** — no API keys.

### Resources

| Resource | Purpose |
|----------|---------|
| **Azure AI Services** | Foundry host + project container |
| **Foundry Project** | `sentiment-analysis` project |
| **GPT-4o Deployment** | Agent's LLM backbone |
| **Azure AI Language** | NLP APIs (sentiment, NER, key phrases, PII) |
| **Log Analytics Workspace** | Centralized logging backend |
| **Application Insights** | Agent telemetry and performance monitoring |
| **Diagnostic Settings** | Routes AI Services logs/metrics to Log Analytics |
| **Fabric Capacity** *(optional)* | F2 SKU compute (when `enableFabric = true`) |
| **Fabric Connection** *(optional)* | Links Foundry project to Fabric Data Agent |

### Role Assignments

| Source | Target | Role |
|--------|--------|------|
| AI Services | Language Service | `Cognitive Services User` |
| Foundry Project | Language Service | `Cognitive Services User` |

### Monitoring

Application Insights provides request tracing, tool execution metrics, and performance insights when `APPLICATIONINSIGHTS_CONNECTION_STRING` is configured. The `AIAgentsInstrumentor` enables Gen AI trace emission from the agents SDK.

---

## 7. Configuration Reference

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `AZURE_AI_SERVICES_ENDPOINT` | Yes | — | AI Services endpoint |
| `AZURE_LANGUAGE_ENDPOINT` | Yes | — | Language service endpoint |
| `FOUNDRY_PROJECT_NAME` | No | `sentiment-analysis` | Foundry project name |
| `GPT_DEPLOYMENT_NAME` | No | `gpt-4o` | GPT model deployment name |
| `LANGUAGE_TOOL_MODE` | No | `sdk` | Tool mode: `sdk` or `mcp` |
| `FABRIC_CONNECTION_NAME` | No | — | Fabric connection name in Foundry |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | — | App Insights connection string |

### Agent Configuration

`agent_config.json` is auto-generated by `create_agent.py`:

```json
{
  "agent_id": "asst_...",
  "agent_name": "sentiment-analysis-agent",
  "endpoint": "https://...",
  "model": "gpt-4o",
  "tool_mode": "sdk"
}
```

### Project Structure

```
sentiment-analysis/
├── infra/
│   ├── main.bicep              # IaC entry point (subscription scope)
│   ├── resources.bicep         # All Azure resources
│   └── resources.bicepparam    # Parameter values
├── src/
│   ├── app.py                  # Streamlit UI + agent interaction
│   ├── create_agent.py         # Agent provisioning script
│   ├── language_tools.py       # Language SDK tool implementations
│   └── test_sdk.py             # End-to-end SDK mode test
├── agent_config.json           # Auto-generated agent config
├── azure.yaml                  # Azure Developer CLI config
├── deploy.ps1                  # Automated deployment script
└── README.md
```

---

## 8. Development

### Tech Stack

| Component | Technology |
|-----------|-----------|
| Frontend | Streamlit 1.32+ |
| Agent SDK | `azure-ai-agents` 1.2.0b5 |
| Language SDK | `azure-ai-textanalytics` 5.3+ |
| Auth | `azure-identity` 1.15+ (DefaultAzureCredential) |
| IaC | Bicep + Azure Developer CLI |

### Adding New Language Tools

1. Add function definition to `language_tools.py` (`TOOL_DEFINITIONS`)
2. Implement the function in the `TOOL_DISPATCH` dict
3. Update agent instructions in `create_agent.py`
4. Recreate the agent: `python src/create_agent.py`

### Testing

```powershell
python src/test_sdk.py
```

Runs an end-to-end test: creates a thread, sends a test message, executes Language tools, and returns results.

---

## 9. Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| *"No assistant found with id 'asst_...'"* | Agent deleted or config stale | `python src/create_agent.py`, then restart Streamlit |
| *"Rate limit exceeded"* | GPT-4o capacity exceeded | App retries automatically; wait or increase `gptCapacity` in Bicep |
| File upload fails | DRM protection or encoding | Save unprotected copy; convert CSV to UTF-8 |
| Agent not responding | Active run blocking the thread | Click **"🗑️ New Conversation"** in the sidebar |
| App Insights empty | `.env` has BOM or missing instrumentor | Regenerate `.env`; ensure `AIAgentsInstrumentor` is enabled |

---

## 10. Cost Considerations

| Resource | Approximate Cost |
|----------|-----------------|
| GPT-4o | ~$0.03 / 1K input tokens, ~$0.06 / 1K output tokens |
| Language Service | ~$2 / 1K text records |
| Fabric Capacity (F2) | [See Fabric pricing](https://azure.microsoft.com/pricing/details/microsoft-fabric/) |

**Tips:** Process files in batches (app limits to 100 responses) · use multi-column analysis sparingly · monitor GPT-4o TPM usage · reduce `gptCapacity` in Bicep if budget-constrained.

---

## License

MIT

## Contributing

1. Fork the repository
2. Create a feature branch
3. Test changes with `python src/test_sdk.py`
4. Submit a pull request
