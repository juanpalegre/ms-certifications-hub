# Sección 5: Implementar soluciones de extracción de información (10-15%)

---

## 5.1 Extracción de información con IA

### Conceptos clave

- **Pipelines multimodales de extracción**: combinan OCR, análisis de diseño (layout) y extracción de campos para procesar documentos complejos (PDFs escaneados, formularios, contratos).
- **Azure AI Document Intelligence** (antes Form Recognizer): servicio de bajo/medio nivel que usa modelos preentrenados y personalizados para extraer texto, tablas y pares clave-valor de documentos. La salida es JSON estructurado con coordenadas, confianza y jerarquía de elementos.
- **Azure AI Content Understanding** (nuevo en Foundry): servicio de alto nivel que define *analizadores* personalizados basados en LLMs multimodales. La salida puede ser JSON estructurado o markdown limpio, directamente utilizable por agentes RAG.
- **Modelos prebuilt de Document Intelligence**:
  - `prebuilt-invoice`: facturas (proveedor, importe, impuestos, líneas de pedido)
  - `prebuilt-receipt`: tickets de compra
  - `prebuilt-idDocument`: DNI, pasaportes, carnets de conducir
  - `prebuilt-businessCard`: tarjetas de visita
  - `prebuilt-contract`: contratos (partes, cláusulas, fechas)
  - `prebuilt-tax.us.w2`: formularios fiscales (EE.UU.)
  - `prebuilt-layout`: cualquier documento → texto + tablas + key-value pairs
- **Modelo layout**: el más genérico; extrae todo el contenido estructural sin necesidad de entrenamiento.
- **Modelos custom**: se entrena con documentos de dominio propio etiquetando campos. Tipos: *custom template* (formularios de diseño fijo) y *custom neural* (documentos más variados).
- **Content Understanding — analizadores**: se define un esquema de extracción (campos, tipos) y el servicio usa un modelo multimodal para rellenarlos. No requiere ejemplos de entrenamiento.

### Código Python de referencia

```python
# Document Intelligence — modelo prebuilt-invoice
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest
from azure.core.credentials import AzureKeyCredential

di_client = DocumentIntelligenceClient(
    endpoint="https://<resource>.cognitiveservices.azure.com/",
    credential=AzureKeyCredential("<api-key>"),
)

# Analizar factura desde URL
poller = di_client.begin_analyze_document(
    model_id="prebuilt-invoice",
    analyze_request=AnalyzeDocumentRequest(
        url_source="https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/master/curl/form-recognizer/sample-invoice.pdf"
    ),
)
result = poller.result()

for invoice in result.documents:
    fields = invoice.fields
    print("Proveedor:", fields.get("VendorName", {}).get("content"))
    print("Total:", fields.get("InvoiceTotal", {}).get("content"))
    print("Fecha:", fields.get("InvoiceDate", {}).get("content"))
    for item in fields.get("Items", {}).get("value", []):
        desc = item.get("value", {}).get("Description", {}).get("content")
        amount = item.get("value", {}).get("Amount", {}).get("content")
        print(f"  Línea: {desc} — {amount}")
```

```python
# Document Intelligence — modelo layout (cualquier documento)
poller = di_client.begin_analyze_document(
    model_id="prebuilt-layout",
    analyze_request=AnalyzeDocumentRequest(url_source="https://ejemplo.com/contrato.pdf"),
)
result = poller.result()

# Extraer tablas
for i, table in enumerate(result.tables):
    print(f"\nTabla {i+1} ({table.row_count} filas × {table.column_count} columnas):")
    for cell in table.cells:
        print(f"  [{cell.row_index},{cell.column_index}] {cell.content}")

# Extraer todo el texto por página
for page in result.pages:
    print(f"\n--- Página {page.page_number} ---")
    for line in page.lines:
        print(line.content)
```

```python
# Document Intelligence — modelo custom
poller = di_client.begin_analyze_document(
    model_id="mi-modelo-custom-facturas",   # ID del modelo entrenado
    analyze_request=AnalyzeDocumentRequest(url_source="https://ejemplo.com/factura-custom.pdf"),
)
result = poller.result()

for doc in result.documents:
    for field_name, field_value in doc.fields.items():
        print(f"{field_name}: {field_value.content} (confianza: {field_value.confidence:.2f})")
```

```python
# Azure AI Content Understanding — crear analizador y extraer
import requests, json, time

ENDPOINT = "https://<resource>.cognitiveservices.azure.com/contentunderstanding"
API_KEY  = "<api-key>"
HEADERS  = {"Ocp-Apim-Subscription-Key": API_KEY, "Content-Type": "application/json"}
API_VER  = "2024-12-01-preview"

# 1. Definir analizador con esquema de extracción
analyzer_def = {
    "description": "Extrae datos clave de contratos",
    "scenario": "documentAnalysis",
    "fieldSchema": {
        "fields": {
            "Partes": {"type": "array", "items": {"type": "string"}},
            "FechaInicio": {"type": "string", "description": "Fecha de inicio del contrato"},
            "FechaFin":   {"type": "string", "description": "Fecha de vencimiento del contrato"},
            "ValorTotal": {"type": "number", "description": "Valor económico total del contrato"},
        }
    },
}

r = requests.put(
    f"{ENDPOINT}/analyzers/analizador-contratos?api-version={API_VER}",
    headers=HEADERS, data=json.dumps(analyzer_def),
)
print("Analizador creado:", r.status_code)

# 2. Analizar un documento
analyze_req = {"url": "https://ejemplo.com/contrato.pdf"}
r = requests.post(
    f"{ENDPOINT}/analyzers/analizador-contratos:analyze?api-version={API_VER}",
    headers=HEADERS, data=json.dumps(analyze_req),
)
operation_url = r.headers["Operation-Location"]

# 3. Esperar resultado
while True:
    r = requests.get(operation_url, headers=HEADERS)
    status = r.json().get("status")
    if status == "succeeded":
        output = r.json()["result"]
        print(json.dumps(output, ensure_ascii=False, indent=2))
        break
    elif status == "failed":
        print("Error:", r.json())
        break
    time.sleep(2)
```

### Puntos clave para el examen

| Aspecto | Detalle |
|---|---|
| Modelo más genérico de Document Intelligence | `prebuilt-layout` (texto + tablas + key-value de cualquier doc) |
| Modelo para facturas | `prebuilt-invoice` |
| Custom template vs custom neural | template = formulario fijo; neural = documentos variados |
| Salida de Document Intelligence | JSON con `fields`, `confidence`, `boundingRegions` |
| Content Understanding — analizador | definir esquema de campos → extracción LLM-powered |
| Salida de Content Understanding | JSON estructurado o markdown (útil para RAG / agentes) |
| Pipeline multimodal típico | PDF → Document Intelligence (OCR+layout) → JSON/markdown → LLM/agente |

---

## 5.2 Comparativa: Document Intelligence vs Content Understanding

### Tabla de diferencias clave

| Criterio | Azure AI Document Intelligence | Azure AI Content Understanding |
|---|---|---|
| **Nivel de abstracción** | Bajo / medio (template/modelo) | Alto (LLM-powered) |
| **Necesita entrenamiento** | Custom models sí; prebuilt no | No (define esquema, sin etiquetado) |
| **Tipos de entrada** | PDFs, imágenes, formularios | PDFs, imágenes, audio, video |
| **Salida principal** | JSON con campos, bounding boxes, confianza | JSON estructurado o markdown limpio |
| **Ideal para** | Formularios estandarizados, extracción de campos concretos | Documentos complejos/no estructurados, pipelines RAG, agentes |
| **Integración con Foundry** | Como skill de Azure AI Search | Como tool nativa de agentes Foundry |
| **Coste** | Menor (modelo determinista) | Mayor (LLM por cada análisis) |
| **Precisión en formularios fijos** | Muy alta (si el diseño es consistente) | Alta, más flexible pero menos determinista |
| **Soporte multimodal** | Principalmente texto/imagen | Texto, imagen, audio, video |

### Cuándo usar cada uno

```
¿El documento tiene una plantilla fija o es un tipo estandarizado (factura, DNI)?
   → Azure AI Document Intelligence (prebuilt o custom template)

¿El documento es variado, no estructurado, o necesitas razonamiento semántico?
   → Azure AI Content Understanding

¿Necesitas integrar la extracción directamente como tool de un agente Foundry?
   → Azure AI Content Understanding

¿Prioridad es bajo coste y alta precisión en campos concretos de formularios repetitivos?
   → Azure AI Document Intelligence
```

### Pipeline multimodal completo (referencia)

```python
# Patrón: PDF → Document Intelligence → markdown → LLM (RAG grounding)
from openai import AzureOpenAI
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest, ContentFormat
from azure.core.credentials import AzureKeyCredential

di_client = DocumentIntelligenceClient(
    endpoint="https://<di-resource>.cognitiveservices.azure.com/",
    credential=AzureKeyCredential("<di-key>"),
)

oai_client = AzureOpenAI(
    azure_endpoint="https://<oai-resource>.openai.azure.com/",
    api_key="<oai-key>",
    api_version="2024-02-01",
)

# Paso 1: Extraer contenido del PDF como markdown
poller = di_client.begin_analyze_document(
    model_id="prebuilt-layout",
    analyze_request=AnalyzeDocumentRequest(url_source="https://ejemplo.com/informe.pdf"),
    output_content_format=ContentFormat.MARKDOWN,  # salida en markdown
)
result = poller.result()
markdown_content = result.content  # texto estructurado en markdown

# Paso 2: Usar como contexto grounded en el LLM
pregunta = "¿Cuál es el resumen ejecutivo del informe?"
response = oai_client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Responde solo usando la información del documento proporcionado."},
        {"role": "user", "content": f"Documento:\n\n{markdown_content}\n\nPregunta: {pregunta}"},
    ],
)
print(response.choices[0].message.content)
```

### Puntos clave para el examen

| Aspecto | Detalle |
|---|---|
| Diferencia principal | Document Intelligence = template/modelo; Content Understanding = LLM-powered |
| `output_content_format=MARKDOWN` | Document Intelligence puede devolver el contenido como markdown |
| Bounding boxes | Document Intelligence incluye coordenadas de cada campo (útil para UI) |
| Content Understanding sin etiquetado | define esquema de extracción; no necesita datos de entrenamiento |
| Integración con Azure AI Search | Document Intelligence como OCR skill en el indexer pipeline |
| Integración con agentes Foundry | Content Understanding como tool nativa; Document Intelligence via custom skill |
| Confianza de campos | Document Intelligence devuelve `confidence` por campo (0.0 – 1.0) |

---

## Recursos

- [Azure AI Document Intelligence — Documentación](https://learn.microsoft.com/azure/ai-services/document-intelligence/overview)
- [Modelos prebuilt de Document Intelligence](https://learn.microsoft.com/azure/ai-services/document-intelligence/concept-model-overview)
- [Custom models — Document Intelligence](https://learn.microsoft.com/azure/ai-services/document-intelligence/concept-custom)
- [Azure AI Content Understanding](https://learn.microsoft.com/azure/ai-services/content-understanding/overview)
- [Content Understanding — Quickstart con analizadores](https://learn.microsoft.com/azure/ai-services/content-understanding/quickstart)
- [Patrón RAG con Document Intelligence + Azure AI Search](https://learn.microsoft.com/azure/search/cognitive-search-concept-image-scenarios)
