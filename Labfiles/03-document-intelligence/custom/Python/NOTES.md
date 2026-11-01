# Notes: `test-model.py` (custom model)

**Exercise:** [Instructions/Exercises/03-document-intelligence.md](../../../../Instructions/Exercises/03-document-intelligence.md), *Extract data with Azure Document Intelligence* (section "Train and test a custom model")

This file ships **already complete**. The exercise has you train a custom extraction model in Document Intelligence Studio (after `setup.sh` uploads training forms to storage), put its `MODEL_ID` in `.env`, and run this script. So no code was changed, only this explanation added.

**Approach used in this lab:** call your **custom-trained Document Intelligence model** directly through the SDK.

**The other approach:** a **Content Understanding custom analyzer**, which is schema-defined with no labelled training set (see `../../../02-content-understanding-api`), or wrapping the model as an **agent tool** (MCP/OpenAPI).

> 🎯 marks an item that maps to the **AI-103** skills outline ([study guide, skills measured as of April 16, 2026](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)).

---

## Calls in this file

| Call | Purpose | REST equivalent |
|---|---|---|
| `DocumentIntelligenceClient(endpoint, AzureKeyCredential(key))` | Client for your DI resource (key-based; `DefaultAzureCredential` is the keyless option) | Header `Ocp-Apim-Subscription-Key` |
| `begin_analyze_document(model_id, AnalyzeDocumentRequest(url_source=formUrl))` | Run the **custom model** on `test1.jpg`. `model_id` is the name you gave the model when training. | `POST {endpoint}/documentintelligence/documentModels/{modelId}:analyze?api-version=2024-11-30` with `{"urlSource": "..."}`, then poll `Operation-Location` |
| `poller.result()` | Wait for the LRO and return an `AnalyzeResult` | `GET .../analyzeResults/{resultId}` |
| `result.documents[]` → `.doc_type`, `.confidence` | `doc_type` is `<model_id>:<doc type>`. For a **composed** model (several custom models combined), this tells you which sub-model matched. | — |
| `document.fields.items()` → `field.get("valueString") or field.get("content")`, `field.get("confidence")` | Your **labelled** fields, with per-field confidence | — |

🎯 *Information extraction → "Extract information by using multimodal pipelines that combine OCR, layout analysis, and **field extraction**"*
🎯 *Plan and manage → "Choose an appropriate model for each task…"*. A custom DI model (template or neural, trained on about 5+ labelled samples, tied to fixed layouts) versus a CU analyzer (generative, schema-only, works across varied layouts) is that choice.
🎯 *Plan and manage → "Monitor model performance, drift…"*. Track the confidence distribution over time, since falling confidence signals new form layouts and the need to retrain.

## Training (done in Studio; SDK equivalent for reference)

```python
from azure.ai.documentintelligence import DocumentIntelligenceAdministrationClient
from azure.ai.documentintelligence.models import BuildDocumentModelRequest, AzureBlobContentSource
admin = DocumentIntelligenceAdministrationClient(endpoint, AzureKeyCredential(key))
poller = admin.begin_build_document_model(BuildDocumentModelRequest(
    model_id="my-model", build_mode="template",
    azure_blob_source=AzureBlobContentSource(container_url=SAS_URL)))
```

REST: `POST {endpoint}/documentintelligence/documentModels:build?api-version=2024-11-30`

🎯 *Plan and manage → "Integrate Foundry projects with **CI/CD** pipelines"* (scripted model builds)

## References

- [Custom models overview](https://learn.microsoft.com/azure/ai-services/document-intelligence/train/custom-model)
- [AI-103 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)
