# Sección 4: Implementar soluciones de análisis de texto y voz (10-15%)

---

## 4.1 Análisis de texto con LLMs

### Conceptos clave

- **Extracción de entidades (NER)**: identificar personas, organizaciones, fechas, lugares, etc. Se puede hacer vía **Azure AI Language** (API clásica) o mediante un prompt estructurado con GPT-4o / GPT-4 Turbo en Foundry.
- **Salidas JSON estructuradas**: usando `response_format={"type":"json_object"}` o Structured Outputs en la API de Azure OpenAI, el modelo devuelve JSON válido listo para parsear.
- **Resumen (Summarization)**: Azure AI Language ofrece resumen extractivo (oraciones clave del original) e abstractivo (texto generado). Los LLMs en Foundry permiten resúmenes más personalizados.
- **Detección de sentimiento**: Azure AI Language → `analyze_sentiment` (positivo / negativo / neutro / mixto con puntuaciones). Con LLMs se puede hacer análisis de tono más matizado.
- **Detección de PII**: Azure AI Language → `recognize_pii_entities` detecta y enmascara información personal (correos, teléfonos, DNI, IBAN). También disponible como skill en Azure AI Search.
- **Contenido confidencial / compliance**: los LLMs (via prompt engineering) pueden clasificar documentos por nivel de confidencialidad o extraer cláusulas de contratos de compliance.
- **Traducción**: **Azure AI Translator** soporta +100 idiomas, traducción de documentos y detección automática de idioma. Alternativa: llamar a un LLM con instrucción de traducción (más flexible, mayor coste).
- **Personalización de dominio**: se pueden combinar prompts de sistema con ejemplos (few-shot) para adaptar la extracción o resumen al lenguaje de un dominio concreto (legal, médico, financiero).

### Código Python de referencia

```python
# Azure AI Language — Análisis de sentimiento
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

client_lang = TextAnalyticsClient(
    endpoint="https://<resource>.cognitiveservices.azure.com/",
    credential=AzureKeyCredential("<api-key>"),
)

documents = ["El servicio fue excelente pero la entrega tardó demasiado."]
result = client_lang.analyze_sentiment(documents)
for doc in result:
    print(f"Sentimiento: {doc.sentiment} — scores: {doc.confidence_scores}")
    for sentence in doc.sentences:
        print(f"  Oración: '{sentence.text}' → {sentence.sentiment}")
```

```python
# Azure AI Language — NER y detección de PII
ner_result = client_lang.recognize_entities(documents)
pii_result = client_lang.recognize_pii_entities(documents)

for doc in ner_result:
    for entity in doc.entities:
        print(f"Entidad: {entity.text} | Categoría: {entity.category} | Score: {entity.confidence_score:.2f}")

for doc in pii_result:
    print("Texto redactado:", doc.redacted_text)
```

```python
# Extracción estructurada con GPT-4 Turbo (JSON estructurado)
from openai import AzureOpenAI
import json

client = AzureOpenAI(
    azure_endpoint="https://<resource>.openai.azure.com/",
    api_key="<api-key>",
    api_version="2024-02-01",
)

response = client.chat.completions.create(
    model="gpt-4-turbo",
    response_format={"type": "json_object"},
    messages=[
        {"role": "system", "content": (
            "Extrae del texto: entidades (nombre, tipo), sentimiento general "
            "y resumen en máximo 2 oraciones. Responde solo en JSON válido."
        )},
        {"role": "user", "content": "La empresa Contoso firmó un acuerdo con Microsoft el 5 de enero de 2025."},
    ],
)
data = json.loads(response.choices[0].message.content)
print(json.dumps(data, ensure_ascii=False, indent=2))
```

```python
# Azure AI Translator — traducción de texto
import requests

endpoint  = "https://api.cognitive.microsofttranslator.com/translate"
api_key   = "<translator-api-key>"
headers   = {"Ocp-Apim-Subscription-Key": api_key, "Content-Type": "application/json"}
params    = {"api-version": "3.0", "to": ["en", "fr"]}
body      = [{"text": "Hola, ¿cómo estás?"}]

response  = requests.post(endpoint, headers=headers, params=params, json=body)
for translation in response.json()[0]["translations"]:
    print(f"{translation['to']}: {translation['text']}")
```

### Puntos clave para el examen

| Aspecto | Detalle |
|---|---|
| JSON estructurado en Azure OpenAI | `response_format={"type":"json_object"}` |
| Resumen extractivo vs abstractivo | Language API: extractivo = frases del original; abstractivo = texto nuevo |
| Detección de PII | `recognize_pii_entities` → devuelve `redacted_text` con entidades enmascaradas |
| Azure AI Translator | REST API, detección automática de idioma, 100+ idiomas, traducción de documentos |
| Personalización de dominio | few-shot prompting en el mensaje de sistema |
| Sentimiento a nivel de oración | disponible en `analyze_sentiment` iterando `doc.sentences` |

---

## 4.2 Soluciones de voz (Speech)

### Conceptos clave

- **Speech-to-Text (STT)**: convierte audio en texto. Azure AI Speech ofrece modelos neurales multi-idioma y soporte para **Whisper** (OpenAI) vía Azure. Útil para transcripción de llamadas, comandos de voz en agentes.
- **Text-to-Speech (TTS)**: convierte texto en audio. Azure AI Speech proporciona voces neuronales de alta calidad (Neural TTS). Se puede controlar tono, velocidad y énfasis mediante **SSML**.
- **Voz como modalidad de agente**: integrar STT + LLM + TTS para crear asistentes de voz end-to-end. El agente escucha (STT) → razona (LLM) → responde con voz (TTS).
- **Modelos de voz personalizados** (Custom Speech / Custom Voice): adaptar el reconocimiento a vocabulario específico de dominio o crear una voz sintética propia de la marca.
- **Razonamiento multimodal desde audio**: GPT-4o (modo audio) puede recibir entradas de audio directamente y razonar sobre contenido hablado.
- **Traducción de voz**: Speech Translation API combina STT + traducción + TTS en un único flujo, o se puede orquestar con Azure AI Translator + LLM.

### Código Python de referencia

```python
# Speech-to-Text (reconocimiento desde micrófono)
import azure.cognitiveservices.speech as speechsdk

speech_config = speechsdk.SpeechConfig(
    subscription="<speech-api-key>",
    region="eastus",
)
speech_config.speech_recognition_language = "es-ES"

recognizer = speechsdk.SpeechRecognizer(speech_config=speech_config)
print("Habla ahora...")
result = recognizer.recognize_once_async().get()

if result.reason == speechsdk.ResultReason.RecognizedSpeech:
    print("Transcripción:", result.text)
else:
    print("Error:", result.reason)
```

```python
# Text-to-Speech con SSML (control de entonación)
tts_config = speechsdk.SpeechConfig(
    subscription="<speech-api-key>",
    region="eastus",
)
tts_config.speech_synthesis_voice_name = "es-ES-ElviraNeural"

ssml = """
<speak version='1.0' xml:lang='es-ES'>
  <voice name='es-ES-ElviraNeural'>
    <prosody rate='slow' pitch='+5%'>
      Bienvenido al asistente de Azure AI.
    </prosody>
  </voice>
</speak>
"""

synthesizer = speechsdk.SpeechSynthesizer(speech_config=tts_config)
result = synthesizer.speak_ssml_async(ssml).get()
if result.reason == speechsdk.ResultReason.SynthesizingAudioCompleted:
    print("Audio sintetizado correctamente.")
```

```python
# Traducción de voz (Speech Translation)
translation_config = speechsdk.translation.SpeechTranslationConfig(
    subscription="<speech-api-key>",
    region="eastus",
)
translation_config.speech_recognition_language = "es-ES"
translation_config.add_target_language("en")

translator = speechsdk.translation.TranslationRecognizer(translation_config=translation_config)
result = translator.recognize_once_async().get()

if result.reason == speechsdk.ResultReason.TranslatedSpeech:
    print("Original:", result.text)
    print("Traducción EN:", result.translations["en"])
```

### Puntos clave para el examen

| Aspecto | Detalle |
|---|---|
| STT en Azure | `SpeechRecognizer` + `SpeechConfig` con idioma |
| TTS neural voices | nombres tipo `es-ES-ElviraNeural`; control con SSML |
| Custom Speech | adapta STT a vocabulario de dominio (acrónimos, nombres propios) |
| Custom Voice | crea una voz sintética propia de la marca |
| Speech Translation | reconocimiento + traducción en un único paso |
| GPT-4o audio | acepta entradas de audio directamente para razonamiento multimodal |

---

## 4.3 Pipelines de recuperación y grounding (RAG)

### Conceptos clave

- **RAG (Retrieval-Augmented Generation)**: patrón de tres fases:
  1. **Ingesta**: chunking de documentos → generación de embeddings → indexación en Azure AI Search.
  2. **Recuperación**: embed de la consulta → búsqueda ANN (Approximate Nearest Neighbor) → top-k chunks.
  3. **Generación**: enviar chunks recuperados como contexto al LLM → respuesta grounded.
- **Azure AI Search — indexer pipeline**:
  - **Indexer**: conecta a una fuente de datos (Blob Storage, SQL, CosmosDB) y ejecuta el pipeline.
  - **Skillset**: cadena de skills aplicados sobre el contenido:
    - *Built-in*: OCR, análisis de imagen, reconocimiento de entidades, frases clave, traducción, PII.
    - *Custom skills*: Azure Function o Web API skill para lógica personalizada.
  - **Index**: almacén vectorial + keyword para búsqueda híbrida.
- **Búsqueda vectorial**: los documentos y las queries se representan como vectores (embeddings con `text-embedding-3-large`). La búsqueda ANN encuentra los vectores más similares.
- **Búsqueda híbrida**: combina BM25 (keyword) + vectorial, con **RRF (Reciprocal Rank Fusion)** para unificar rankings. Generalmente supera a cada método por separado.
- **Semantic ranking**: capa adicional de re-ranking semántico usando un modelo de lenguaje pequeño (activar con `queryType=semantic`).
- **Enriquecimiento en ingesta**: los skills añaden campos enriquecidos al índice (texto extraído de imágenes via OCR, entidades, captions de imágenes, etc.).
- **RAG con agentes**: el pipeline de recuperación se puede conectar como *tool* de un agente en Foundry, permitiendo que el agente decida cuándo buscar.

### Código Python de referencia

```python
# Crear embeddings e indexar documentos en Azure AI Search
from openai import AzureOpenAI
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
from azure.core.credentials import AzureKeyCredential

oai_client = AzureOpenAI(
    azure_endpoint="https://<oai-resource>.openai.azure.com/",
    api_key="<oai-key>",
    api_version="2024-02-01",
)

search_client = SearchClient(
    endpoint="https://<search-resource>.search.windows.net",
    index_name="documentos-rag",
    credential=AzureKeyCredential("<search-key>"),
)

# Generar embedding de un chunk
def embed(text: str) -> list[float]:
    return oai_client.embeddings.create(
        model="text-embedding-3-large",
        input=text,
    ).data[0].embedding

# Indexar un documento
doc = {
    "id": "chunk-001",
    "content": "Azure AI Search admite búsqueda vectorial e híbrida.",
    "content_vector": embed("Azure AI Search admite búsqueda vectorial e híbrida."),
    "source": "manual-azure-search.pdf",
}
search_client.upload_documents([doc])
print("Documento indexado.")
```

```python
# Recuperación híbrida (keyword + vectorial) y generación (RAG)
query = "¿Cómo funciona la búsqueda híbrida en Azure AI Search?"
query_vector = embed(query)

vector_query = VectorizedQuery(
    vector=query_vector,
    k_nearest_neighbors=5,
    fields="content_vector",
)

results = search_client.search(
    search_text=query,           # búsqueda keyword (BM25)
    vector_queries=[vector_query],  # búsqueda vectorial
    query_type="semantic",       # re-ranking semántico (opcional)
    semantic_configuration_name="default",
    top=5,
    select=["id", "content", "source"],
)

# Construir contexto para el LLM
context = "\n\n".join(r["content"] for r in results)

response = oai_client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Responde usando solo la información del contexto proporcionado."},
        {"role": "user", "content": f"Contexto:\n{context}\n\nPregunta: {query}"},
    ],
)
print(response.choices[0].message.content)
```

```python
# Skillset con OCR (configuración vía REST — ejemplo simplificado)
import requests, json

SEARCH_ENDPOINT = "https://<resource>.search.windows.net"
API_KEY = "<search-admin-key>"
API_VERSION = "2024-05-01-preview"

skillset = {
    "name": "skillset-ocr",
    "description": "OCR + extracción de entidades",
    "skills": [
        {
            "@odata.type": "#Microsoft.Skills.Vision.OcrSkill",
            "name": "ocr",
            "inputs": [{"name": "image", "source": "/document/normalized_images/*"}],
            "outputs": [{"name": "text", "targetName": "extracted_text"}],
        },
        {
            "@odata.type": "#Microsoft.Skills.Text.EntityRecognitionSkill",
            "name": "ner",
            "inputs": [{"name": "text", "source": "/document/extracted_text"}],
            "outputs": [{"name": "organizations", "targetName": "organizations"}],
        },
    ],
}

url = f"{SEARCH_ENDPOINT}/skillsets/skillset-ocr?api-version={API_VERSION}"
headers = {"Content-Type": "application/json", "api-key": API_KEY}
r = requests.put(url, headers=headers, data=json.dumps(skillset))
print(r.status_code, r.json())
```

### Puntos clave para el examen

| Aspecto | Detalle |
|---|---|
| Fases de RAG | Ingestar (chunk→embed→index) → Recuperar (embed query→ANN) → Generar (LLM+contexto) |
| Modelo de embedding recomendado | `text-embedding-3-large` (Azure OpenAI) |
| Búsqueda híbrida | BM25 + vectorial, fusionados con RRF |
| `queryType=semantic` | activa re-ranking semántico (requiere plan Standard S1+) |
| OCR skill en skillset | `#Microsoft.Skills.Vision.OcrSkill` → extrae texto de imágenes en PDFs |
| Custom skill | Azure Function expuesta como Web API skill |
| RAG como tool de agente | el agente decide cuándo invocar la búsqueda |

---

## Recursos

- [Azure AI Language — Referencia](https://learn.microsoft.com/azure/ai-services/language-service/overview)
- [Azure AI Translator](https://learn.microsoft.com/azure/ai-services/translator/overview)
- [Azure AI Speech — STT y TTS](https://learn.microsoft.com/azure/ai-services/speech-service/overview)
- [Azure AI Search — Búsqueda vectorial](https://learn.microsoft.com/azure/search/vector-search-overview)
- [Azure AI Search — Skillsets](https://learn.microsoft.com/azure/search/cognitive-search-working-with-skillsets)
- [Patrón RAG con Azure AI Search](https://learn.microsoft.com/azure/search/retrieval-augmented-generation-overview)
- [Azure OpenAI — Structured Outputs](https://learn.microsoft.com/azure/ai-services/openai/how-to/structured-outputs)
