# Notes: `knowledge/python/search-app.py`

**Exercise:** none references this folder directly. [04-knowledge-mining](../../../Instructions/Exercises/04-knowledge-mining.md) and [05-rag-pipeline](../../../Instructions/Exercises/05-rag-pipeline.md) only download [`../documents.zip`](../documents.zip) from here. The client that exercise 04 actually uses is [`../../04-knowledge-mining/search-app.py`](../../04-knowledge-mining/search-app.py).

**What this file is:** an earlier version of the same search client, written for the field names that the Azure portal's **Import data** wizard creates by default:

| This file | Lab 04 version | Where the field comes from |
|---|---|---|
| `metadata_storage_name` | `title` | Blob indexer metadata (the file name) vs a mapped field |
| `people` | `persons` | Entity Recognition skill output name |
| `keyphrases` | `keyPhrases` | Key Phrase Extraction skill output name |

Use this version if you built the index with the wizard's defaults, and the lab 04 version if you followed exercise 04's field names.

**Code change on this branch:** `requirements.txt` only listed `dotenv`, so `from azure.search.documents import SearchClient` failed. It now also pins `azure-search-documents==11.6.0`, the same version lab 04 uses. The Python file is unchanged.

> 🎯 marks an item that maps to the **AI-103** skills outline ([study guide, skills measured as of April 16, 2026](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)).

---

## Calls

| Call | Purpose and key parameters | REST equivalent |
|---|---|---|
| `SearchClient(search_endpoint, index, AzureKeyCredential(query_key))` | A read-only **query key** is least privilege for a search client. The keyless alternative is `DefaultAzureCredential()` with the *Search Index Data Reader* role. | Header `api-key: <query key>` |
| `search_client.search(search_text=..., select=["metadata_storage_name", "locations", "people", "keyphrases"], order_by=["metadata_storage_name"], include_total_count=True)` | Full-text query that returns the **AI-enriched** fields, sorted, with a total count | `POST {endpoint}/indexes/{index}/docs/search?api-version=2024-07-01` with `{"search": "...", "select": "metadata_storage_name,locations,people,keyphrases", "orderby": "metadata_storage_name", "count": true}` |
| `found_documents.get_count()` and iterating results as dicts | Total hits and per-document fields | — |

**Agent + MCP equivalent:** give a Foundry agent the same index with `AzureAISearchTool` (a project connection plus the index name), or put it behind a **Foundry IQ knowledge base** and attach it as an `MCPTool` (`allowed_tools=["knowledge_base_retrieve"]`). The agent then writes cited answers instead of listing documents. The lab 04 [NOTES.md](../../04-knowledge-mining/NOTES.md) covers this in full.

🎯 *Information extraction → "Implement **enrichment** by using custom or built-in skills for text, images, and layout"* (`people`, `locations`, `keyphrases`)
🎯 *Information extraction → "**Ingest and index content**, such as documents…"* (blob indexer → `metadata_storage_name`)
🎯 *Information extraction → "Configure semantic search, hybrid search, and vector search for grounding"* (keyword search here; semantic and vector are the next steps)
🎯 *Manage, monitor, and secure → "…keyless credentials, and **role policies**"* (query key vs RBAC)

## References

- [Azure AI Search: Import data wizard](https://learn.microsoft.com/azure/search/search-import-data-portal)
- [AI-103 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)
