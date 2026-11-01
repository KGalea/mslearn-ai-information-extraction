# Notes: `create-analyzer.py` and `read-card.py`

**Exercise:** [Instructions/Exercises/02-content-understanding-api.md](../../Instructions/Exercises/02-content-understanding-api.md), *Develop a Content Understanding client application*

**Approach used in this lab:** use the `azure-ai-contentunderstanding` SDK directly to **create a custom analyzer** from a JSON schema (`biz-card.json`), then **analyze** business-card images with it.

**The other approach:** give a **Foundry agent** the analyzer as a tool, through an MCP server, an OpenAPI tool for the CU REST API, or a function tool. Alternatively, ask a **multimodal LLM** to extract the same fields to JSON (`responses.create` with `input_image` and a JSON schema `text.format`).

> 🎯 marks an item that maps to the **AI-103** skills outline ([study guide, skills measured as of April 16, 2026](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)).

> ⚠️ **Install note:** `requirements.txt` only lists `python-dotenv`. The exercise installs the SDK on the command line: `pip install -r requirements.txt azure-ai-contentunderstanding`.

---

## 1. Client: `ContentUnderstandingClient(endpoint, credential=AzureKeyCredential(key))`

| | |
|---|---|
| **`endpoint`** | The Foundry resource endpoint, `https://<resource>.services.ai.azure.com/`. |
| **`credential`** | **Key-based** here (`KEY` in `.env`). The keyless alternative is `DefaultAzureCredential()` with the *Cognitive Services User* role, as used in the mslearn-ai-vision CU lab. Prefer keyless in production, and keep keys in Key Vault. |
| **REST equivalent** | Header `Ocp-Apim-Subscription-Key: <key>`, or `Authorization: Bearer <token>` |

🎯 *Manage, monitor, and secure → "Configure security, including managed identity… **keyless credentials**, and role policies"*. Recognise that this lab is the key-based variant.

---

## 2. The analyzer schema (`biz-card.json`)

| Key | Meaning |
|---|---|
| `baseAnalyzerId: "prebuilt-document"` | Start from the document analyzer (OCR, layout, markdown) and add custom fields on top. |
| `models: {completion, embedding}` | The **LLM and embedding deployments** that CU uses for field extraction. They must exist in the resource, and you map them during "Configure Content Understanding models". |
| `config.returnDetails: true` | Include confidence scores, spans and bounding regions in the result. |
| `fieldSchema.fields` | `Company`, `Name`, `Title`, `Email` and `Phone`. Each has `type`, `description` (which acts as the prompt for that field) and **`method`**. |
| `method` | `"extract"` takes the value as it appears in the content (grounded, with a source span). `"generate"` lets the model produce it (summaries, tags). `"classify"` picks from an `enum`. |

🎯 *Information extraction → "Extract information by using multimodal pipelines that combine **OCR, layout analysis, and field extraction**"*
🎯 *Information extraction → "Implement **analyzers** for generating structured or markdown outputs for downstream reasoning by using Content Understanding"*

## 3. Create: `client.begin_create_analyzer(analyzer_id, resource=dict, allow_replace=True)` → `poller.result()`

| Parameter | Purpose |
|---|---|
| `analyzer_id` | Your name for the analyzer (`businesscardanalyzer`). It's reused at analyze time. |
| `resource` | The schema as a dict (`json.loads(schema)`) or a `ContentAnalyzer` model. |
| `allow_replace` | Overwrite an existing analyzer with the same ID, which makes reruns idempotent. |
| Returns | An `LROPoller`. `.result()` waits for creation to finish and returns the `ContentAnalyzer`. |

| | |
|---|---|
| **REST equivalent** | `PUT {endpoint}/contentunderstanding/analyzers/{analyzerId}?api-version=2025-11-01` with the JSON body → `201` plus an `Operation-Location` header → `GET` that URL until `"status": "succeeded"` |
| **Agent equivalent** | None at definition time. Analyzers are provisioned resources, like an index, and are usually created by a setup script or CI/CD. |

🎯 *Plan and manage → "Integrate Foundry projects with **CI/CD** pipelines"*. Schema-as-code plus `allow_replace` is deployable infrastructure.

## 4. Analyze: `client.begin_analyze_binary(analyzer_id, binary_input=bytes)` → `poller.result()`

| | |
|---|---|
| **Parameters** | `analyzer_id`; `binary_input` (raw file bytes); optional `content_type` (defaults to `application/octet-stream`), `content_range` (pages or time range), `processing_location` |
| **vs `begin_analyze`** | `begin_analyze(analyzer_id, inputs=[AnalysisInput(url=... or data=...)])` takes URLs or several inputs (used in the vision lab). `analyze_binary` is the simplest option for a single local file. |
| **Result** | `AnalysisResult`. `result.contents[]` each have `.markdown` (OCR and layout as markdown) and `.fields` (dict of field name → typed field with `.value`, plus `.confidence` and spans when `returnDetails` is on). |
| **`json.dump(dict(result), ..., default=str)`** | Saves the raw result for inspection (`results.json`). |
| **REST equivalent** | `POST {endpoint}/contentunderstanding/analyzers/{analyzerId}:analyzeBinary?api-version=2025-11-01` with the image bytes as the body → `202` plus `Operation-Location` → `GET {endpoint}/contentunderstanding/analyzerResults/{id}?api-version=2025-11-01` until `Succeeded` |
| **Agent/MCP equivalent** | An MCP tool such as `read_business_card(image_url)` that wraps this call. The agent can then *act* on the fields, for example by creating a CRM contact through another tool, with `require_approval="always"` on the write. |

🎯 *Information extraction → "Produce **clean, grounded representations** to use with agents and RAG by using Content Understanding"*
🎯 *Build agents → "Integrate agent tools, including APIs, knowledge stores, search, **content understanding**, and custom functions"*
🎯 *Text analysis → "Implement solutions to extract entities… and **structured JSON outputs**…"*

---

## 5. CU analyzer vs LLM-only extraction vs agent tool

| | CU analyzer (this lab) | Multimodal LLM + JSON schema | Agent + CU tool |
|---|---|---|---|
| Grounding | `extract` fields with source spans and confidence | No spans; can hallucinate | Same as CU |
| Consistency | Schema enforced by the service | `text.format={"type":"json_schema",...}` enforces shape only | Same as CU |
| Composition | Single step | Single step | Chains extraction with actions |

🎯 *Plan and manage → "Choose the appropriate Foundry services for generative tasks, grounding… or **multimodal processing**"*

## References

- [Content Understanding REST quickstart](https://learn.microsoft.com/azure/ai-services/content-understanding/quickstart/use-rest-api)
- [Create a custom analyzer](https://learn.microsoft.com/azure/ai-services/content-understanding/tutorial/create-custom-analyzer)
- [AI-103 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)
