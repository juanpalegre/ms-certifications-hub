# AI-103 — Cheat Sheet de repaso rápido

> Resumen ejecutivo de las 5 secciones. Repaso pre-examen.
> Fecha objetivo: ~31 mayo 2026

---

## Sección 1: Planear y administrar Azure AI (25-30%) ⚠️

- **Jerarquía Foundry**: Subscription → **Hub** (infraestructura compartida: Storage, Key Vault, VNet, Log Analytics) → **Project** (modelos, agentes, índices — aislado por proyecto).
- **Selección de modelo**: GPT-4o = razonamiento complejo; **GPT-4o-mini / Phi-4** = bajo coste; **text-embedding-3-large** = RAG de alta precisión; **Whisper** = STT; **DALL-E 3** = generar imágenes; **GPT-4o vision** = entender imágenes.
- **Tipos de deployment**: **Standard** (pay-per-token, puede sufrir throttling) → **Global Standard** (enrutamiento global, mayor disponibilidad) → **Provisioned / PTU** (capacidad reservada, latencia predecible, coste fijo). Open-source (Llama, Mistral) → **Serverless MaaS**.
- **PTU** = Provisioned Throughput Units = sin throttling dinámico + coste fijo. Elegir cuando la latencia predecible sea crítica en producción.
- **Cuotas**: TPM (Tokens/min) y RPM (Requests/min) por deployment. Mitigar con retry + exponential backoff ante error **429**.
- **Seguridad**: usar **Managed Identity** + `DefaultAzureCredential` (sin API keys en código). **Key Vault** para secretos. **Private Endpoints** = tráfico sin salir a internet.
- **RBAC clave**: `Cognitive Services User` (solo inferencia) | `AI Inference Deployment Operator` (gestionar deployments) | `AI Developer` (todo en dev/staging) | `Search Index Data Contributor` (escribir en índices).
- **Monitoreo**: Azure Monitor Metrics (`TokensConsumed`) → Application Insights (`DurationMs`) → Log Analytics con **KQL** (detectar errores 429, drift). Alertas cuando scores de evaluación bajen de umbrales.
- **CI/CD**: commit → lint + unit tests → **evaluaciones AI automáticas** (gate de calidad) → staging → aprobación humana → producción.
- **IA Responsable — capas**: Filtros de contenido (inputs y outputs) → **Prompt Shield** (inyección directa e indirecta) → **Groundedness Detection** (alucinaciones) → Evaluaciones periódicas → Auditoría con `thread_id` + `run_id`.

---

## Sección 2: IA Generativa y Agentes (30-35%) ⚠️⚠️

- **RAG — 3 fases**: (1) **Ingesta**: chunk → embed (`text-embedding-3-large`) → indexar en Azure AI Search. (2) **Recuperación**: embed query → ANN → top-k chunks. (3) **Generación**: chunks como contexto → LLM → respuesta grounded.
- **Búsqueda híbrida** = BM25 (keyword) + vectorial, fusionados con **RRF** (Reciprocal Rank Fusion). Es el patrón RAG recomendado. `queryType=semantic` activa re-ranking semántico (requiere plan S1+).
- **`temperature`**: `0` = respuestas determinísticas/factuales (RAG); `0.7–1.0` = creatividad/variedad.
- **Agente = Percibir → Razonar (LLM) → Actuar (tool call) → Observar resultado → ciclo**. Componentes: Model, Instructions (system prompt), Tools, **Thread** (memoria/historial), Run, Messages.
- **Herramientas del agente**: `CodeInterpreterTool` (sandbox Python) | `FileSearchTool` (RAG lite sobre archivos subidos) | `BingGroundingTool` (búsqueda web) | `FunctionTool` (Function Calling personalizado) | `AzureAISearchTool` (RAG sobre índice propio).
- **`AIProjectClient.from_connection_string()`** es la forma oficial de conectar apps a Foundry. Usa `DefaultAzureCredential`.
- **Multiagente**: patrón **Orquestador → Agentes especializados** via Function Calling. También con **Semantic Kernel**: `ChatCompletionAgent` + `AgentGroupChat` + `TerminationStrategy`.
- **Human-in-the-Loop (HITL)**: cuando `run.status == "requires_action"` el agente pausa y espera `submit_tool_outputs_to_run()` con aprobación humana.
- **Prompt engineering clave**: system prompt claro con rol + restricciones; **few-shot** (ejemplos en el prompt); **Chain-of-Thought** (pedir razonamiento paso a paso); `response_format={"type":"json_object"}` para salidas estructuradas.
- **Evaluación de calidad**: usar **LLM-as-judge** (modelo GPT-4o actúa como juez). SDK: `evaluate(data=..., evaluators={...})`. Dataset en JSONL con campos `query`, `context`, `response`, `ground_truth`.
- **Prompt Flow**: orquestación visual de cadenas LLM (DAG). Tipos de flujo: Standard, Chat, Evaluation. Integrable en CI/CD.
- **Fine-tuning**: cuando el prompting no es suficiente para adaptar el modelo al dominio. Más caro pero más preciso que few-shot para vocabulario muy especializado.

---

## Sección 3: Computer Vision (10-15%)

- **Generación de imágenes**: **DALL-E 3** (texto → imagen, parámetros: `size`, `quality="hd"`, `style="vivid|natural"`, `n`). **gpt-image-1** = generación y **inpainting** (requiere imagen original + máscara PNG con zonas transparentes).
- **`response_format="b64_json"`** devuelve imagen como base64 (sin URL expirable); `"url"` = enlace temporal.
- **GPT-4o vision (multimodal)**: imagen vía URL pública o `data:image/jpeg;base64,<datos>`. Parámetro `detail`: `"low"` (rápido, menos tokens) vs `"high"` (más preciso). Usos: VQA, captioning, alt text (WCAG), OCR visual.
- **Azure AI Video Indexer**: extrae automáticamente transcripciones, caras, objetos, escenas, emociones, marcas y palabras clave de vídeos. Output: índice consultable via REST API.
- **Azure AI Content Understanding**: define **analizadores** con esquema de campos → extracción LLM-powered de imágenes/docs. Salida JSON o markdown. Útil para pipelines RAG y agentes.
- **Moderación de imágenes**: `ContentSafetyClient.analyze_image()`. Severidad **0–6**; umbral de bloqueo típico ≥ 4. Categorías: Hate, Violence, Sexual, Self-harm.
- **Prompt injection visual**: texto oculto en imagen puede manipular al modelo. Mitigación: extraer texto con GPT-4o → analizar con Content Safety antes de procesar.

---

## Sección 4: Análisis de texto + Speech + RAG (10-15%)

- **Azure AI Language** (`TextAnalyticsClient`): `analyze_sentiment` (positivo/negativo/neutro/mixto, nivel de oración) | `recognize_entities` (NER: personas, orgs, fechas) | `recognize_pii_entities` → devuelve `redacted_text` con entidades enmascaradas.
- **Resumen**: extractivo (oraciones del original) vs **abstractivo** (texto generado). Language API ofrece ambos; LLMs son más flexibles.
- **Structured Outputs** (Azure OpenAI): `response_format={"type":"json_object"}` garantiza JSON válido. Útil para extracción y clasificación.
- **Azure AI Translator**: REST API, 100+ idiomas, detección automática de idioma, traducción de documentos completos. Header `Ocp-Apim-Subscription-Key`.
- **Speech-to-Text**: `SpeechRecognizer` + `SpeechConfig(subscription, region, language)`. **Custom Speech** = adapta STT a vocabulario de dominio (acrónimos, nombres propios).
- **Text-to-Speech**: voces neuronales (`es-ES-ElviraNeural`). Control de tono/velocidad/énfasis con **SSML**. **Custom Voice** = voz sintética propia de la marca.
- **RAG en AI Search — Skillset pipeline**: `Indexer` (conecta fuente: Blob/SQL/CosmosDB) → `Skillset` (OCR `#Microsoft.Skills.Vision.OcrSkill`, NER, traducción, PII, custom skill via Azure Function) → `Index` (vectorial + keyword).

---

## Sección 5: Extracción de información (10-15%)

- **Azure AI Document Intelligence** (`DocumentIntelligenceClient`): `begin_analyze_document(model_id, analyze_request)`. Modelos prebuilt: `prebuilt-layout` (más genérico: texto + tablas + key-value), `prebuilt-invoice`, `prebuilt-receipt`, `prebuilt-idDocument`, `prebuilt-contract`.
- **Custom models**: `custom template` = formularios de diseño fijo (alta precisión) | `custom neural` = documentos variados (más flexible). Requieren etiquetado de datos.
- **Salida de Document Intelligence**: JSON con `fields` (content + confidence + boundingRegions). `output_content_format=MARKDOWN` para salida en markdown (ideal para RAG grounding).
- **Azure AI Content Understanding**: define analizador con `fieldSchema` → extracción LLM-powered sin etiquetado. Salida JSON o markdown. Soporta texto, imagen, audio, video.
- **Cuándo elegir cada uno**: doc estandarizado/formulario fijo → **Document Intelligence** (más barato, determinista). Doc variado/no estructurado o integración como tool de agente → **Content Understanding**.
- **Pipeline típico**: PDF → Document Intelligence (OCR + layout → markdown) → LLM/agente (grounding).

---

## SDKs y clases clave (Python)

| SDK / Paquete | Clase / Función principal | Uso |
|---|---|---|
| `azure-ai-projects` | `AIProjectClient.from_connection_string()` | Conectar a Foundry project (agentes, índices, evaluaciones) |
| `azure-ai-inference` | `ChatCompletionsClient` | Llamar a modelos serverless (Phi, Llama, Mistral) |
| `openai` (azure) | `AzureOpenAI` | Llamar a GPT-4o, embeddings, DALL-E, Whisper |
| `azure-search-documents` | `SearchClient`, `SearchIndexClient` | CRUD de documentos e índices en Azure AI Search |
| `azure-ai-textanalytics` | `TextAnalyticsClient` | NER, sentimiento, PII, resumen |
| `azure-cognitiveservices-speech` | `SpeechRecognizer`, `SpeechSynthesizer` | STT y TTS con Azure Speech |
| `azure-ai-documentintelligence` | `DocumentIntelligenceClient` | Extracción de formularios y documentos |
| `azure-ai-contentsafety` | `ContentSafetyClient` | Moderación de texto e imagen |
| `azure-ai-evaluation` | `evaluate()`, `GroundednessEvaluator`, etc. | Evaluación de calidad de respuestas LLM |
| `semantic-kernel` | `Kernel`, `ChatCompletionAgent`, `AgentGroupChat` | Orquestación multiagente con SK |
| `azure-identity` | `DefaultAzureCredential` | Autenticación sin API keys (Managed Identity / CLI) |

---

## Servicios y cuándo usarlos

| Necesito… | Servicio |
|---|---|
| Chat / generación de texto con LLM | Azure OpenAI (GPT-4o) vía Foundry |
| Bajo coste, tasks simples | GPT-4o-mini o Phi-4 (SLM) |
| Edge / sin internet | Phi-3-mini (on-device) |
| Búsqueda semántica en documentos propios | Azure AI Search (vectorial + híbrida) |
| Agente con tools y memoria | Azure AI Agent Service |
| Orquestación compleja multiagente | Semantic Kernel o Prompt Flow |
| Generar imágenes | DALL-E 3 / gpt-image-1 |
| Analizar imágenes / VQA | GPT-4o vision |
| Transcripción de audio | Azure Speech (STT) / Whisper |
| Texto a voz | Azure Speech (TTS Neural) |
| NER, sentimiento, PII, resumen | Azure AI Language |
| Traducción | Azure AI Translator |
| Extraer campos de facturas/formularios | Azure AI Document Intelligence |
| Extracción LLM de docs complejos | Azure AI Content Understanding |
| Moderación de contenido | Azure AI Content Safety |
| Evaluar calidad de respuestas | Azure AI Evaluation SDK |
| Indexación pipeline con OCR/NER | Azure AI Search Skillset (indexer) |

---

## Métricas de evaluación

| Métrica | Qué mide | Rango | Cuándo es crítica |
|---|---|---|---|
| **Groundedness** | Respuesta fundamentada en el contexto RAG (≠ alucinación) | 1–5 | Siempre en RAG |
| **Relevance** | La respuesta responde la pregunta del usuario | 1–5 | Chatbots, QA |
| **Coherence** | Consistencia lógica y buena estructura | 1–5 | Respuestas largas |
| **Fluency** | Calidad gramatical y lingüística natural | 1–5 | Comunicación al cliente |
| **F1 Score** | Precisión + recall para QA extractiva | 0–1 | Extracción de campos |
| **Similarity** | Similitud semántica con ground truth de referencia | 0–1 | Fine-tuning, benchmarks |
| **Hallucination rate** | % de fabricaciones detectadas | 0–1 | IA crítica / compliance |
| **Safety** | Contenido dañino (Hate, Violence, Sexual, Self-harm) | Pass/Fail | Producción pública |

> **LLM-as-judge**: GPT-4o actúa como modelo juez para puntuar coherencia, relevancia y groundedness automáticamente.

---

## Responsible AI checklist

- ☑ **Filtros de contenido** configurados (Hate, Violence, Sexual, Self-harm) — aplican a **inputs Y outputs** — umbral recomendado: Medium.
- ☑ **Prompt Shield** habilitado — protege contra inyección **directa** (usuario malicioso) e **indirecta** (documentos recuperados en RAG con instrucciones ocultas).
- ☑ **Groundedness Detection** activo — detecta afirmaciones no respaldadas por el contexto → herramienta principal anti-alucinación.
- ☑ **Managed Identity** + RBAC con mínimo privilegio — sin API keys en código; Key Vault para secretos; Private Endpoints en producción.
- ☑ **Evaluaciones periódicas** con golden dataset — detectar drift de calidad; alertas en Azure Monitor cuando scores bajen del umbral.
- ☑ **Auditoría y trazabilidad** — logging de inputs/outputs en Log Analytics (`thread_id` + `run_id`); considerar anonimización de PII antes de almacenar logs.
- ☑ **Instrucciones de sistema explícitas** en agentes — definir rol, restricciones y herramientas mínimas necesarias; limitar `max_iterations`.
- ☑ **Human-in-the-Loop (HITL)** para acciones de alto riesgo — el agente pausa en `requires_action` hasta aprobación humana.

---

## Acrónimos y términos clave

| Término | Definición |
|---|---|
| **RAG** | Retrieval-Augmented Generation — patrón que enriquece el prompt con documentos recuperados |
| **PTU** | Provisioned Throughput Units — capacidad reservada con coste fijo y latencia predecible |
| **TPM / RPM** | Tokens Per Minute / Requests Per Minute — límites de velocidad por deployment |
| **SLM** | Small Language Model (ej: Phi-4, Phi-3-mini) — bajo coste, edge, tasks simples |
| **LLM** | Large Language Model (ej: GPT-4o) — razonamiento complejo, contexto largo |
| **ANN** | Approximate Nearest Neighbor — búsqueda vectorial rápida por similitud |
| **RRF** | Reciprocal Rank Fusion — algoritmo para combinar rankings en búsqueda híbrida |
| **BM25** | Best Match 25 — algoritmo de búsqueda keyword clásico (léxico, frecuencia de términos) |
| **HNSW** | Hierarchical Navigable Small World — algoritmo de índice vectorial en Azure AI Search |
| **Grounding** | Fundamentar la respuesta del LLM en documentos/contexto real (≠ alucinación) |
| **Hallucination** | El modelo genera información falsa o no presente en el contexto |
| **Groundedness** | Métrica que mide si la respuesta refleja las fuentes provistas (1–5) |
| **Thread** | Historial de conversación de un agente — persiste la memoria de corto plazo |
| **Run** | Ejecución de un turno del agente sobre un Thread |
| **HITL** | Human-in-the-Loop — aprobación humana antes de que el agente ejecute acciones críticas |
| **Inpainting** | Modificación selectiva de regiones de una imagen usando una máscara PNG |
| **VQA** | Visual Question Answering — responder preguntas basándose en el contenido de una imagen |
| **SSML** | Speech Synthesis Markup Language — controla tono, velocidad y énfasis en TTS |
| **STT / TTS** | Speech-to-Text / Text-to-Speech — transcripción de voz y síntesis de voz |
| **NER** | Named Entity Recognition — identificar entidades (personas, orgs, fechas, lugares) en texto |
| **PII** | Personally Identifiable Information — datos personales (email, teléfono, DNI, IBAN) |
| **Skillset** | Cadena de skills (OCR, NER, traducción…) aplicados en un indexer pipeline de AI Search |
| **MaaS** | Model as a Service — modelos serverless sin gestionar infraestructura (pay-per-token) |
| **WCAG** | Web Content Accessibility Guidelines — estándar de accesibilidad web (alt text en imágenes) |
| **C2PA** | Content Credentials / provenance watermarking — marca de agua en imágenes generadas por IA |
