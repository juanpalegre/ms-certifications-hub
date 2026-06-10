# Cómo aplica AI-103 en el bot del club deportivo

> El ángulo del ingeniero que implementa el núcleo de IA.
> Escrito en español coloquial, como si lo estuviéramos hablando en una reunión de trabajo.

---

## Las tres perspectivas del mismo proyecto

El bot de WhatsApp para el club deportivo es un solo proyecto, pero cada certificación lo mira desde un lugar distinto:

| Perspectiva | Certificación | "¿Qué soy yo en este proyecto?" |
|---|---|---|
| El que diseña todo | AB-100 | "Yo defino la arquitectura, elijo Foundry, diseño la topología de agentes, gobierno los costos" |
| El que automatiza todo | GH-600 | "Yo configuro el CI/CD, los MCP servers, los Copilot coding agents, las GitHub Actions" |
| **El que construye el cerebro** | **AI-103** | **"Yo implemento el RAG, los agentes en Python, los filtros de seguridad, la evaluación"** |

Los tres trabajan sobre el mismo repositorio, el mismo proyecto en Foundry, el mismo sistema en producción. Lo que cambia es el área de responsabilidad.

---

## El proyecto en 30 segundos

Un socio del club le manda un mensaje por WhatsApp: *"¿Tienen canchas de pádel libres el sábado a las 18?"*

Por debajo pasan estas cosas (foco en lo que hace el AI-103 engineer):

1. El mensaje llega al webhook Node.js → se pasa al servicio Python de IA
2. El servicio Python corre **Content Safety** sobre el mensaje (filtro de entrada)
3. El mensaje se **embebe** con `text-embedding-3-small` y se hace una búsqueda vectorial en **Azure AI Search** (knowledge base del club)
4. Los chunks relevantes (horarios, precios, reglas) se inyectan como contexto en el prompt
5. **GPT-4o vía Azure AI Agent Service** genera la respuesta, potencialmente llamando tools como `checkCourtAvailability`
6. La respuesta pasa por **Content Safety** (filtro de salida)
7. Se guarda un resumen de la conversación en Redis para sesiones largas
8. Se devuelve la respuesta al usuario

El AI-103 engineer construyó cada uno de esos pasos en Python. Eso es lo que este documento mapea al examen.

---

## Sección 1: Planear y administrar Azure AI (25–30%)

### Lo que evalúa el examen

Esta sección no es solo teoría — evalúa decisiones de arquitectura técnica que toma el desarrollador:

- ¿Qué modelo elegís y por qué?
- ¿Cómo configurás Foundry correctamente (hub, proyecto, conexiones, deployments)?
- ¿Cómo manejás costos y cuotas sin quedarte sin tokens en producción?
- ¿Cómo aplicás IA responsable: managed identity, Content Safety, groundedness?

### Cómo aplica en el bot

---

#### Elegir los servicios adecuados

El AB-100 definió la arquitectura general. El AI-103 engineer toma esa decisión y la **implementa y valida técnicamente**. Estas son las elecciones concretas del bot:

| Necesidad | Servicio elegido | Por qué |
|---|---|---|
| Conversación principal | GPT-4o | Razonamiento complejo, function calling, multimodal |
| Consultas simples (FAQ rápido) | GPT-4o-mini | Mucho más barato para respuestas directas |
| Búsqueda semántica en knowledge base | text-embedding-3-small | Buen balance precisión/costo para textos cortos de club |
| Notas de voz de socios | Whisper (Azure Speech) | Los socios mandan audios por WhatsApp |
| Análisis de fotos de canchas | GPT-4o vision | Disputas de reserva: "¿la cancha estaba ocupada?" |
| Generación de imágenes | ❌ DALL-E 3 — NO usado | El bot no necesita generar imágenes |
| Orquestación de agentes | Azure AI Agent Service | Alternativa: LangChain propio, pero Agent Service da trazabilidad, persistencia y tools integradas |
| Índice de conocimiento | Azure AI Search | Mejor que Qdrant o Chroma para producción en Azure — SLA, seguridad, integración nativa |

**¿Por qué Azure AI Agent Service en vez de orquestación custom?**

Podrías armar tu propio loop de agente con LangChain. Pero Agent Service da gratis: persistencia de threads, ejecución de tools, trazabilidad de cada llamada a función, y se integra directamente con Foundry. Para un examen, la respuesta siempre es "prefiere los servicios administrados de Azure".

---

#### Configurar Foundry

El AI-103 engineer es quien conecta la app Python al proyecto de Foundry. El AB-100 creó el hub y el proyecto; el AI-103 los usa desde código:

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

# Conectar al proyecto de Foundry usando Managed Identity
# En Container Apps: la identidad administrada se configura sin secretos
client = AIProjectClient.from_connection_string(
    conn_str=os.environ["AIPROJECT_CONNECTION_STRING"],
    credential=DefaultAzureCredential()
)

# Obtener el cliente de Azure OpenAI desde el proyecto
# (no hace falta manejar el endpoint/key por separado)
openai_client = client.inference.get_azure_openai_client(api_version="2024-12-01-preview")
```

La connection string tiene el formato: `<region>.api.azureml.ms;<subscription_id>;<resource_group>;<project_name>`. Viene de la variable de entorno — nunca hardcodeada.

**Deployments que configura el AI-103 engineer en Foundry:**

| Nombre del deployment | Modelo base | Para qué |
|---|---|---|
| `gpt-4o-club` | gpt-4o (2024-11-20) | Conversación principal + vision |
| `gpt-4o-mini-club` | gpt-4o-mini | FAQ rápido, clasificación de intención |
| `text-embedding-3-small` | text-embedding-3-small | Embeddings para RAG |

---

#### Administrar costos y cuotas

GPT-4o es caro. En un club con 500 socios activos, si cada uno manda 10 mensajes al día, estás hablando de miles de tokens diarios. La estrategia de costos es parte del trabajo del AI-103 engineer:

```python
def route_to_model(message: str, context: dict) -> str:
    """
    Decide qué modelo usar según la complejidad de la consulta.
    Simple → GPT-4o-mini (barato). Complejo → GPT-4o (potente).
    """
    simple_intents = ["horarios", "precio", "contacto", "ubicacion"]
    
    # Clasificación rápida con GPT-4o-mini
    intent = classify_intent_cheap(message)  # usa GPT-4o-mini
    
    if intent in simple_intents:
        return "gpt-4o-mini-club"
    else:
        return "gpt-4o-club"
```

**Cuotas y límites que el examen pregunta:**

- Los deployments estándar en Azure OpenAI tienen límites de **Tokens Per Minute (TPM)** y **Requests Per Minute (RPM)**.
- Un deployment estándar de GPT-4o suele dar 100K TPM. Para un club con tráfico pico (viernes a la tarde reservando canchas del fin de semana), puede no alcanzar.
- Solución: **Provisioned Throughput Units (PTU)** para producción con volumen predecible. El examen pregunta la diferencia entre Standard (pago por uso, sujeto a cuota compartida) y Provisioned (capacidad dedicada, costo fijo).

---

#### Seguridad y IA responsable

Esta parte el examen la toma muy en serio. Tres cosas que implementa el AI-103 engineer:

**1. Managed Identity — nunca API keys en el código:**

```python
# Bien: DefaultAzureCredential usa la Managed Identity del Container App
credential = DefaultAzureCredential()

# Mal: nunca esto en código que va al repo
# openai.api_key = "sk-abcd1234..."  ← NO
```

**2. Content Safety — filtrar mensajes de socios:**

```python
from azure.ai.contentsafety import ContentSafetyClient
from azure.ai.contentsafety.models import AnalyzeTextOptions

safety_client = ContentSafetyClient(
    endpoint=os.environ["CONTENT_SAFETY_ENDPOINT"],
    credential=DefaultAzureCredential()
)

def is_safe(text: str) -> bool:
    response = safety_client.analyze_text(
        AnalyzeTextOptions(text=text, categories=["Hate", "Violence", "SelfHarm", "Sexual"])
    )
    # Si cualquier categoría supera severity 2, bloqueamos
    return all(item.severity < 2 for item in response.categories_analysis)
```

**3. Groundedness filter — detectar alucinaciones:**

El bot no puede inventar precios o reglas del club. El AI-103 engineer configura un filtro de groundedness en Azure AI Evaluation que verifica que las respuestas del bot estén ancladas en el knowledge base, no generadas de la nada.

```python
from azure.ai.evaluation import GroundednessEvaluator

evaluator = GroundednessEvaluator(model_config={...})

score = evaluator(
    response="La clase de pádel cuesta $500 por mes",
    context="Según el reglamento del club, las clases de pádel tienen un costo de $500 mensuales..."
)
# score["groundedness"] entre 1 y 5. Si < 3, algo está mal.
```

---

## Sección 2: IA generativa y agentes (30–35%) ⚠️⚠️

### Lo que evalúa el examen

La sección más pesada. Evalúa:

- Diseñar e implementar RAG completo (desde documentos hasta respuesta)
- Crear y usar agentes con Azure AI Agent Service (function calling, tools, threads)
- Evaluar la calidad de las respuestas (groundedness, relevance, coherence)
- Optimizar prompts y configurar parámetros de inferencia

### Cómo aplica en el bot

---

#### RAG para el conocimiento del club

El club tiene toda su información en PDFs: tarifas, reglamentos, calendarios de torneos, horarios de clases. El AI-103 engineer construye el pipeline que convierte esos PDFs en respuestas del bot.

**El pipeline completo:**

```
PDF (reglamento.pdf)
    ↓ Document Intelligence (Layout model)
Texto estructurado por páginas/secciones
    ↓ Chunking (500 tokens, 50 de overlap)
Chunks de texto
    ↓ text-embedding-3-small
Vectores (1536 dimensiones)
    ↓ Azure AI Search
Índice vectorial + híbrido (keyword + semantic)
    ↓ Query time: embed pregunta → búsqueda → top-3 chunks → contexto
GPT-4o con contexto grounded
```

**Código de búsqueda vectorial en runtime:**

```python
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery

search_client = SearchClient(
    endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
    index_name="club-knowledge-base",
    credential=DefaultAzureCredential()
)

def retrieve_context(user_question: str, top_k: int = 3) -> str:
    # Embeber la pregunta del usuario
    embedding_response = openai_client.embeddings.create(
        input=user_question,
        model="text-embedding-3-small"
    )
    query_vector = embedding_response.data[0].embedding

    # Búsqueda híbrida: vectorial + keyword
    results = search_client.search(
        search_text=user_question,  # keyword search
        vector_queries=[
            VectorizedQuery(
                vector=query_vector,
                k_nearest_neighbors=top_k,
                fields="content_vector"
            )
        ],
        select=["content", "source", "section"],
        top=top_k
    )

    # Armar el contexto para el prompt
    context_parts = []
    for r in results:
        context_parts.append(f"[{r['source']} - {r['section']}]\n{r['content']}")
    
    return "\n\n---\n\n".join(context_parts)
```

**Prompt con RAG:**

```python
def build_augmented_prompt(user_message: str, context: str) -> list[dict]:
    return [
        {
            "role": "system",
            "content": f"""Sos el asistente del Club Deportivo San Martín.
Respondés únicamente basándote en la información del club que se te provee.
Si la información no está en el contexto, decís que no tenés esa información y sugerís contactar a la administración.

INFORMACIÓN DEL CLUB:
{context}
"""
        },
        {"role": "user", "content": user_message}
    ]
```

El examen va a preguntar sobre: **chunking strategy** (tamaño, overlap), **hybrid search vs pure vector**, **semantic ranker** de Azure AI Search, y cómo se construye el prompt con el contexto.

---

#### Azure AI Agent Service para los agentes del club

El bot no solo busca información — también *hace cosas*: verifica disponibilidad de canchas, crea reservas, cancela turnos. Para eso usa agentes con function calling.

```python
from azure.ai.projects.models import FunctionTool, ToolSet

# Definir las herramientas que puede usar el agente
def check_court_availability(court_id: str, date: str, time: str) -> str:
    """Consulta si una cancha está disponible en fecha y hora dadas."""
    # Llama a la API de reservas del club
    response = requests.get(
        f"{CLUB_API_URL}/courts/{court_id}/availability",
        params={"date": date, "time": time}
    )
    data = response.json()
    if data["available"]:
        return f"La cancha {court_id} está disponible el {date} a las {time}."
    else:
        return f"La cancha {court_id} ya está reservada. Próximos turnos libres: {data['next_available']}"

def create_reservation(court_id: str, member_id: str, date: str, time: str) -> str:
    """Crea una reserva para un socio."""
    response = requests.post(
        f"{CLUB_API_URL}/reservations",
        json={"court_id": court_id, "member_id": member_id, "date": date, "time": time}
    )
    reservation = response.json()
    return f"Reserva confirmada. Código: {reservation['confirmation_code']}"

# Crear el agente en Foundry
tools = FunctionTool(functions={check_court_availability, create_reservation})
toolset = ToolSet()
toolset.add(tools)

agent = client.agents.create_agent(
    model="gpt-4o-club",
    name="club-booking-agent",
    instructions="""Sos el asistente de reservas del Club San Martín.
Ayudás a los socios a consultar disponibilidad y hacer reservas de canchas.
Siempre confirmás los detalles antes de crear una reserva.""",
    toolset=toolset
)
```

**El flujo de una conversación:**

```python
# Crear un thread por conversación (uno por sesión de WhatsApp)
thread = client.agents.create_thread()

# Agregar el mensaje del usuario
client.agents.create_message(
    thread_id=thread.id,
    role="user",
    content="Quiero reservar una cancha de pádel para el sábado a las 18"
)

# Ejecutar el agente
run = client.agents.create_and_process_run(
    thread_id=thread.id,
    agent_id=agent.id
)

# El agente decide llamar check_court_availability → ejecuta → obtiene resultado → responde
messages = client.agents.list_messages(thread_id=thread.id)
response = messages.get_last_text_message_by_role("assistant")
print(response.text.value)
```

**Cómo funciona el function calling internamente:**

1. GPT-4o recibe el mensaje del usuario y decide que necesita llamar `check_court_availability`
2. Agent Service ejecuta la función Python localmente
3. El resultado vuelve al modelo como contexto adicional
4. GPT-4o genera la respuesta final para el usuario

El examen pregunta sobre este ciclo, sobre cómo se define el schema de las tools (JSON Schema), y cuándo usar agentes vs llamadas directas al modelo.

---

#### Evaluar el bot

En producción no alcanza con "funciona bien en mis pruebas". El AI-103 engineer configura evaluación sistemática:

```python
from azure.ai.evaluation import (
    GroundednessEvaluator,
    RelevanceEvaluator,
    CoherenceEvaluator,
    evaluate
)

# Dataset de prueba: preguntas con respuestas esperadas y contexto
test_data = [
    {
        "query": "¿Cuánto cuesta la clase de pádel?",
        "response": "La clase de pádel cuesta $500 por mes",
        "context": "Las clases de pádel tienen un costo de $500 mensuales...",
        "ground_truth": "$500 por mes"
    },
    # ... más casos
]

# Correr evaluación
results = evaluate(
    data=test_data,
    evaluators={
        "groundedness": GroundednessEvaluator(model_config=model_config),
        "relevance": RelevanceEvaluator(model_config=model_config),
        "coherence": CoherenceEvaluator(model_config=model_config),
    }
)

print(results["metrics"])
# {"groundedness.mean": 4.2, "relevance.mean": 4.5, "coherence.mean": 4.8}
```

**En el pipeline de CI/CD (GH-600 lo orquesta, AI-103 lo define):**

```yaml
# En el GitHub Actions workflow:
- name: Evaluate AI quality
  run: python scripts/run_evaluation.py
  env:
    QUALITY_THRESHOLD_GROUNDEDNESS: "3.5"
    # Si el promedio de groundedness baja de 3.5, el deployment falla
```

---

#### Optimizar prompts y parámetros

```python
# Para consultas factuales (precios, horarios): temperatura baja
factual_response = openai_client.chat.completions.create(
    model="gpt-4o-club",
    messages=build_augmented_prompt(user_message, context),
    temperature=0.3,   # respuestas consistentes y precisas
    max_tokens=300     # respuestas cortas para WhatsApp
)

# Para conversación libre: temperatura más alta
conversational_response = openai_client.chat.completions.create(
    model="gpt-4o-club",
    messages=messages,
    temperature=0.7,   # más variedad en el lenguaje
    max_tokens=500
)
```

**Chain-of-thought para el flujo de reserva:**

El proceso de reserva tiene varios pasos: verificar disponibilidad → confirmar con el usuario → crear la reserva → enviar confirmación. El system prompt guía este flujo:

```python
BOOKING_SYSTEM_PROMPT = """
Cuando un socio quiere hacer una reserva, seguís estos pasos:
1. Primero verificás qué cancha y horario quiere (si no lo dijo, preguntás)
2. Consultás disponibilidad con check_court_availability
3. Le informás el resultado y pedís confirmación explícita
4. Solo creás la reserva si el socio confirma con "sí" o "confirmar"
5. Enviás el código de confirmación

Nunca creás una reserva sin confirmación explícita del socio.
"""
```

---

## Sección 3: Computer Vision (10–15%)

### Lo que evalúa el examen

Análisis de imágenes, descripción de contenido visual, detección de objetos, OCR visual, y uso de modelos multimodales. También filtros de contenido sobre imágenes.

### Cómo aplica en el bot

Esta sección es la de menor prioridad para el bot — seamos honestos. Pero tiene aplicaciones reales:

---

#### Fotos de canchas para disputas de reserva

Un socio dice "la cancha estaba ocupada cuando llegué pero yo tenía la reserva". Puede mandar una foto desde WhatsApp. GPT-4o vision la analiza:

```python
import base64

def analyze_court_photo(image_bytes: bytes, context: str) -> str:
    """Analiza una foto de cancha enviada por WhatsApp."""
    image_b64 = base64.b64encode(image_bytes).decode("utf-8")
    
    response = openai_client.chat.completions.create(
        model="gpt-4o-club",  # GPT-4o tiene capacidad vision nativa
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": f"Esta foto fue tomada el {context}. ¿La cancha está ocupada con personas jugando? Describí lo que ves."
                    },
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:image/jpeg;base64,{image_b64}",
                            "detail": "low"  # "low" es más barato; "high" para más detalle
                        }
                    }
                ]
            }
        ]
    )
    return response.choices[0].message.content
```

#### Content Safety sobre imágenes

Los socios pueden mandar fotos inapropiadas por WhatsApp. Antes de procesar cualquier imagen:

```python
from azure.ai.contentsafety.models import AnalyzeImageOptions, ImageData

def is_image_safe(image_bytes: bytes) -> bool:
    response = safety_client.analyze_image(
        AnalyzeImageOptions(
            image=ImageData(content=image_bytes)
        )
    )
    return all(item.severity < 2 for item in response.categories_analysis)
```

#### Nota sobre prioridad

Para el examen, esta sección vale 10–15%. Vale la pena conocer los conceptos (Azure AI Vision, Image Analysis 4.0, Florence model, GPT-4o vision), pero no es donde vas a ganar o perder el examen. Foco en Secciones 1 y 2.

---

## Sección 4: Análisis de texto y Speech (10–15%)

### Lo que evalúa el examen

Azure AI Language (NER, PII, sentiment, summarization, clasificación) y Azure AI Speech (STT, TTS, transcripción). También cómo integrar estas capacidades en pipelines más grandes.

### Cómo aplica en el bot

---

#### Azure AI Language

**Extracción de entidades para entender al socio mejor:**

```python
from azure.ai.language.conversations import ConversationAnalysisClient
from azure.ai.textanalytics import TextAnalyticsClient

text_client = TextAnalyticsClient(
    endpoint=os.environ["LANGUAGE_ENDPOINT"],
    credential=DefaultAzureCredential()
)

def extract_booking_entities(message: str) -> dict:
    """Extrae cancha, fecha y hora de un mensaje en lenguaje natural."""
    response = text_client.recognize_entities([message])
    
    entities = {}
    for entity in response[0].entities:
        if entity.category == "DateTime":
            entities["datetime"] = entity.text
        elif entity.category == "Location":
            entities["location"] = entity.text  # "cancha 3", "pádel 2"
    
    return entities

# "Quiero la cancha 3 de tenis el viernes a las 19"
# → {"datetime": "viernes a las 19", "location": "cancha 3 de tenis"}
```

**Detección de PII — proteger datos sensibles:**

Un socio podría mandar su número de tarjeta en el chat (pasa). El AI-103 engineer implementa detección antes de que el texto llegue al modelo:

```python
def sanitize_pii(text: str) -> str:
    """Detecta y enmascara PII antes de enviar al modelo."""
    response = text_client.recognize_pii_entities(
        [text],
        categories=["CreditCardNumber", "PhoneNumber", "Email"]
    )
    
    redacted = response[0].redacted_text
    # "Mi tarjeta es 4111 1111 1111 1111" → "Mi tarjeta es ████████████████"
    return redacted
```

**Resumen de conversación para Redis:**

Las conversaciones largas no se pueden pasar completas a GPT-4o (contexto caro y limitado). Mejor guardar un resumen:

```python
def summarize_conversation(conversation_history: list[str]) -> str:
    """Genera un resumen de la conversación para guardar en Redis."""
    full_text = "\n".join(conversation_history)
    
    response = text_client.extract_summary(
        [full_text],
        sentence_count=3
    )
    
    return response[0].sentences[0].text if response else ""

# El resumen se guarda en Redis con TTL de 24h
# Al día siguiente, el bot recupera el contexto sin repasar todo el historial
```

---

#### Speech — notas de voz de WhatsApp

WhatsApp soporta mensajes de voz. Los socios los mandan. El AI-103 engineer integra Whisper:

```python
import azure.cognitiveservices.speech as speechsdk

def transcribe_whatsapp_audio(audio_bytes: bytes, audio_format: str = "ogg") -> str:
    """
    Transcribe un audio de WhatsApp (.ogg con codec opus).
    WhatsApp manda ogg/opus — hay que convertir a wav primero.
    """
    # En producción: convertir con ffmpeg antes de pasar a Speech SDK
    # Aquí asumimos que ya está en formato compatible
    
    speech_config = speechsdk.SpeechConfig(
        subscription=os.environ["SPEECH_KEY"],
        region=os.environ["SPEECH_REGION"]
    )
    speech_config.speech_recognition_language = "es-AR"  # español rioplatense
    
    audio_stream = speechsdk.audio.PushAudioInputStream()
    audio_stream.write(audio_bytes)
    audio_stream.close()
    
    audio_config = speechsdk.audio.AudioConfig(stream=audio_stream)
    recognizer = speechsdk.SpeechRecognizer(speech_config, audio_config)
    
    result = recognizer.recognize_once()
    
    if result.reason == speechsdk.ResultReason.RecognizedSpeech:
        return result.text
    else:
        return ""  # log el error y continuar

# Flujo: audio OGG → ffmpeg → WAV → Speech SDK → texto → pipeline RAG normal
```

El examen pregunta sobre STT vs TTS, los modelos de Speech (Whisper vs Speech-to-Text nativo de Azure), y la configuración de idioma. Para el bot del club, Whisper (via Azure OpenAI) y Azure Speech SDK son intercambiables — la elección depende de latencia y formato de audio.

---

#### El RAG pipeline completo (conexión con Document Intelligence)

Este punto conecta Secciones 4 y 5. Los PDFs del club pasan por este pipeline para alimentar el knowledge base:

```
PDF del club (brochure, reglamento, tarifas)
    ↓ Azure AI Document Intelligence (Layout model)
    → Texto preservando estructura (tablas, secciones, headers)
    ↓ Chunking con overlap
    → Chunks de ~500 tokens
    ↓ text-embedding-3-small
    → Vectores
    ↓ Azure AI Search (upload + index)
    → Índice híbrido listo para queries
```

---

## Sección 5: Extracción de información (10–15%)

### Lo que evalúa el examen

Azure AI Document Intelligence (modelos preentrenados y custom), Azure AI Content Understanding (extracción de alto nivel), y cómo se integra la extracción estructurada en pipelines de IA.

### Cómo aplica en el bot

---

#### Documentos del club que hay que procesar

El club tiene estos tipos de documentos que el AI-103 engineer necesita procesar:

| Documento | Tipo | Qué extraer |
|---|---|---|
| Reglamento del club | PDF texto | Reglas, sanciones, procedimientos |
| Tarifas y planes | PDF con tablas | Precios por actividad, planes de membresía |
| Calendario de torneos | PDF/Excel | Fechas, categorías, inscripciones |
| Contratos de membresía | PDF formulario | Datos del socio, plan elegido, firma |
| Facturas de socios | PDF | Monto, período, servicios incluidos |

---

#### Azure AI Document Intelligence

```python
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest

doc_client = DocumentIntelligenceClient(
    endpoint=os.environ["DOCUMENT_INTELLIGENCE_ENDPOINT"],
    credential=DefaultAzureCredential()
)

def extract_club_document(pdf_bytes: bytes) -> dict:
    """
    Extrae texto estructurado de un PDF del club.
    Usa el modelo 'prebuilt-layout' que preserva tablas y estructura.
    """
    poller = doc_client.begin_analyze_document(
        model_id="prebuilt-layout",
        body=AnalyzeDocumentRequest(bytes_source=pdf_bytes)
    )
    result = poller.result()
    
    extracted = {
        "pages": [],
        "tables": [],
        "paragraphs": []
    }
    
    # Extraer texto por páginas (respeta orden de lectura)
    for page in result.pages:
        page_text = " ".join([line.content for line in (page.lines or [])])
        extracted["pages"].append(page_text)
    
    # Extraer tablas estructuradas (ej: tabla de precios)
    for table in (result.tables or []):
        table_data = []
        for cell in table.cells:
            table_data.append({
                "row": cell.row_index,
                "col": cell.column_index,
                "content": cell.content
            })
        extracted["tables"].append(table_data)
    
    return extracted
```

**Para documentos con formato conocido (ej: contratos de membresía):**

```python
# Si el contrato siempre tiene la misma estructura, se puede entrenar
# un modelo custom en Document Intelligence Studio
# → Después se usa igual pero con model_id="membresía-contrato-v1"

def extract_membership_contract(pdf_bytes: bytes) -> dict:
    poller = doc_client.begin_analyze_document(
        model_id="membresia-contrato-v1",  # modelo custom entrenado
        body=AnalyzeDocumentRequest(bytes_source=pdf_bytes)
    )
    result = poller.result()
    
    # Con modelo custom, los campos son directamente accesibles
    fields = result.documents[0].fields if result.documents else {}
    return {
        "nombre_socio": fields.get("NombreSocio", {}).content,
        "plan": fields.get("PlanMembresía", {}).content,
        "fecha_inicio": fields.get("FechaInicio", {}).content,
        "monto_mensual": fields.get("MontoMensual", {}).content,
    }
```

---

#### Azure AI Content Understanding

Content Understanding va un nivel más arriba que Document Intelligence. En vez de "dame el texto de esta página", decís "dame todas las actividades con sus precios y horarios de este brochure":

```python
# Content Understanding usa una API de análisis de alto nivel
# Ideal para extraer información semántica de documentos complejos

# Ejemplo conceptual (la API exacta varía según versión del SDK):
def extract_activities_from_brochure(document_url: str) -> list[dict]:
    """
    Extrae todas las actividades deportivas con precios y horarios
    del brochure del club, sin importar el formato del PDF.
    """
    # Content Understanding entiende la semántica, no solo el texto
    # → "Pádel: martes y jueves 18-20h, $500/mes" se convierte en estructura
    result = content_understanding_client.analyze(
        document_url=document_url,
        extraction_schema={
            "activities": [{
                "name": "string",
                "days": ["string"],
                "times": "string",
                "monthly_price": "number",
                "instructor": "string"
            }]
        }
    )
    return result["activities"]
```

**El output alimenta Azure AI Search:**

```python
def index_club_document(pdf_path: str, source_name: str):
    """Pipeline completo: PDF → extracción → chunks → embeddings → índice."""
    
    # 1. Extraer texto con Document Intelligence
    with open(pdf_path, "rb") as f:
        extracted = extract_club_document(f.read())
    
    # 2. Chunking del texto extraído
    all_text = "\n\n".join(extracted["pages"])
    chunks = chunk_text(all_text, chunk_size=500, overlap=50)
    
    # 3. Embeddings para cada chunk
    documents = []
    for i, chunk in enumerate(chunks):
        embedding = openai_client.embeddings.create(
            input=chunk, model="text-embedding-3-small"
        ).data[0].embedding
        
        documents.append({
            "id": f"{source_name}-chunk-{i}",
            "content": chunk,
            "content_vector": embedding,
            "source": source_name,
            "section": f"página {i // 3 + 1}"  # aproximado
        })
    
    # 4. Subir al índice de Azure AI Search
    search_client.upload_documents(documents=documents)
    print(f"Indexados {len(documents)} chunks de {source_name}")
```

El examen pregunta sobre los **modelos preentrenados** de Document Intelligence (`prebuilt-read`, `prebuilt-layout`, `prebuilt-invoice`, `prebuilt-receipt`) y cuándo conviene usar cada uno vs un modelo custom.

---

## Cómo se conectan las tres certificaciones en este proyecto

Al final del día, el bot del club es un proyecto donde tres ingenieros trabajan en distintas capas:

| Aspecto del proyecto | AB-100 | GH-600 | AI-103 |
|---|---|---|---|
| Define la arquitectura general | ✅ | | |
| Elige Azure AI Foundry como plataforma | ✅ | | |
| Diseña la topología de agentes | ✅ | | |
| Implementa en Python | | | ✅ |
| Despliega modelos en Foundry | | | ✅ |
| Construye el RAG pipeline | | | ✅ |
| Implementa los agentes (Agent Service) | | | ✅ |
| Configura MCP servers para las APIs | | ✅ | |
| CI/CD con GitHub Actions | | ✅ | |
| Copilot coding agent en el SDLC | | ✅ | |
| Content Safety y PII | | | ✅ |
| Evaluación de calidad (Azure AI Eval) | | ✅ | ✅ |
| Gobernanza y Azure Policy | ✅ | | |
| Monitoreo en producción | ✅ | ✅ | |
| Análisis de costos y cuotas | ✅ | | ✅ |
| Document Intelligence pipeline | | | ✅ |
| Gestión de secretos y Managed Identity | ✅ | ✅ | ✅ |

El punto clave para el examen AI-103: **el AI-103 engineer no define QUÉ construir, no automatiza CÓMO se despliega. Implementa EL QUÉ en Python**, con los servicios que eligió el AB-100, en la infraestructura que automatizó el GH-600.

---

## Resumen de snippets por sección

| Sección | Código clave que debés conocer |
|---|---|
| 1 — Planear | `AIProjectClient.from_connection_string()`, `DefaultAzureCredential()`, `ContentSafetyClient.analyze_text()` |
| 2 — Generativa | `SearchClient.search()` con `VectorizedQuery`, `client.agents.create_agent()`, `client.agents.create_and_process_run()`, `evaluate()` |
| 3 — Vision | `openai_client.chat.completions.create()` con `image_url` content, `ContentSafetyClient.analyze_image()` |
| 4 — Texto/Speech | `text_client.recognize_pii_entities()`, `text_client.extract_summary()`, `SpeechRecognizer.recognize_once()` |
| 5 — Extracción | `DocumentIntelligenceClient.begin_analyze_document()`, pipeline chunk→embed→index |
