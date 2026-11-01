# Notes: `search-app.py`

**Exercise:** [Instructions/Exercises/04-knowledge-mining.md](../../Instructions/Exercises/04-knowledge-mining.md), *Knowledge mining with Azure AI Search*

This file ships **already complete**. The exercise builds the index in the Azure portal (the *Import data* wizard: a blob data source, a skillset with AI enrichment for locations, people and key phrases, an indexer and the `margies-index`), and then asks you to **review and run** this client. No code was changed.

**Approach used in this lab:** query the enriched index **directly** with `azure-search-documents` (`SearchClient.search`).

**The other approach:** give a **Foundry agent** the index as a knowledge tool, either with `AzureAISearchTool` (a project connection plus the index name) or through a **Foundry IQ knowledge base** exposed as an **MCP** endpoint (`MCPTool(..., allowed_tools=["knowledge_base_retrieve"])`). The agent then decides when to search and writes a grounded, cited answer.

> 🎯 marks an item that maps to the **AI-103** skills outline ([study guide, skills measured as of April 16, 2026](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)).

---

## 1. Client: `SearchClient(search_endpoint, index, AzureKeyCredential(query_key))`

| | |
|---|---|
| **Library** | `azure-search-documents==11.6.0` |
| **`search_endpoint`** | `https://<search-service>.search.windows.net` |
| **`index`** | The index name (`margies-index`). |
| **`AzureKeyCredential(query_key)`** | A **query key**, which is **read-only**: it can search but not modify the index. This is least privilege for a client app. The keyless alternative is `DefaultAzureCredential()` with the *Search Index Data Reader* role (role-based access must be enabled on the service). |
| **REST equivalent** | Header `api-key: <query key>` (or `Authorization: Bearer`, scope `https://search.azure.com/.default`) |

🎯 *Manage, monitor, and secure → "Configure security, including managed identity… keyless credentials, and **role policies**"* (query key versus admin key versus RBAC)

## 2. Query: `search_client.search(search_text, select, order_by, include_total_count)`

| Parameter | Purpose |
|---|---|
| `search_text` | A full-text query using the simple syntax by default (`query_type="full"` gives Lucene syntax). `"*"` matches everything. |
| `select` | Fields to return: `title`, `locations`, `persons` and `keyPhrases`. The last three are **AI-enriched** fields produced by the skillset's entity recognition and key-phrase skills. |
| `order_by` | `["title"]`. The field must be *sortable*. |
| `include_total_count` | Return the total hit count, read with `found_documents.get_count()`. |
| Optional | `filter="locations/any(l: l eq 'London')"` (OData), `facets=["locations"]`, `top`, `query_type="semantic"` with `semantic_configuration_name` (semantic ranker), `vector_queries=[...]` (vector or hybrid, see lab 05) |
| Returns | A `SearchItemPaged` iterator of dicts. |

| | |
|---|---|
| **REST equivalent** | `POST {endpoint}/indexes/{index}/docs/search?api-version=2024-07-01` with body `{"search": "London", "select": "title,locations,persons,keyPhrases", "orderby": "title", "count": true}` |
| **Agent equivalent** | `AzureAISearchTool(azure_ai_search=AzureAISearchToolResource(indexes=[AISearchIndexResource(project_connection_id=..., index_name="margies-index", query_type=AzureAISearchQueryType.SIMPLE)]))` in a `PromptAgentDefinition`. The agent issues the queries and cites results as `【message_idx:search_idx†source】`. |

🎯 *Information extraction → "Configure semantic search, hybrid search, and vector search for grounding"* (this lab is keyword search; semantic and vector are the next steps)
🎯 *Information extraction → "Implement **enrichment** by using custom or built-in skills for text, images, and layout"* (the skillset behind `locations`, `persons` and `keyPhrases`)
🎯 *Information extraction → "**Ingest and index content**, such as documents…"* (data source → skillset → indexer → index)
🎯 *Information extraction → "Connect retrieval pipelines directly to workflows and **agent tools**"*
🎯 *Plan and manage → "Monitor data ingestion quality, **search index health**, and relevance performance"* (indexer status and warnings, plus document count)

## References

- [Azure AI Search: query overview](https://learn.microsoft.com/azure/search/search-query-overview)
- [Azure AI Search tool for agents](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/ai-search)
- [Connect a Foundry IQ knowledge base to an agent](https://learn.microsoft.com/azure/foundry/agents/how-to/foundry-iq-connect)
- [AI-103 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)
