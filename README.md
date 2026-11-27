# Foundry IQ: End-to-End Agentic Retrieval with Azure AI Search

A demo repo showcasing **Foundry IQ**, the managed knowledge layer in Azure AI Search that turns enterprise content into reusable, permission-aware knowledge bases for agents. It goes from the basics of knowledge bases and agents through **Microsoft Retriever 1**, **topic graphs** and **entity graphs**.

![Foundry IQ learning path: the getting started track covers the knowledge layer and agent via MCP; the advanced retrieval track covers dataset preparation, Microsoft Retriever 1, topic graph and entity graph; the security track covers sensitivity labels](docs/images/learning-path.svg)

## Tracks at a glance

| Track | Folder | What you learn | Dataset |
|---|---|---|---|
| **Getting started** | [getting-started/](getting-started/) | Knowledge sources, knowledge bases, agentic retrieval, Foundry Agent integration via MCP | NASA Earth at Night + Microsoft Learn |
| **Advanced retrieval** | [advanced-retrieval/](advanced-retrieval/) | Microsoft Retriever 1, topic graphs, entity graphs | Wikipedia space exploration corpus |
| **Security** | [security/](security/) | Microsoft Purview sensitivity labels, enforced at query time | Sample Word documents |

Start with **Getting started**. The other two tracks are independent of each other. Each track's notebooks are meant to be run in the order listed below.

### Getting started

| Order | Notebook | What it covers |
|---|---|---|
| 1 | [knowledge-layer.ipynb](getting-started/knowledge-layer.ipynb) | Search index, two knowledge sources (SearchIndex + Microsoft Learn MCP server), knowledge base, agentic retrieval |
| 2 | [agent-mcp-integration.ipynb](getting-started/agent-mcp-integration.ipynb) | Connect the knowledge base to a Foundry Agent via MCP; multi-turn conversations with citations |

This track deliberately combines two *unrelated* sources, NASA Earth at Night (indexed) and Microsoft Learn (remote MCP), so the activity log clearly shows query planning **routing** subqueries to the right source.

### Advanced retrieval

| Order | Notebook | What it covers |
|---|---|---|
| 1 | [prepare-dataset.ipynb](advanced-retrieval/prepare-dataset.ipynb) | **Run once first.** Fetch ~230 Wikipedia articles, upload to Blob Storage, build an indexer + skillset pipeline that chunks and embeds server-side |
| 2 | [microsoft-retriever-1.ipynb](advanced-retrieval/microsoft-retriever-1.ipynb) | **Complete answers to multi-part questions** without an external planning model: a fact-checklist scoreboard of hybrid search vs Retriever 1 vs `auto` vs `medium`, with latency |
| 3 | [topic-graph.ipynb](advanced-retrieval/topic-graph.ipynb) | **Broader evidence for broad questions**: the same question with `none` / `balanced` / `diverse`, side by side, scored and written up as answers. Built with the Foundry IQ skills plugin (**manual step in GitHub Copilot CLI**) |
| 4 | [entity-graph.ipynb](advanced-retrieval/entity-graph.ipynb) | **Answers that join facts across documents**: plain RAG vs entity graph side by side; ontology design, graph-enabled Blob knowledge source with ingestion cache, ontology v1 → v2, GQL |

> ### ⚠️ To do before you run this track
> 1. **Run [prepare-dataset.ipynb](advanced-retrieval/prepare-dataset.ipynb) once.** Every other notebook in this track reads the `space-exploration` index it creates.
> 2. **For [topic-graph.ipynb](advanced-retrieval/topic-graph.ipynb), install GitHub Copilot CLI and the Foundry IQ skills plugin first.** Step 3 of that notebook isn't a code cell: you build the topic graph by pasting two prompts into Copilot CLI and approving each plan. Step 4 (the comparison) only produces results after that. The notebook walks you through it with a checklist.
> 3. **On WSL**, make sure `node`, `npm` and `az` are the *Linux* installs (`which node npm az` must not show `/mnt/c/...`) and install `libicu-dev`. Otherwise Copilot CLI or the plugin's Azure MCP server won't start. See [the guide](docs/foundry-iq-skills-guide.md#troubleshooting).

All four notebooks share one relationship-rich Wikipedia corpus on **space exploration**, so you can ask the *same* questions with plain search, Retriever 1, a topic graph and an entity graph and compare. It has many **themes** (Space Race, Apollo, space stations, Mars, telescopes, commercial spaceflight) for topic graphs, and many **relationships** (astronaut → mission → launch site → country) for entity graphs. After `prepare-dataset`, notebooks 2–4 can be run in any order.

See also [docs/foundry-iq-skills-guide.md](docs/foundry-iq-skills-guide.md): how to use the Foundry IQ skills plugin, and what you can and can't do without it.

### Security

| Notebook | What it covers |
|---|---|
| [sensitivity-labels.ipynb](security/sensitivity-labels.ipynb) | Index Microsoft Purview-labelled documents and enforce label-based access at query time |

This notebook is self-contained: it has its own prerequisites (Microsoft Purview label policies, labels applied to the sample files in Word) and its own `.env` variables, described inside the notebook.

## Prerequisites

The tables below cover the **Getting started** and **Advanced retrieval** tracks. The **Security** track lists its own prerequisites in the notebook.

### Azure resources

| Resource | Needed by | Notes |
|---|---|---|
| **Azure AI Search**, Basic tier or higher | All | Region must support [agentic retrieval](https://learn.microsoft.com/azure/search/search-region-support); `microsoft-retriever-1` needs **Japan East, North Europe, West Central US or West US 3** |
| Search **Standard Knowledge Retrieval** plan | `microsoft-retriever-1` | Required for Microsoft Retriever 1 |
| **Microsoft Foundry resource** (`services.ai.azure.com`) | All | LLM deployment (e.g. `gpt-5.4-mini`) and `text-embedding-3-large` |
| Chat deployment for graphs (`gpt-5.6-luna` recommended) | `topic-graph`, `entity-graph` | Entity extraction, graph query planning, topic naming. Falls back to `LLM_DEPLOYMENT` |
| **Microsoft Foundry project** | `agent-mcp-integration` | Agent integration |
| **Storage account**: general-purpose v2, **not** ADLS Gen2, with Blob and Table storage | `prepare-dataset`, `entity-graph` | Keyless (shared-key access can stay disabled). Blob **soft delete on** and **versioning off**: set once with `az storage account blob-service-properties update` (see `prepare-dataset`, Step 3) |
| **GitHub Copilot CLI**, Git, Node.js, Python 3.12–3.14, Azure CLI, Windows x64 or Linux x64/ARM64 | `topic-graph` | Foundry IQ skills plugin; see the [guide](docs/foundry-iq-skills-guide.md). On WSL use the Linux Node.js and Azure CLI, plus `libicu-dev` |

### Permissions and identity

| Scope | Role | Assign to | Purpose |
|-------|------|-----------|---------|
| Azure AI Search | Search Service Contributor | You | Create indexes, indexers, knowledge sources, knowledge bases |
| Azure AI Search | Search Index Data Contributor | You | Upload documents |
| Azure AI Search | Search Index Data Reader | You **and** the Foundry project managed identity | Query indexes and knowledge bases |
| Foundry resource | Cognitive Services User | Azure AI Search managed identity | Embedding skill, vectorizers, knowledge base models, graph extraction |
| Foundry resource | Cognitive Services OpenAI User | You | Local LLM calls in `entity-graph` (extraction demo) |
| Foundry project | Foundry User, Foundry Project Manager | You | Agents and project connections (`agent-mcp-integration`) |
| Storage account | Storage Blob Data Contributor | You | Create containers and upload articles (`prepare-dataset`) |
| Storage account | Storage Account Contributor | You | One-time soft delete / versioning settings (`prepare-dataset`, Step 3) |
| Storage account | Storage Blob Data Contributor, Storage Table Data Contributor | Azure AI Search managed identity | Indexer reads blobs; entity graph reads blobs and writes its ingestion cache |

> **Identity setup:** enable a **system-assigned managed identity** on the search service and give it **Cognitive Services User** on the Foundry resource and the storage roles above. Storage is accessed **without keys**: the notebooks use your Entra identity (`DefaultAzureCredential`), and the search service uses `ResourceId=…` connections with its managed identity. Only `AZURE_STORAGE_RESOURCE_ID` is needed in `.env`.

## Getting started

1. **Clone** this repo
2. **Create `.env`** from the template and fill in your values:
   ```bash
   cp .env.sample .env
   ```
3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```
4. **Sign in** (for `DefaultAzureCredential`):
   ```bash
   az login
   ```
5. **Run the notebooks** track by track, in the order listed above. In **Advanced retrieval**, run `prepare-dataset` once before the other notebooks, and complete the manual Copilot CLI step in `topic-graph` (Step 3) before its Step 4.

## API versions

The **Getting started** and **Advanced retrieval** notebooks use the same SDK: **`azure-search-documents==12.2.0b1`**, which targets the **`2026-11-01-preview`** REST API. That version is required for Microsoft Retriever 1, topic graphs, entity graphs, the ingestion cache and MCP server knowledge sources. The knowledge base MCP endpoint in `agent-mcp-integration` uses the same API version (`SEARCH_API_VERSION`). The **Security** notebook targets `2026-05-01-preview`; see the notebook for details.

## Data and licensing

The space exploration corpus is fetched from English Wikipedia at run time and cached in `data/space-exploration/cache/` (git-ignored). Wikipedia text is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); every indexed chunk keeps its `source_url` for attribution. The committed files are only the title lists and ontologies.

## Key references

- [Agentic retrieval overview](https://learn.microsoft.com/azure/search/agentic-retrieval-overview)
- [What is Foundry IQ?](https://learn.microsoft.com/azure/ai-foundry/agents/concepts/what-is-foundry-iq)
- [Use Microsoft Retriever 1](https://learn.microsoft.com/azure/search/agentic-retrieval-how-to-use-microsoft-retriever)
- [Knowledge graph features overview](https://learn.microsoft.com/azure/search/knowledge-graph-features-overview)
- [Configure topic graph retrieval](https://learn.microsoft.com/azure/search/agentic-knowledge-source-how-to-topic-graph)
- [Create and query an entity graph](https://learn.microsoft.com/azure/search/agentic-knowledge-source-how-to-entity-graph)
- [Install Foundry IQ skills](https://learn.microsoft.com/azure/search/agentic-retrieval-how-to-install-foundry-iq-skills)
- [MCP server knowledge source](https://learn.microsoft.com/azure/search/agentic-knowledge-source-how-to-mcp-server)
- [Connect Foundry IQ to Foundry Agent Service](https://learn.microsoft.com/azure/ai-foundry/agents/how-to/foundry-iq-connect)
