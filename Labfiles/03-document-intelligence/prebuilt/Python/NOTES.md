# Notes: `document-analysis.py` (prebuilt invoice model)

**Exercise:** [Instructions/Exercises/03-document-intelligence.md](../../../../Instructions/Exercises/03-document-intelligence.md), *Extract data with Azure Document Intelligence* (section "Analyze an invoice with a prebuilt model using the Python SDK")

**Approach used in this lab:** call **Azure Document Intelligence** directly with `azure-ai-documentintelligence`, using the **prebuilt-invoice** model on a PDF at a public URL.

**The other approach:** expose invoice extraction to a **Foundry agent** as a tool (an MCP server, an OpenAPI tool for the DI REST API, or a function tool), or index extracted invoices in Azure AI Search and give the agent `AzureAISearchTool` or a Foundry IQ knowledge base over `MCPTool`. A newer service alternative is **Content Understanding's `prebuilt-invoice` analyzer**.

> 🎯 marks an item that maps to the **AI-103** skills outline ([study guide, skills measured as of April 16, 2026](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)).

---

## 1. Client: `DocumentIntelligenceClient(endpoint, credential=AzureKeyCredential(key))`

| | |
|---|---|
| **Library** | `azure-ai-documentintelligence==1.0.2` (the GA API `2024-11-30`; this replaces the older `azure-ai-formrecognizer`, which is why the app still prints "Forms Recognizer") |
| **`endpoint`** | `https://<resource>.cognitiveservices.azure.com/` |
| **`credential`** | **Key-based** here. The keyless alternative is `DefaultAzureCredential()` with the *Cognitive Services User* role. |
| **REST equivalent** | Header `Ocp-Apim-Subscription-Key: <key>` (or a bearer token) |

🎯 *Manage, monitor, and secure → "…**keyless credentials**, and role policies"* (this lab is the key-based variant)

## 2. Analyze: `begin_analyze_document(model_id, AnalyzeDocumentRequest(url_source=...), locale=...)`

| Parameter | Purpose |
|---|---|
| `model_id` | `"prebuilt-invoice"`. Other prebuilt models include `prebuilt-read` (OCR), `prebuilt-layout` (tables, structure, markdown), `prebuilt-receipt`, `prebuilt-idDocument` and `prebuilt-businessCard`, or a **custom model ID** (see `../../custom`). |
| `body` | `AnalyzeDocumentRequest(url_source=...)` for a URL, or `bytes_source=<bytes>` (or a file stream) for a local file. |
| `locale` | `"en-US"`, a hint for date and currency parsing. |
| Optional | `pages="1-3"`, `features=["queryFields", "keyValuePairs", "ocrHighResolution"]`, `query_fields=["PONumber"]` (ad-hoc extra fields), `output_content_format="markdown"` (RAG-friendly output) |
| Returns | An `AnalyzeDocumentLROPoller`. `.result()` is an `AnalyzeResult`. |

| | |
|---|---|
| **REST equivalent** | `POST {endpoint}/documentintelligence/documentModels/prebuilt-invoice:analyze?api-version=2024-11-30&locale=en-US` with body `{"urlSource": "https://..."}` → `202` plus `Operation-Location` → `GET {endpoint}/documentintelligence/documentModels/prebuilt-invoice/analyzeResults/{resultId}?api-version=2024-11-30` until `"status": "succeeded"` |

🎯 *Information extraction → "Extract information by using multimodal pipelines that combine **OCR, layout analysis, and field extraction**"*

## 3. Read the fields: `result.documents[].fields`

| Field | Accessor | Notes |
|---|---|---|
| `VendorName`, `CustomerName` | `.get("valueString")`, `.get("confidence")` | Each field is a `DocumentField` with type-specific value keys. |
| `InvoiceTotal` | `.get("valueCurrency")` → `{amount, currencySymbol, currencyCode}` | Currency fields are parsed into amount and symbol. |
| Others | `InvoiceDate` (`valueDate`), `Items` (`valueArray` of objects), `DueDate`, `InvoiceId`, … | Schema documented per prebuilt model |

**Why confidence matters:** route low-confidence fields (for example, under 0.8) to **human review** rather than trusting them automatically.

🎯 *Responsible AI → "Implement auditing through trace logging, **provenance metadata**, and **approval workflows**"*. Confidence plus bounding regions are the provenance, and low-confidence values go through human approval.
🎯 *Information extraction → "Produce clean, grounded representations to use with agents and RAG…"*. Use `output_content_format="markdown"`, or CU, for the RAG text.

## 4. Agent + MCP equivalent (sketch)

```python
# MCP server tool wrapping this call:  extract_invoice(url) -> {"vendor":..., "total":..., "confidence":...}
agent = project.agents.create_version(agent_name="ap-clerk", definition=PromptAgentDefinition(
    model="gpt-4.1",
    instructions="Extract invoices with the tool. If any field confidence < 0.8, ask the user to confirm.",
    tools=[MCPTool(server_label="docintel", server_url=DI_MCP_URL, require_approval="never",
                   allowed_tools=["extract_invoice"])]))
```

🎯 *Build agents → "Integrate agent tools, including APIs…"* · *"Build autonomous or semiautonomous workflows with **safeguards and approval flow controls**"*

## References

- [Document Intelligence prebuilt invoice model](https://learn.microsoft.com/azure/ai-services/document-intelligence/prebuilt/invoice)
- [Document Intelligence REST/SDK quickstart](https://learn.microsoft.com/azure/ai-services/document-intelligence/quickstarts/get-started-sdks-rest-api)
- [AI-103 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)
