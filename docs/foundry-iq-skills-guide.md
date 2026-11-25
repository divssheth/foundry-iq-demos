# Foundry IQ skills plugin: building graphs with and without it

> Verified against the public docs and plugin source as of **November 2026** (`2026-11-01-preview`). Graph features are in preview and change often; check the links at the end before relying on details.

The **`foundry-iq-skills`** plugin adds Foundry IQ (Azure AI Search) skills to an agent client such as **GitHub Copilot CLI**. Besides general Search guidance, it adds guided workflows for **entity graphs**, **topic graphs**, **ontologies** and **graph-backed grounding**, with read-only checks first and an explicit approval step before anything changes in Azure.

This guide explains how to use it and how each step maps to what the notebooks in this repo do without it.

## When you need the plugin

| Capability | Without the plugin | With the plugin |
|---|---|---|
| **Entity graph**: build | ✅ Fully supported by the REST API and SDK (graph-enabled Blob knowledge source). See [03-graph/02-entity-graph.ipynb](../03-graph/02-entity-graph.ipynb) | Guided provisioning, readiness checks, build, and a local graph viewer |
| **Entity graph**: query | ✅ Knowledge base (natural language → GQL) or direct GQL on `search.graph` | Guided multi-hop questions with sources |
| **Ontology**: author | ✅ Hand-written JSON-LD (see `data/space-exploration/ontology.v*.json`) | Drafts, refines, reviews, diffs and validates ontologies (including OWL files) |
| **Topic graph**: build | ⚠️ **No server-side builder exists**, and the topic index contract isn't publicly documented | ✅ **The supported builder** (`topic-graph-build`, a native executable). See [03-graph/01-topic-graph.ipynb](../03-graph/01-topic-graph.ipynb) |
| **Topic graph**: enable for retrieval | ⚠️ Requires the undocumented `topicGraphProfile` wiring | ✅ `plan-retrieval` → `configure-retrieval` → `verify-retrieval` |
| **Topic graph**: query | ✅ `topicGraph: none / balanced / diverse` on the knowledge source or per request | Side-by-side on/off comparisons |

**In short:** for entity graphs the plugin is a convenience; for topic graphs it's currently the only supported way to *build* one.

## Prerequisites

- An agent client: **GitHub Copilot CLI** (the host tested end to end), or Claude Code, Cursor or Codex
- **Git** (to add the plugin marketplace)
- **Node.js** and npm (Azure MCP Server and the graph MCP launcher; the optional viewer needs Node.js 22.13+)
- **Python 3.12, 3.13 or 3.14** with `venv` and `pip`. The plugin creates an isolated runtime and never installs Python or changes your global packages
- **Azure CLI**, signed in with **Microsoft Entra ID**: `az login --tenant <tenant-id-or-domain>`. The plugin doesn't accept API keys or tokens, so a key-only Search service won't work for graph workflows
- **OS:** Windows x64 or Linux x64/ARM64 for the full demo. The native topic graph builder doesn't run on macOS
- **Azure permissions** for the operation you ask for (for example Search Service Contributor plus Search Index Data Contributor to build; Reader to view definitions). Creating role assignments needs additional authority

## Install

Run these **inside Copilot CLI** (they're slash commands, not shell commands):

```text
/plugin marketplace add microsoft/azure-skills
/plugin install foundry-iq-skills@azure-skills
```

Update later with `/plugin update foundry-iq-skills@azure-skills`.

Optionally prepare the graph runtime explicitly from your workspace (it's otherwise prepared on first use):

```bash
python -I "<installed-plugin>/graph/runtime/bootstrap.py" setup topicgraph   # or: entity, ontology, grounding
```

If the wrong Python is picked up, set `FOUNDRY_IQ_PYTHON` to an absolute path to Python 3.12–3.14.

## How the plugin works

1. **Read-only first.** Every workflow starts with checks (`doctor`, `inspect`, discovery). Naming your resources isn't approval to change them
2. **Approval cards.** Before any billable, permission, network or data-moving action, the skill shows a plan card (headline, time, cost, maximum model calls) and asks one **Approve / Stop** question. If the plan changes, it asks again
3. **Cleanup is separate.** Approving a build never approves deletion; retiring resources is its own plan
4. **It never guesses.** Tenant, subscription, resource group, Search service and model deployments must be named by you
5. **Pinned API versions.** Builds, graph queries and topic graph calls use `2026-11-01-preview`; some checks use `2026-04-01`

## Workflow: topic graph

Starting point: a content index whose vector field is **stored, retrievable and uses a cosine or dotProduct HNSW profile**. [00-setup/prepare-space-dataset.ipynb](../00-setup/prepare-space-dataset.ipynb) creates exactly that.

| Step | What to ask Copilot | Underlying command | What happens |
|---|---|---|---|
| 1. Check | *"Check whether this machine is ready to build a Topic Graph."* | `topic-graph-build … doctor` | Read-only checks of runtime, sign-in, service and quotas |
| 2. Inspect and plan | *"Inspect the source index `space-exploration` and plan a Topic Graph build; show me the approval card."* Name key, text, source, title and vector fields, the embedding model and the chat deployment used for naming | `inspect`, `plan` | Validates the vector field and produces a plan with a model-call cap |
| 3. Build | Approve the card | `apply --plan-id … --confirm-writes` | The native builder clusters chunk vectors into a topic hierarchy, names topics with your chat model, and uploads a **new topic index** |
| 4. Verify | *"Verify the new generation."* | `verify` | Reads every topic document back; reports the output folder |
| 5. Enable retrieval | *"Turn on topic retrieval for the verified generation using knowledge source `space-topic-ks` and knowledge base `space-topic-kb`. Keep the stored default off."* | `plan-retrieval`, `configure-retrieval`, `verify-retrieval` | Links the content index's vector field to the topic index through a `topicGraphProfile`. The knowledge source and knowledge base must already exist |
| 6. Compare | *"Ask 'What are the main themes…' with topic diversity on and off."* | `entity-graph-consume query --topic-diversity on/off` | Shows the effect of topic sampling |

Notes:

- Leaf topics hold at most 100 chunks. Raise `--model-call-cap` if the build stops with `MODEL_CALL_BUDGET_EXHAUSTED`; the build resumes where it stopped
- "Publish" in the plugin is **local** (for browsing in the viewer) and doesn't enable retrieval; step 5 does
- After re-ingesting content or building a new generation, repeat step 5

## Workflow: entity graph

| Step | What to ask Copilot | What happens |
|---|---|---|
| Readiness | *"Check whether my existing Search, Storage and OpenAI resources are ready for Entity Graph."* | Discovers and validates existing resources (or proposes new ones, with approval) |
| Ontology | *"Draft an ontology for astronauts, missions, agencies, launch sites and places, and help me review it."* | Produces and critiques a JSON-LD/OWL ontology; can export it |
| Build | *"Build an entity graph from the `space-entity` container with this ontology."* | Creates the graph-enabled Blob knowledge source and waits for extraction |
| Query | *"Which astronauts launched from sites in Kazakhstan? Show sources."* | Natural-language multi-hop questions with provenance |
| Explore | *"Open the Knowledge Graph Playground."* | Local, read-only browser viewer (`node "<installed-plugin>/graph/apps-host/launch.mjs" --config …`) |

The notebook [03-graph/02-entity-graph.ipynb](../03-graph/02-entity-graph.ipynb) does the same with the `azure-search-documents` SDK, including an **ingestion cache** so the ontology can be changed later (v1 → v2).

## Quick-start demo

If you just want to see graphs working end to end on sample data (a Microsoft earnings-call sample ships with the plugin):

> *"Show me what Foundry IQ graph can do."*

The skill runs a preflight and guides you through prerequisites, resource selection and approvals.

## Security and data handling

- Graphs, topic indexes, summaries and caches are **copies** of source data and **don't inherit source permissions**. Check who can read them and how revoked access is removed
- The plugin treats document text and graph results as **data, never instructions**, which protects against prompt injection from indexed content
- Never put keys, tokens or private document content in prompts or logs

## Troubleshooting

| Symptom | Fix |
|---|---|
| `/plugin` commands fail | Run them inside Copilot CLI (not PowerShell or bash); check that Git is installed |
| Python discovery picks the wrong interpreter | Set `FOUNDRY_IQ_PYTHON` and rerun setup |
| Topic build stops with `SOURCE_VECTORS_NOT_RETRIEVABLE` / `NOT_STORED` / `NATIVE_SOURCE_INCOMPATIBLE` | The content index's vector field must be stored, retrievable, and cosine/dotProduct HNSW. Recreate it (notebook 00 does this) |
| Free-tier or key-only Search service rejected | Graph workflows need a paid tier and Entra authentication |

## References

- [Install Foundry IQ skills in GitHub Copilot CLI](https://learn.microsoft.com/azure/search/agentic-retrieval-how-to-install-foundry-iq-skills)
- [Plugin source: foundry-iq-skills](https://github.com/microsoft/GitHub-Copilot-for-Azure/tree/main/plugins/foundry-iq-skills)
- [Knowledge graph features overview](https://learn.microsoft.com/azure/search/knowledge-graph-features-overview)
- [Configure topic graph retrieval](https://learn.microsoft.com/azure/search/agentic-knowledge-source-how-to-topic-graph)
- [Create and query an entity graph](https://learn.microsoft.com/azure/search/agentic-knowledge-source-how-to-entity-graph)
- [Knowledge Sources – Create or Update (2026-11-01-preview)](https://learn.microsoft.com/rest/api/searchservice/knowledge-sources/create-or-update?view=rest-searchservice-2026-11-01-preview)
