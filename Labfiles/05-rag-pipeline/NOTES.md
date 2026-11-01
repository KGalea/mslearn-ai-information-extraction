# Notes: `create-analyzer.py`, `ingest-pipeline.py` and `rag-agent.py`

**Exercise:** [Instructions/Exercises/05-rag-pipeline.md](../../Instructions/Exercises/05-rag-pipeline.md), *Build an automated RAG ingestion pipeline*

All three files ship **already complete**. The exercise asks you to review and run them in turn, then add `tokyo-guide.txt` to see the watch mode pick it up. No code was changed.

**Approach used in this lab:** a **hand-built RAG pipeline** that calls each service directly:
**Content Understanding** (extract markdown, a summary and topics) → **chunk** → **Azure OpenAI embeddings** → **Azure AI Search** (vector index) → **hybrid retrieval** → **chat completion** with the retrieved context.

**The other approach:** let the platform own retrieval. A **Foundry agent** with `AzureAISearchTool`, or a **Foundry IQ knowledge base** through `MCPTool`, performs the retrieval and citation. Ingestion can also move server-side with Azure AI Search **indexers + skillsets + integrated vectorization**, instead of the polling script.

> 🎯 marks an item that maps to the **AI-103** skills outline ([study guide, skills measured as of April 16, 2026](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)).

---

## A. `create-analyzer.py`

| Call | Purpose | REST equivalent |
|---|---|---|
| `ContentUnderstandingClient(endpoint, AzureKeyCredential(key))` | CU client (key-based) | `Ocp-Apim-Subscription-Key` header |
| `begin_create_analyzer("rag_document_analyzer", resource={...}, allow_replace=True).result()` | Creates an analyzer on top of `prebuilt-document` that returns **markdown** plus two **generated** fields, `Summary` and `KeyTopics` (`method: "generate"`), using the `gpt-4.1` and `text-embedding-3-large` deployments | `PUT {endpoint}/contentunderstanding/analyzers/rag_document_analyzer?api-version=2025-11-01`, then poll `Operation-Location` |

🎯 *Information extraction → "Implement analyzers for generating structured or **markdown outputs for downstream reasoning** by using Content Understanding"*
🎯 *Information extraction → "Produce **clean, grounded representations** to use with agents and RAG…"*

---

## B. `ingest-pipeline.py`

| Step / call | Key parameters and why | REST equivalent |
|---|---|---|
| **Change detection:** `file_hash()` + `processed_files.json` manifest | SHA-256 of each file in `data/`, so only new or changed files are processed (incremental ingestion). `--watch` polls every 30 seconds, and `--reset` reprocesses everything. | — |
| **Index:** `SearchIndexClient.create_or_update_index(SearchIndex(fields, vector_search))` | Fields are `id` (key), `content` and `summary`/`key_topics` (searchable), `file_name` (filterable) and **`content_vector`** (`Collection(Single)`, **3072 dims** to match `text-embedding-3-large`, `vector_search_profile_name="vector-profile"`). `VectorSearch` uses an **HNSW** ANN algorithm. | `PUT {search}/indexes/rag-content-index?api-version=2024-07-01` |
| **Extract:** `cu_client.begin_analyze_binary(ANALYZER_ID, binary_input=bytes).result()` | Handles PDF, PNG, JPG, DOCX and TXT. `result.contents[].markdown` is the text (with OCR for scans), and `.fields` holds `Summary` and `KeyTopics`. | `POST {endpoint}/contentunderstanding/analyzers/{id}:analyzeBinary?api-version=2025-11-01` |
| **Chunk:** `chunk_content(text, max_chars=2000)` | Splits on paragraph boundaries into chunks of about 2,000 characters, with **no overlap**. That's simple, but answers that span a chunk boundary can be lost. Token-based chunking with 10–20% overlap is the common improvement. | — |
| **Embed:** `openai_client.embeddings.create(input=chunk, model=embedding_deployment)` | One call per chunk. `input` also accepts a **list**, so batching chunks cuts round trips and helps with rate limits. | `POST {endpoint}/openai/deployments/{dep}/embeddings?api-version=2024-06-01` with `{"input": "..."}` |
| **Index docs:** `search_client.upload_documents(documents=[...])` | Each chunk's `id` is the **deterministic** `md5(file::chunk)`, so re-ingesting a file overwrites its chunks instead of duplicating them. Check `r.succeeded` for each document. | `POST {search}/indexes/{index}/docs/index?api-version=2024-07-01` with `{"value": [{"@search.action": "upload", ...}]}` |
| **Clients:** `AzureOpenAI(azure_endpoint, api_key, api_version="2024-06-01")`, `SearchClient(..., AzureKeyCredential(search_key))` | Key-based throughout. In production, use `DefaultAzureCredential` with *Search Index Data Contributor* and *Cognitive Services OpenAI User*. | — |

🎯 *Information extraction → "**Ingest and index content**, such as documents, images, audio, and video"*
🎯 *Information extraction → "Configure **RAG ingestion flow**, including documents and using **OCR**"*
🎯 *Information extraction → "Configure semantic search, hybrid search, and **vector search** for grounding"*
🎯 *Plan and manage → "Choose an appropriate method for **retrieval and indexing**"* (HNSW, chunk size, embedding model and dimensions)
🎯 *Plan and manage → "Monitor **data ingestion quality**, search index health…"* (the `Skipped (no extractable content)` log line and per-document `succeeded`)
🎯 *Plan and manage → "Manage quotas, scaling, **rate limits**, and cost footprints…"* (per-chunk embedding calls hit TPM limits, so batch them)
🎯 *Manage, monitor, and secure → "…keyless credentials…"* (this lab is the key-based variant)

**Server-side alternative:** a Search **indexer** over blob storage, with a **skillset** (Document Layout or CU skill → Text Split skill → **AzureOpenAIEmbedding** skill) and an index **vectorizer** for query-time embedding. This replaces the script's extract, chunk, embed and upload steps, and adds scheduling and change tracking.
🎯 *Information extraction → "Implement enrichment by using custom or built-in skills for text, images, and layout"*

---

## C. `rag-agent.py`

| Call | Key parameters and why | REST equivalent |
|---|---|---|
| `openai_client.embeddings.create(input=question, model=embedding_deployment)` | Embed the **question** with the **same** model used at ingestion; otherwise the vectors aren't comparable. | `POST .../embeddings` |
| `VectorizedQuery(vector=..., k_nearest_neighbors=3, fields="content_vector")` | The k-NN vector part of the query. | `"vectorQueries": [{"kind": "vector", "vector": [...], "k": 3, "fields": "content_vector"}]` |
| `search_client.search(search_text=question, vector_queries=[vq], select=[...], top=3)` | **Hybrid search**: keyword (BM25) and vector results are fused with **Reciprocal Rank Fusion (RRF)**. Adding `query_type="semantic"` turns on the semantic re-ranker on top. | `POST {search}/indexes/{index}/docs/search?api-version=2024-07-01` with `{"search": "...", "vectorQueries": [...], "select": "content,file_name,summary", "top": 3}` |
| Prompt assembly (`generate_answer`) | Retrieved chunks go into the **system message** as `[Source: file]` blocks, with grounding rules: "Use only the information from the context… If the context doesn't contain enough… say so… Cite the source". If nothing is found, it says so instead of guessing. | — |
| `openai_client.chat.completions.create(model, messages, temperature=1, max_completion_tokens=800)` | ChatCompletions with a system message and the user question. `max_completion_tokens` (rather than `max_tokens`) is the parameter for reasoning-era models. A **lower temperature** (0–0.3) usually gives more faithful RAG answers. | `POST {endpoint}/openai/deployments/{dep}/chat/completions?api-version=2024-06-01` |

**Note on "agent":** despite the name, this is a single-shot **RAG chain**. It retrieves on every turn, keeps no conversation memory, and the model can't choose tools. A true agent decides *when* and *what* to retrieve.

🎯 *Build generative applications → "Implement **retrieval-augmented generation (RAG)** in an application"*
🎯 *Information extraction → "Configure… **hybrid search**, and vector search for grounding"*
🎯 *Optimize → "**Tune generation behavior**, such as prompt engineering and adjusting model parameters"* (grounding rules, temperature and max tokens)
🎯 *Build generative applications → "Evaluate models and apps, including **detecting fabrications**, relevance, quality, and safety"*. Groundedness and relevance evaluators (`azure-ai-evaluation`) score `answer` against `context`.
🎯 *Plan and manage → "Monitor model performance, drift, safety events, and **grounding quality**"*

---

## D. The same Q&A as a Foundry agent (built-in tool, or MCP)

```python
from azure.ai.projects.models import (PromptAgentDefinition, AzureAISearchTool, AzureAISearchToolResource,
                                      AISearchIndexResource, AzureAISearchQueryType, MCPTool)

tools = [AzureAISearchTool(azure_ai_search=AzureAISearchToolResource(indexes=[AISearchIndexResource(
            project_connection_id=SEARCH_CONNECTION_ID, index_name="rag-content-index",
            query_type=AzureAISearchQueryType.VECTOR_SEMANTIC_HYBRID)]))]
# OR, a Foundry IQ knowledge base (agentic retrieval) over MCP:
# tools = [MCPTool(server_label="knowledge-base",
#                  server_url=f"{SEARCH_ENDPOINT}/knowledgebases/{KB}/mcp?api-version=...",
#                  allowed_tools=["knowledge_base_retrieve"], require_approval="never",
#                  project_connection_id=SEARCH_CONNECTION)]

agent = project.agents.create_version(agent_name="travel-rag", definition=PromptAgentDefinition(
    model="gpt-4.1",
    instructions="Answer only from the knowledge tool; cite sources; say 'I don't know' otherwise.",
    tools=tools))
with project.get_openai_client(agent_name=agent.name) as openai:
    conv = openai.conversations.create()
    r = openai.responses.create(conversation=conv.id, input="What are the top attractions in Tokyo?")
```

| Concern | Hand-built chain (this lab) | Agent + Search tool / Foundry IQ MCP |
|---|---|---|
| Query embedding | Your code | Search **vectorizer** (index-side) |
| When to retrieve | Every turn | The model decides, and can run several or reformulated queries (agentic retrieval) |
| Citations | Prompted `[Source: …]` | Structured `url_citation` / `file_citation` annotations |
| Memory | None | `conversations` |
| Credentials | Keys in `.env` | **Project connection** (managed identity possible) |
| Observability | `print` | Foundry tracing of tool calls, tokens and latency |

🎯 *Plan and manage → "Choose appropriate memory, tool, and **knowledge integration services** for agent solutions"*
🎯 *Build agents → "Build agents that integrate **retrieval**, function-calling, and **conversation memory**"*
🎯 *Build agents → "Integrate agent tools, including APIs, knowledge stores, **search**…"*
🎯 *Information extraction → "Connect retrieval pipelines directly to workflows and **agent tools**"*
🎯 *Responsible AI → "Implement auditing through trace logging, **provenance metadata**…"* (citations)

## References

- [RAG and retrieval in Azure AI Search](https://learn.microsoft.com/azure/search/retrieval-augmented-generation-overview)
- [Hybrid search](https://learn.microsoft.com/azure/search/hybrid-search-overview)
- [Azure AI Search tool for agents](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/ai-search)
- [Connect a Foundry IQ knowledge base to an agent](https://learn.microsoft.com/azure/foundry/agents/how-to/foundry-iq-connect)
- [AI-103 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)
