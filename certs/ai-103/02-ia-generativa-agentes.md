# Sección 2: Implementación de soluciones de IA generativa y agente (30-35%)

> ⚠️⚠️ **Sección de MÁXIMO PESO en el examen** — representa entre el 30% y el 35% de la nota total.
> Dominar RAG, agentes, evaluación y observabilidad es crítico para aprobar AI-103.

---

## Índice de contenidos

- [2.1 Aplicaciones generativas con Foundry](#21-aplicaciones-generativas-con-foundry)
- [2.2 Crear agentes con Foundry](#22-crear-agentes-con-foundry)
- [2.3 Optimizar y operacionalizar sistemas de IA generativa](#23-optimizar-y-operacionalizar-sistemas-de-ia-generativa)
- [Recursos](#recursos)

---

## 2.1 Aplicaciones generativas con Foundry

### Conceptos clave

#### Modelos disponibles en Azure AI Foundry

| Tipo | Ejemplos | Uso principal |
|------|----------|--------------|
| **LLM** (Large Language Models) | GPT-4o, GPT-4-turbo, GPT-3.5-turbo | Chat, generación de texto, razonamiento |
| **SLM** (Small Language Models) | Phi-3 Mini/Medium, Mistral Small | Edge, bajo costo, inferencia local |
| **Modelos de código** | GPT-4o, Codex, StarCoder | Generación y análisis de código |
| **Multimodales** | GPT-4o (vision), DALL-E 3, Whisper | Imagen+texto, audio, video |
| **Embeddings** | text-embedding-3-small/large | Vectorización para RAG y búsqueda |

> 💡 **Clave para el examen:** Phi-3 y Phi-3.5 son los SLMs de Microsoft. Se despliegan también como serverless o en compute dedicado.

#### Desplegar un modelo en Foundry

Existen tres modalidades de despliegue:

1. **Serverless (pay-per-token):** Sin infraestructura gestionada, ideal para producción a escala variable.
2. **Managed online endpoint:** Compute dedicado en Azure ML, control total del escalado.
3. **Azure OpenAI Service:** Para modelos OpenAI (GPT-4o, embeddings). Recursos propios de Azure OpenAI.

```python
# Consumir un modelo desplegado como serverless endpoint
from azure.ai.inference import ChatCompletionsClient
from azure.ai.inference.models import SystemMessage, UserMessage
from azure.core.credentials import AzureKeyCredential

client = ChatCompletionsClient(
    endpoint="https://<resource>.inference.ai.azure.com",
    credential=AzureKeyCredential("<api-key>")
)

response = client.complete(
    messages=[
        SystemMessage(content="Eres un asistente experto en deportes."),
        UserMessage(content="¿Cuáles son los beneficios del entrenamiento HIIT?")
    ],
    model="Phi-3-medium-128k-instruct",
    temperature=0.7,
    max_tokens=512
)

print(response.choices[0].message.content)
```

```python
# Consumir Azure OpenAI (GPT-4o) via SDK
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://<resource>.openai.azure.com/",
    api_key="<api-key>",
    api_version="2024-02-01"
)

response = client.chat.completions.create(
    model="gpt-4o",           # nombre del deployment, no del modelo base
    messages=[
        {"role": "system", "content": "Eres un asistente deportivo."},
        {"role": "user", "content": "Explica qué es el VO2 máx."}
    ],
    temperature=0,
    max_tokens=300
)
print(response.choices[0].message.content)
```

---

### RAG: Retrieval-Augmented Generation

#### ¿Qué es RAG?

RAG es un patrón arquitectónico que **enriquece el prompt** con información recuperada de una fuente externa (documentos, base de datos vectorial) antes de enviarla al LLM. Esto permite:
- Respuestas fundamentadas en datos reales y actualizados.
- Reducción de fabricaciones (hallucinations).
- Control del conocimiento sin re-entrenar el modelo.

#### Arquitectura RAG completa

```
Documentos fuente
      │
      ▼
[Chunking + Embedding]
      │
      ▼
Azure AI Search (índice vectorial)
      │
      ├──── Query time ────────────────────────────────────┐
      │                                                    │
Usuario → embed query → búsqueda vectorial → top-k chunks │
                                                          │
                        Prompt: system + contexto + query ◄┘
                                       │
                               GPT-4o / modelo
                                       │
                               Respuesta fundamentada
```

#### Implementación paso a paso

**Paso 1 — Indexar documentos en Azure AI Search**

```python
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex, SimpleField, SearchableField,
    SearchField, SearchFieldDataType, VectorSearch,
    HnswAlgorithmConfiguration, VectorSearchProfile
)
from azure.core.credentials import AzureKeyCredential
from openai import AzureOpenAI
import json

# Clientes
openai_client = AzureOpenAI(
    azure_endpoint="https://<oai-resource>.openai.azure.com/",
    api_key="<oai-key>",
    api_version="2024-02-01"
)

index_client = SearchIndexClient(
    endpoint="https://<search-resource>.search.windows.net",
    credential=AzureKeyCredential("<search-key>")
)

# Crear índice con soporte vectorial
fields = [
    SimpleField(name="id", type=SearchFieldDataType.String, key=True),
    SearchableField(name="content", type=SearchFieldDataType.String),
    SearchField(
        name="content_vector",
        type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
        searchable=True,
        vector_search_dimensions=1536,
        vector_search_profile_name="myHnswProfile"
    )
]

vector_search = VectorSearch(
    algorithms=[HnswAlgorithmConfiguration(name="myHnsw")],
    profiles=[VectorSearchProfile(name="myHnswProfile", algorithm_configuration_name="myHnsw")]
)

index = SearchIndex(name="sports-docs", fields=fields, vector_search=vector_search)
index_client.create_or_update_index(index)

# Función para generar embeddings
def get_embedding(text: str) -> list[float]:
    response = openai_client.embeddings.create(
        input=text,
        model="text-embedding-3-small"  # nombre del deployment
    )
    return response.data[0].embedding

# Indexar documentos (chunkeados previamente)
search_client = SearchClient(
    endpoint="https://<search-resource>.search.windows.net",
    index_name="sports-docs",
    credential=AzureKeyCredential("<search-key>")
)

documents = [
    {"id": "1", "content": "El HIIT mejora la capacidad cardiovascular..."},
    {"id": "2", "content": "El VO2 máx es la cantidad máxima de oxígeno..."},
]

docs_with_vectors = []
for doc in documents:
    doc["content_vector"] = get_embedding(doc["content"])
    docs_with_vectors.append(doc)

search_client.upload_documents(documents=docs_with_vectors)
print("Documentos indexados correctamente.")
```

**Paso 2 — Consultar con búsqueda híbrida (vector + keyword)**

```python
from azure.search.documents.models import VectorizedQuery

def rag_query(user_question: str, top_k: int = 5) -> str:
    # 1. Generar embedding de la pregunta
    query_vector = get_embedding(user_question)

    # 2. Búsqueda híbrida: vectorial + texto completo
    vector_query = VectorizedQuery(
        vector=query_vector,
        k_nearest_neighbors=top_k,
        fields="content_vector"
    )

    results = search_client.search(
        search_text=user_question,          # keyword search
        vector_queries=[vector_query],      # vector search
        select=["content"],
        top=top_k
    )

    # 3. Construir contexto
    context_chunks = [r["content"] for r in results]
    context = "\n\n---\n\n".join(context_chunks)

    # 4. Construir prompt aumentado
    system_prompt = """Eres un asistente deportivo experto.
Responde ÚNICAMENTE basándote en el contexto proporcionado.
Si la información no está en el contexto, indica que no lo sabes.
No inventes datos."""

    augmented_prompt = f"""Contexto recuperado:
{context}

Pregunta del usuario: {user_question}"""

    # 5. Llamar al LLM
    response = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": augmented_prompt}
        ],
        temperature=0,        # 0 = determinístico, mejor para RAG
        max_tokens=800
    )

    return response.choices[0].message.content

# Uso
answer = rag_query("¿Cómo mejora el HIIT la resistencia cardiovascular?")
print(answer)
```

> 💡 **Clave para el examen:** La búsqueda **híbrida** (vector + keyword) con re-ranking semántico suele dar mejores resultados que solo vectorial. Azure AI Search soporta esto con `semantic_configuration_name`.

---

### Flujos de trabajo y pipelines multi-paso

#### Prompt Flow en Azure AI Foundry

Prompt Flow es la herramienta visual y programática de Foundry para construir pipelines de IA:

```
Nodos disponibles:
  - LLM node: llamada a un modelo
  - Python node: código Python personalizado
  - Tool node: búsqueda, API externa
  - Conditional: ramificación lógica
```

**Tipos de flujos:**
| Tipo | Descripción | Caso de uso |
|------|-------------|-------------|
| **Standard flow** | Secuencia de nodos LLM+Python | Chatbot, summarización |
| **Chat flow** | Historial de conversación integrado | Asistente conversacional |
| **Evaluation flow** | Métricas automáticas sobre outputs | QA, testing |

```python
# Ejemplo: pipeline multi-paso con Prompt Flow SDK
from promptflow.client import PFClient
from promptflow.entities import Run

pf = PFClient()

# Ejecutar un flow local
run = pf.run(
    flow="./my_rag_flow",          # directorio con flow.dag.yaml
    data="./test_data.jsonl",      # datos de entrada
    column_mapping={
        "question": "${data.question}",
        "chat_history": "${data.history}"
    }
)

pf.stream(run)  # ver output en tiempo real
details = pf.get_details(run)
print(details)
```

#### Pipeline de razonamiento multi-paso (Chain-of-Thought pipeline)

```python
def multi_step_reasoning_pipeline(user_query: str) -> dict:
    """
    Pipeline: Análisis → Recuperación → Razonamiento → Respuesta
    """
    # Paso 1: Analizar y descomponer la pregunta
    analysis_response = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "Analiza la pregunta y extrae los conceptos clave. Responde en JSON."},
            {"role": "user", "content": f"Pregunta: {user_query}"}
        ],
        response_format={"type": "json_object"},
        temperature=0
    )
    analysis = json.loads(analysis_response.choices[0].message.content)
    keywords = analysis.get("keywords", [user_query])

    # Paso 2: Recuperar contexto para cada keyword
    all_context = []
    for kw in keywords[:3]:  # limitar búsquedas
        result = rag_query(kw, top_k=3)
        all_context.append(result)

    # Paso 3: Razonamiento final con todo el contexto
    final_response = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "Sintetiza la información y responde de forma completa."},
            {"role": "user", "content": f"Información recuperada:\n{chr(10).join(all_context)}\n\nPregunta original: {user_query}"}
        ],
        temperature=0.3
    )

    return {
        "query": user_query,
        "analysis": analysis,
        "answer": final_response.choices[0].message.content
    }
```

---

### Evaluación de modelos y aplicaciones

#### Métricas de evaluación en Azure AI Foundry

| Métrica | Descripción | Rango | Herramienta |
|---------|-------------|-------|-------------|
| **Groundedness** | ¿La respuesta está fundamentada en el contexto? | 1-5 | Foundry Evaluate |
| **Relevance** | ¿La respuesta responde la pregunta? | 1-5 | Foundry Evaluate |
| **Coherence** | ¿Es la respuesta bien estructurada y lógica? | 1-5 | Foundry Evaluate |
| **Fluency** | ¿El lenguaje es natural y correcto? | 1-5 | Foundry Evaluate |
| **F1 Score** | Precisión+recall para QA extractiva | 0-1 | Custom |
| **Similarity** | Similitud semántica con respuesta de referencia | 0-1 | Custom |
| **Hallucination rate** | Porcentaje de fabricaciones detectadas | 0-1 | Foundry Safety |
| **Safety** | Contenido dañino, sesgos, datos personales | Pass/Fail | Content Safety |

```python
# Evaluación programática con Azure AI Evaluation SDK
from azure.ai.evaluation import (
    GroundednessEvaluator,
    RelevanceEvaluator,
    CoherenceEvaluator,
    FluencyEvaluator,
    evaluate
)

# Configurar el modelo juez (judge model)
model_config = {
    "azure_endpoint": "https://<resource>.openai.azure.com/",
    "api_key": "<key>",
    "azure_deployment": "gpt-4o",
    "api_version": "2024-02-01"
}

# Instanciar evaluadores
groundedness_eval = GroundednessEvaluator(model_config=model_config)
relevance_eval    = RelevanceEvaluator(model_config=model_config)
coherence_eval    = CoherenceEvaluator(model_config=model_config)
fluency_eval      = FluencyEvaluator(model_config=model_config)

# Ejecutar evaluación sobre un dataset
results = evaluate(
    data="./eval_dataset.jsonl",    # campos: query, context, response, ground_truth
    evaluators={
        "groundedness": groundedness_eval,
        "relevance":    relevance_eval,
        "coherence":    coherence_eval,
        "fluency":      fluency_eval
    },
    output_path="./eval_results.json"
)

print(f"Groundedness promedio: {results['metrics']['groundedness.groundedness']:.2f}")
print(f"Relevance promedio:    {results['metrics']['relevance.relevance']:.2f}")
```

#### Detectar fabricaciones (hallucinations)

```python
from azure.ai.evaluation import HallucinationEvaluator

hallucination_eval = HallucinationEvaluator(model_config=model_config)

# Evaluar una respuesta individual
score = hallucination_eval(
    query="¿Cuántas calorías quema 30 minutos de HIIT?",
    context="El HIIT de 30 minutos quema entre 300-450 calorías dependiendo del peso.",
    response="El HIIT de 30 minutos quema entre 300-450 calorías."  # fundamentado ✓
)
print(score)  # {"hallucination": 0, "reason": "Response is grounded in context."}

# Respuesta fabricada
score2 = hallucination_eval(
    query="¿Cuántas calorías quema 30 minutos de HIIT?",
    context="El HIIT de 30 minutos quema entre 300-450 calorías.",
    response="El HIIT quema exactamente 720 calorías en 30 minutos."  # fabricado ✗
)
print(score2)  # {"hallucination": 1, "reason": "The figure 720 is not in the context."}
```

---

### Integrar flujos generativos con SDK y conectores de Foundry

```python
# Conectar app a un proyecto Foundry
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

# La cadena de conexión está en: Foundry Portal → Project → Settings → Connection string
client = AIProjectClient.from_connection_string(
    conn_str="eastus.api.azureml.ms;<subscription-id>;<resource-group>;<project-name>",
    credential=DefaultAzureCredential()
)

# Obtener cliente OpenAI desde el proyecto (usa las conexiones configuradas)
openai_client = client.inference.get_azure_openai_client(api_version="2024-02-01")

response = openai_client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hola, ¿cómo puedo empezar con HIIT?"}]
)
print(response.choices[0].message.content)

# Obtener cliente de búsqueda desde el proyecto
from azure.search.documents import SearchClient
search_connection = client.connections.get("<search-connection-name>")
search_client = SearchClient(
    endpoint=search_connection.endpoint_url,
    index_name="sports-docs",
    credential=search_connection.key_credential
)
```

### Puntos clave para el examen — Sección 2.1

- ✅ RAG = chunk → embed → index → retrieve → augment prompt → generate.
- ✅ `temperature=0` para respuestas determinísticas y factuales (RAG); `0.7-1.0` para creatividad.
- ✅ **Groundedness** es la métrica más importante para RAG (¿está fundamentado?).
- ✅ Búsqueda híbrida (vector + keyword) supera a la búsqueda solo vectorial.
- ✅ `AIProjectClient.from_connection_string()` es la forma oficial de conectar apps a Foundry.
- ✅ Prompt Flow soporta flujos Standard, Chat y Evaluation.
- ✅ Los modelos Phi-3/Phi-3.5 son los SLMs de Microsoft disponibles en Foundry.

---

## 2.2 Crear agentes con Foundry

### Conceptos clave

#### ¿Qué es un agente de IA?

Un agente es un sistema de IA que:
1. **Percibe** el entorno (mensajes, herramientas disponibles, memoria).
2. **Razona** sobre qué acción tomar (usando un LLM).
3. **Actúa** llamando herramientas o APIs.
4. **Observa** el resultado y repite el ciclo hasta completar el objetivo.

```
┌─────────────────────────────────────┐
│           AGENTE                    │
│                                     │
│  Instrucciones + Herramientas       │
│         ↓                           │
│  [LLM] ──→ ¿Usar herramienta?      │
│         ↓          ↓               │
│  Responder    [Tool Call]           │
│               ↓                    │
│          Resultado                  │
│               ↓                    │
│          [LLM] → Respuesta final   │
└─────────────────────────────────────┘
```

#### Componentes de un agente en Foundry

| Componente | Descripción |
|------------|-------------|
| **Model** | LLM base (ej: gpt-4o) |
| **Instructions** | System prompt: persona, objetivo, restricciones |
| **Tools** | Funciones, APIs, búsqueda, code interpreter |
| **Thread** | Historial de conversación (memoria) |
| **Run** | Ejecución de un turno del agente |
| **Messages** | Mensajes del usuario y del agente |

---

### Crear y ejecutar un agente básico

```python
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    Agent, AgentThread, AgentRun,
    MessageRole, RunStatus
)
from azure.identity import DefaultAzureCredential
import time

# Conectar al proyecto Foundry
client = AIProjectClient.from_connection_string(
    conn_str="<connection-string>",
    credential=DefaultAzureCredential()
)

# 1. Crear el agente (se persiste en el proyecto)
agent = client.agents.create_agent(
    model="gpt-4o",
    name="AsistenteDeportivo",
    instructions="""Eres un asistente experto en entrenamiento deportivo y nutrición.
Tu objetivo es ayudar a los usuarios a mejorar su rendimiento deportivo.
Responde siempre en español, de forma clara y motivadora.
Cuando necesites calcular algo, usa la herramienta de código.
Cuando necesites información actualizada, usa la búsqueda web."""
)
print(f"Agente creado: {agent.id}")

# 2. Crear un hilo de conversación
thread = client.agents.create_thread()
print(f"Thread creado: {thread.id}")

# 3. Enviar mensaje del usuario
message = client.agents.create_message(
    thread_id=thread.id,
    role=MessageRole.USER,
    content="¿Cuál debería ser mi frecuencia cardíaca objetivo para quemar grasa?"
)

# 4. Ejecutar el agente y esperar resultado
run = client.agents.create_and_process_run(
    thread_id=thread.id,
    agent_id=agent.id
)

print(f"Run status: {run.status}")

# 5. Leer la respuesta
if run.status == RunStatus.COMPLETED:
    messages = client.agents.list_messages(thread_id=thread.id)
    last_msg = messages.get_last_text_message_by_role(MessageRole.ASSISTANT)
    print(f"\nAsistente: {last_msg.text.value}")
elif run.status == RunStatus.FAILED:
    print(f"Error: {run.last_error}")
```

---

### Herramientas integradas del agente

#### 1. Code Interpreter — ejecutar código Python

```python
from azure.ai.projects.models import CodeInterpreterTool

# Agregar code_interpreter al agente
agent_with_code = client.agents.create_agent(
    model="gpt-4o",
    name="AnalistaBiometrico",
    instructions="Eres un analista que calcula métricas deportivas. Usa código Python para cálculos precisos.",
    tools=CodeInterpreterTool().definitions  # sandbox Python seguro
)

# El agente puede ahora escribir y ejecutar código:
# Usuario: "Calcula mi IMC: peso 75kg, altura 1.78m"
# Agente: genera código → ejecuta → devuelve resultado
```

#### 2. File Search — RAG integrado

```python
from azure.ai.projects.models import FileSearchTool
import os

# Subir archivos al proyecto
with open("guia_entrenamiento.pdf", "rb") as f:
    uploaded_file = client.agents.upload_file_and_poll(
        file=f,
        purpose="assistants"
    )

# Crear vector store con los archivos
vector_store = client.agents.create_vector_store_and_poll(
    file_ids=[uploaded_file.id],
    name="DocumentosDeportivos"
)

# Agente con file_search (RAG lite automático)
agent_with_files = client.agents.create_agent(
    model="gpt-4o",
    name="AgenteDocumentos",
    instructions="Responde preguntas sobre entrenamiento basándote en los documentos proporcionados.",
    tools=FileSearchTool().definitions,
    tool_resources=FileSearchTool(vector_store_ids=[vector_store.id]).resources
)
```

#### 3. Bing Grounding — búsqueda web

```python
from azure.ai.projects.models import BingGroundingTool

# Obtener la conexión Bing configurada en el proyecto
bing_connection = client.connections.get("<bing-connection-name>")

agent_with_bing = client.agents.create_agent(
    model="gpt-4o",
    name="AgenteNoticias",
    instructions="Busca información actualizada sobre eventos deportivos y competiciones.",
    tools=BingGroundingTool(connection_id=bing_connection.id).definitions
)
```

#### 4. Function Calling — funciones personalizadas

```python
from azure.ai.projects.models import FunctionTool
import json

# Definir función como herramienta JSON Schema
def get_training_plan(sport: str, level: str, days_per_week: int) -> str:
    """Genera un plan de entrenamiento personalizado."""
    # Lógica real aquí (API, base de datos, etc.)
    return json.dumps({
        "sport": sport,
        "level": level,
        "sessions": days_per_week,
        "plan": f"Plan {level} para {sport}: {days_per_week} sesiones/semana"
    })

# Schema JSON para que el agente sepa cómo llamar la función
training_function_def = {
    "type": "function",
    "function": {
        "name": "get_training_plan",
        "description": "Genera un plan de entrenamiento personalizado para un deporte y nivel específico.",
        "parameters": {
            "type": "object",
            "properties": {
                "sport": {
                    "type": "string",
                    "description": "Deporte (ej: 'fútbol', 'natación', 'ciclismo')"
                },
                "level": {
                    "type": "string",
                    "enum": ["principiante", "intermedio", "avanzado"],
                    "description": "Nivel del deportista"
                },
                "days_per_week": {
                    "type": "integer",
                    "description": "Número de días de entrenamiento por semana (1-7)",
                    "minimum": 1,
                    "maximum": 7
                }
            },
            "required": ["sport", "level", "days_per_week"]
        }
    }
}

# Crear agente con función personalizada
functions = FunctionTool(functions={get_training_plan})
agent_with_functions = client.agents.create_agent(
    model="gpt-4o",
    name="PlanificadorDeportivo",
    instructions="Ayuda a crear planes de entrenamiento. Usa get_training_plan cuando el usuario pida un plan.",
    tools=functions.definitions
)
```

#### 5. Azure AI Search como herramienta del agente

```python
from azure.ai.projects.models import AzureAISearchTool

# Conectar búsqueda del proyecto
search_connection = client.connections.get("<search-connection-name>")

search_tool = AzureAISearchTool(
    index_connection_id=search_connection.id,
    index_name="sports-docs"
)

agent_with_search = client.agents.create_agent(
    model="gpt-4o",
    name="AgenteKB",
    instructions="Busca en la base de conocimiento para responder preguntas deportivas.",
    tools=search_tool.definitions,
    tool_resources=search_tool.resources
)
```

---

### Memoria de conversación y gestión de hilos

```python
# El thread mantiene el historial automáticamente
def conversacion_multiturno(agent_id: str, thread_id: str, user_message: str) -> str:
    """Continuar una conversación existente."""
    # Agregar mensaje al thread existente
    client.agents.create_message(
        thread_id=thread_id,
        role=MessageRole.USER,
        content=user_message
    )

    # Ejecutar el agente (tiene acceso a todo el historial del thread)
    run = client.agents.create_and_process_run(
        thread_id=thread_id,
        agent_id=agent_id
    )

    if run.status == RunStatus.COMPLETED:
        messages = client.agents.list_messages(thread_id=thread_id)
        return messages.get_last_text_message_by_role(MessageRole.ASSISTANT).text.value
    return f"Error: {run.last_error}"

# Simular conversación
thread = client.agents.create_thread()

respuesta1 = conversacion_multiturno(agent.id, thread.id, "Quiero empezar a correr.")
print(f"Agente: {respuesta1}")

respuesta2 = conversacion_multiturno(agent.id, thread.id, "¿Cuántos km debo correr la primera semana?")
print(f"Agente: {respuesta2}")  # El agente recuerda el contexto anterior
```

---

### Soluciones multiagente orquestadas

#### Patrón de orquestación con Foundry

```
                    ┌──────────────────┐
  Usuario ──────→  │ Agente Orquestador│
                    │   (Router)        │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ Agente Nutri │ │ Agente Train │ │ Agente Stats │
    │ (Nutrición)  │ │ (Plan.Entr.) │ │ (Analytics)  │
    └──────────────┘ └──────────────┘ └──────────────┘
```

```python
# Implementar multiagente con function calling entre agentes
import json

# Definir agentes especializados
agent_nutricion = client.agents.create_agent(
    model="gpt-4o",
    name="EspecialistaNutricion",
    instructions="Eres especialista en nutrición deportiva. Calcula macros, hidratación y suplementación."
)

agent_entrenamiento = client.agents.create_agent(
    model="gpt-4o",
    name="EspecialistaEntrenamiento",
    instructions="Eres especialista en planes de entrenamiento. Diseña rutinas personalizadas."
)

# Funciones que llaman a los agentes especializados
def consultar_nutricion(consulta: str) -> str:
    """Consultar al agente de nutrición."""
    thread = client.agents.create_thread()
    client.agents.create_message(thread.id, MessageRole.USER, consulta)
    run = client.agents.create_and_process_run(thread.id, agent_nutricion.id)
    msgs = client.agents.list_messages(thread.id)
    return msgs.get_last_text_message_by_role(MessageRole.ASSISTANT).text.value

def consultar_entrenamiento(consulta: str) -> str:
    """Consultar al agente de entrenamiento."""
    thread = client.agents.create_thread()
    client.agents.create_message(thread.id, MessageRole.USER, consulta)
    run = client.agents.create_and_process_run(thread.id, agent_entrenamiento.id)
    msgs = client.agents.list_messages(thread.id)
    return msgs.get_last_text_message_by_role(MessageRole.ASSISTANT).text.value

# Agente orquestador con acceso a los especializados
funciones_especializadas = FunctionTool(functions={consultar_nutricion, consultar_entrenamiento})

agent_orquestador = client.agents.create_agent(
    model="gpt-4o",
    name="OrquestadorDeportivo",
    instructions="""Eres el coordinador principal. 
Cuando el usuario pida información de nutrición, usa consultar_nutricion.
Cuando pida planes de entrenamiento, usa consultar_entrenamiento.
Sintetiza las respuestas de forma coherente.""",
    tools=funciones_especializadas.definitions
)
```

#### Orquestación con Semantic Kernel (Python)

```python
import asyncio
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion
from semantic_kernel.agents import ChatCompletionAgent, AgentGroupChat
from semantic_kernel.agents.strategies import TerminationStrategy

# Configurar el kernel
kernel = Kernel()
kernel.add_service(AzureChatCompletion(
    deployment_name="gpt-4o",
    endpoint="https://<resource>.openai.azure.com/",
    api_key="<key>"
))

# Agentes especializados con SK
agente_nutricion = ChatCompletionAgent(
    kernel=kernel,
    name="EspecialistaNutricion",
    instructions="Eres especialista en nutrición deportiva. Solo respondes preguntas de nutrición."
)

agente_entrenamiento = ChatCompletionAgent(
    kernel=kernel,
    name="EspecialistaEntrenamiento",
    instructions="Eres especialista en planes de entrenamiento. Solo respondes sobre rutinas."
)

# Estrategia de terminación personalizada
class MaxTurnsStrategy(TerminationStrategy):
    async def should_agent_terminate(self, agent, history) -> bool:
        return len(history) >= 6  # Terminar después de 6 mensajes

# Chat grupal entre agentes
async def multiagente_chat(pregunta: str):
    group_chat = AgentGroupChat(
        agents=[agente_nutricion, agente_entrenamiento],
        termination_strategy=MaxTurnsStrategy()
    )

    await group_chat.add_chat_message(role="user", content=pregunta)

    async for response in group_chat.invoke():
        print(f"[{response.name}]: {response.content}")

asyncio.run(multiagente_chat("Necesito un plan integral de entrenamiento y dieta para un maratón."))
```

---

### Flujos autónomos con guardrails y controles de aprobación

```python
from azure.ai.projects.models import RunStatus, RequiredAction

def ejecutar_con_aprobacion_humana(thread_id: str, agent_id: str):
    """
    Ejecutar agente con Human-in-the-Loop:
    El agente se detiene antes de ejecutar acciones críticas y espera aprobación.
    """
    run = client.agents.create_run(
        thread_id=thread_id,
        agent_id=agent_id
    )

    while True:
        # Polling del estado del run
        run = client.agents.get_run(thread_id=thread_id, run_id=run.id)

        if run.status == RunStatus.COMPLETED:
            messages = client.agents.list_messages(thread_id=thread_id)
            return messages.get_last_text_message_by_role(MessageRole.ASSISTANT).text.value

        elif run.status == RunStatus.REQUIRES_ACTION:
            # El agente quiere llamar una herramienta: pedir aprobación humana
            required_action = run.required_action
            tool_calls = required_action.submit_tool_outputs.tool_calls

            print("\n⚠️  El agente solicita ejecutar acciones:")
            tool_outputs = []

            for tool_call in tool_calls:
                fn_name = tool_call.function.name
                fn_args = json.loads(tool_call.function.arguments)
                print(f"  → Función: {fn_name}")
                print(f"  → Argumentos: {fn_args}")

                # CONTROL DE APROBACIÓN HUMANA
                aprobado = input(f"\n¿Aprobar ejecución de '{fn_name}'? (s/n): ").lower() == 's'

                if aprobado:
                    # Ejecutar la función real
                    resultado = ejecutar_funcion(fn_name, fn_args)
                else:
                    resultado = "Acción rechazada por el operador humano."

                tool_outputs.append({
                    "tool_call_id": tool_call.id,
                    "output": str(resultado)
                })

            # Devolver resultados al agente para continuar
            client.agents.submit_tool_outputs_to_run(
                thread_id=thread_id,
                run_id=run.id,
                tool_outputs=tool_outputs
            )

        elif run.status in [RunStatus.FAILED, RunStatus.CANCELLED, RunStatus.EXPIRED]:
            raise Exception(f"Run fallido: {run.status} - {run.last_error}")

        else:
            time.sleep(1)  # Esperar si está en progreso

def ejecutar_funcion(nombre: str, args: dict) -> str:
    """Dispatcher de funciones disponibles."""
    funciones_disponibles = {
        "get_training_plan": get_training_plan,
        "consultar_nutricion": consultar_nutricion,
    }
    if nombre in funciones_disponibles:
        return funciones_disponibles[nombre](**args)
    return f"Función '{nombre}' no reconocida."
```

---

### Supervisión y evaluación del comportamiento del agente

```python
# Analizar runs y mensajes para debugging
def analizar_comportamiento_agente(thread_id: str, run_id: str):
    """Auditar el comportamiento completo de un run."""
    # Obtener pasos del run (steps = razonamiento interno del agente)
    run_steps = client.agents.list_run_steps(
        thread_id=thread_id,
        run_id=run_id
    )

    print(f"=== Análisis del Run {run_id} ===\n")
    for step in run_steps.data:
        print(f"Paso: {step.type} | Estado: {step.status}")

        if step.type == "tool_calls":
            for tool_call in step.step_details.tool_calls:
                print(f"  ↳ Herramienta: {tool_call.type}")
                if hasattr(tool_call, 'function'):
                    print(f"    Función: {tool_call.function.name}")
                    print(f"    Args: {tool_call.function.arguments}")
                    print(f"    Output: {tool_call.function.output}")

        elif step.type == "message_creation":
            msg_id = step.step_details.message_creation.message_id
            print(f"  ↳ Mensaje creado: {msg_id}")

    # Obtener todos los mensajes del thread
    messages = client.agents.list_messages(thread_id=thread_id)
    print(f"\nTotal mensajes en thread: {len(messages.data)}")
```

### Puntos clave para el examen — Sección 2.2

- ✅ **Thread** = historial de conversación. Un agente puede tener múltiples threads.
- ✅ **Run** = ejecución de un turno. Status: `queued → in_progress → requires_action → completed/failed`.
- ✅ `REQUIRES_ACTION` = el agente quiere llamar una función → Human-in-the-Loop.
- ✅ `code_interpreter` ejecuta código Python en sandbox seguro.
- ✅ `file_search` implementa RAG automático sobre archivos subidos.
- ✅ `bing_grounding` proporciona información web actualizada al agente.
- ✅ Multiagente: cada agente es especialista; el orquestador los coordina via function calling.
- ✅ Semantic Kernel: `ChatCompletionAgent` + `AgentGroupChat` para orquestación Python.
- ✅ Los guardrails se implementan interceptando `REQUIRES_ACTION` antes de aprobar tool calls.

---

## 2.3 Optimizar y operacionalizar sistemas de IA generativa

### Conceptos clave

### Prompt Engineering

#### Estructura del system prompt

```
[PERSONA]     → Quién es el asistente y su dominio de expertise
[CONTEXTO]    → Información relevante del dominio
[RESTRICCIONES] → Qué NO debe hacer (scope, temas prohibidos)
[FORMATO]     → Cómo debe responder (JSON, markdown, longitud)
```

```python
system_prompt_optimizado = """
# Persona
Eres ClubBot, el asistente oficial del Club Deportivo Elite.
Eres experto en entrenamiento físico, nutrición deportiva y gestión de socios.

# Contexto
Operas para un club deportivo con las siguientes instalaciones:
- Piscina olímpica (50m), gimnasio de pesas, pistas de tenis y pádel.
- Horario: Lunes-Viernes 6:00-23:00, Sábado-Domingo 8:00-21:00.

# Restricciones
- NO brindes asesoramiento médico ni diagnósticos de lesiones.
- NO discutas precios ni descuentos sin consultar al personal administrativo.
- Si no conoces la respuesta, di "No tengo esa información" sin inventar datos.
- Responde siempre en el idioma del usuario.

# Formato de respuesta
- Usa listas cuando enumeres más de 3 items.
- Para planes de entrenamiento, usa formato de tabla cuando sea posible.
- Máximo 300 palabras por respuesta, a menos que el usuario pida detalle.
"""
```

#### Patrones de prompting

```python
# Zero-shot: instrucción directa sin ejemplos
zero_shot = "Clasifica esta reseña como POSITIVA, NEGATIVA o NEUTRAL: '{reseña}'"

# Few-shot: con ejemplos para guiar el comportamiento
few_shot = """
Clasifica reseñas de instalaciones deportivas.

Ejemplos:
Reseña: "Las piscinas están impolutas y el agua perfecta." → POSITIVA
Reseña: "Los vestuarios huelen mal y están sucios." → NEGATIVA
Reseña: "Las instalaciones son normales para un club de este precio." → NEUTRAL

Ahora clasifica: "{reseña}"
"""

# Chain-of-Thought: razonamiento paso a paso
chain_of_thought = """
Determina si este plan de entrenamiento es adecuado para un principiante.
Piensa paso a paso:
1. Analiza la intensidad de los ejercicios.
2. Evalúa el volumen total (series x repeticiones).
3. Considera el tiempo de recuperación.
4. Concluye si es apropiado para un principiante.

Plan: {plan}

Razonamiento:"""
```

---

### Ajuste de parámetros de generación

| Parámetro | Rango | Efecto | Valor recomendado |
|-----------|-------|--------|------------------|
| **temperature** | 0.0 - 2.0 | Aleatoriedad del output | 0 (factual), 0.7 (balanceado), 1.0+ (creativo) |
| **top_p** | 0.0 - 1.0 | Nucleus sampling (tokens candidatos) | 0.9 por defecto; bajar para más precisión |
| **max_tokens** | 1 - límite modelo | Longitud máxima de respuesta | Ajustar según caso de uso |
| **frequency_penalty** | -2.0 - 2.0 | Penaliza tokens frecuentes (reduce repetición) | 0.3-0.5 para texto largo |
| **presence_penalty** | -2.0 - 2.0 | Penaliza tokens ya usados (fomenta novedad) | 0.3-0.6 para creatividad |
| **stop** | lista de strings | Detener generación al encontrar token | `["\n\n", "###"]` |

> ⚠️ **Importante para el examen:** No usar `temperature` y `top_p` con valores altos simultáneamente. OpenAI recomienda modificar solo uno de los dos.

```python
# Configuración para RAG (respuestas factuales)
rag_config = {
    "temperature": 0.0,
    "top_p": 1.0,
    "max_tokens": 500,
    "frequency_penalty": 0.0,
    "presence_penalty": 0.0
}

# Configuración para generación creativa (descripciones, marketing)
creative_config = {
    "temperature": 0.9,
    "top_p": 0.95,
    "max_tokens": 800,
    "frequency_penalty": 0.4,
    "presence_penalty": 0.3
}

# Respuesta estructurada en JSON
structured_response = openai_client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Extrae datos del texto y devuelve JSON válido."},
        {"role": "user", "content": "Juan García, 28 años, 5 sesiones/semana, objetivo: perder peso."}
    ],
    response_format={"type": "json_object"},  # Garantiza JSON válido
    temperature=0
)
datos = json.loads(structured_response.choices[0].message.content)
```

---

### Reflexión del modelo y auto-crítica

```python
def generar_con_reflexion(prompt_usuario: str) -> dict:
    """
    Patrón Reflexion: generar → criticar → mejorar.
    Reduce errores y mejora la calidad del output final.
    """
    # Paso 1: Generación inicial
    respuesta_inicial = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "Eres un experto en nutrición deportiva."},
            {"role": "user", "content": prompt_usuario}
        ],
        temperature=0.7
    ).choices[0].message.content

    # Paso 2: Auto-crítica — el modelo evalúa su propia respuesta
    critica = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": """Eres un crítico experto en nutrición deportiva.
Evalúa la siguiente respuesta en términos de:
1. Precisión científica
2. Completitud de la información
3. Posibles riesgos o advertencias omitidas
4. Claridad y estructura
Sé específico sobre qué mejorar."""},
            {"role": "user", "content": f"Respuesta a evaluar:\n{respuesta_inicial}\n\nPregunta original: {prompt_usuario}"}
        ],
        temperature=0
    ).choices[0].message.content

    # Paso 3: Respuesta mejorada incorporando la crítica
    respuesta_mejorada = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "Eres un experto en nutrición deportiva. Mejora tu respuesta anterior basándote en la crítica recibida."},
            {"role": "user", "content": prompt_usuario},
            {"role": "assistant", "content": respuesta_inicial},
            {"role": "user", "content": f"Crítica de tu respuesta:\n{critica}\n\nPor favor, mejora tu respuesta incorporando estos puntos."}
        ],
        temperature=0.5
    ).choices[0].message.content

    return {
        "respuesta_inicial": respuesta_inicial,
        "critica": critica,
        "respuesta_mejorada": respuesta_mejorada
    }

resultado = generar_con_reflexion("¿Qué debo comer antes de un maratón?")
print(resultado["respuesta_mejorada"])
```

#### Chain-of-Thought (CoT) estructurado

```python
def cot_evaluacion(plan_entrenamiento: str) -> dict:
    """Evaluación estructurada con Chain-of-Thought explícito."""
    response = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": """Evalúa planes de entrenamiento siguiendo estos pasos EXACTOS:

PASO 1 - ANÁLISIS DE VOLUMEN: Calcula el volumen total (series × repeticiones).
PASO 2 - ANÁLISIS DE INTENSIDAD: Evalúa si la intensidad es apropiada para el nivel.
PASO 3 - ANÁLISIS DE RECUPERACIÓN: Verifica descanso entre sesiones y ejercicios.
PASO 4 - ANÁLISIS DE PROGRESIÓN: Hay un plan de progresión claro.
PASO 5 - CONCLUSIÓN Y PUNTUACIÓN: Da una puntuación del 1-10 y recomendaciones.

Responde en JSON con las claves: volumen, intensidad, recuperacion, progresion, puntuacion, recomendaciones."""},
            {"role": "user", "content": f"Plan a evaluar:\n{plan_entrenamiento}"}
        ],
        response_format={"type": "json_object"},
        temperature=0
    )
    return json.loads(response.choices[0].message.content)
```

---

### Observabilidad: trazabilidad, tokens y latencia

#### Tracing con Azure AI Foundry

```python
# Habilitar trazabilidad automática de Azure AI
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

client = AIProjectClient.from_connection_string(
    conn_str="<connection-string>",
    credential=DefaultAzureCredential()
)

# Activar tracing → envía trazas a Application Insights del proyecto
client.telemetry.enable()

# A partir de aquí, TODAS las llamadas LLM se trazan automáticamente
# Captura: inputs, outputs, token counts, latencia, modelo, deployment
```

```python
# Instrumentación manual con OpenTelemetry
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from azure.monitor.opentelemetry import configure_azure_monitor

# Configurar Application Insights
configure_azure_monitor(
    connection_string="InstrumentationKey=<key>;IngestionEndpoint=..."
)

tracer = trace.get_tracer(__name__)

def llamada_llm_trazada(prompt: str, contexto: str) -> str:
    with tracer.start_as_current_span("rag_generation") as span:
        # Registrar atributos personalizados
        span.set_attribute("prompt.length", len(prompt))
        span.set_attribute("context.length", len(contexto))
        span.set_attribute("model", "gpt-4o")

        inicio = time.time()

        response = openai_client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "Responde usando el contexto proporcionado."},
                {"role": "user", "content": f"Contexto: {contexto}\n\nPregunta: {prompt}"}
            ]
        )

        latencia = time.time() - inicio

        # Registrar métricas de tokens
        usage = response.usage
        span.set_attribute("tokens.prompt", usage.prompt_tokens)
        span.set_attribute("tokens.completion", usage.completion_tokens)
        span.set_attribute("tokens.total", usage.total_tokens)
        span.set_attribute("latency.seconds", latencia)

        return response.choices[0].message.content
```

#### Análisis de tokens y costos

```python
import tiktoken

def analizar_tokens(texto: str, modelo: str = "gpt-4o") -> dict:
    """Calcular tokens antes de enviar al API (ahorra costos)."""
    encoding = tiktoken.encoding_for_model("gpt-4o")
    tokens = encoding.encode(texto)

    # Precios aproximados GPT-4o (USD por 1M tokens)
    precio_input  = 5.00 / 1_000_000
    precio_output = 15.00 / 1_000_000

    return {
        "num_tokens": len(tokens),
        "costo_estimado_input": len(tokens) * precio_input,
        "dentro_limite_4k": len(tokens) < 4096,
        "dentro_limite_128k": len(tokens) < 128000
    }

# Monitoreo de costos por request
def llamada_con_monitoreo(messages: list) -> dict:
    response = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=messages
    )

    usage = response.usage
    costo = (usage.prompt_tokens * 5 + usage.completion_tokens * 15) / 1_000_000

    print(f"Tokens entrada:  {usage.prompt_tokens}")
    print(f"Tokens salida:   {usage.completion_tokens}")
    print(f"Total tokens:    {usage.total_tokens}")
    print(f"Costo estimado:  ${costo:.6f} USD")

    return {
        "content": response.choices[0].message.content,
        "usage": usage.model_dump(),
        "cost_usd": costo
    }
```

#### Content Safety — señales de seguridad

```python
from azure.ai.contentsafety import ContentSafetyClient
from azure.ai.contentsafety.models import AnalyzeTextOptions, TextCategory
from azure.core.credentials import AzureKeyCredential

safety_client = ContentSafetyClient(
    endpoint="https://<resource>.cognitiveservices.azure.com/",
    credential=AzureKeyCredential("<key>")
)

def verificar_seguridad(texto: str, umbral: int = 2) -> dict:
    """
    Verificar contenido antes de procesar o mostrar al usuario.
    Categorías: Hate, SelfHarm, Sexual, Violence
    Severidad: 0 (safe) → 6 (muy dañino)
    """
    request = AnalyzeTextOptions(
        text=texto,
        categories=[
            TextCategory.HATE,
            TextCategory.SELF_HARM,
            TextCategory.SEXUAL,
            TextCategory.VIOLENCE
        ],
        output_type="FourSeverityLevels"
    )

    response = safety_client.analyze_text(request)

    resultado = {"texto": texto, "seguro": True, "categorias": {}}
    for categoria in response.categories_analysis:
        resultado["categorias"][categoria.category] = categoria.severity
        if categoria.severity > umbral:
            resultado["seguro"] = False

    return resultado

# Uso: guardrail de entrada y salida
def pipeline_seguro(user_input: str) -> str:
    # 1. Verificar input del usuario
    check_input = verificar_seguridad(user_input)
    if not check_input["seguro"]:
        return "Lo siento, no puedo procesar esa solicitud."

    # 2. Generar respuesta
    respuesta = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": user_input}]
    ).choices[0].message.content

    # 3. Verificar output del modelo
    check_output = verificar_seguridad(respuesta)
    if not check_output["seguro"]:
        return "No pude generar una respuesta apropiada para esa consulta."

    return respuesta
```

---

### Orquestar múltiples modelos y motores híbridos

```python
class OrquestadorHibrido:
    """
    Motor híbrido: selecciona el modelo óptimo según la tarea.
    Combina LLM, reglas y modelos especializados.
    """

    def __init__(self):
        self.openai_client = AzureOpenAI(
            azure_endpoint="https://<resource>.openai.azure.com/",
            api_key="<key>",
            api_version="2024-02-01"
        )

    def clasificar_intencion(self, query: str) -> str:
        """Reglas simples antes de llamar al LLM (ahorra costos)."""
        query_lower = query.lower()
        if any(kw in query_lower for kw in ["horario", "precio", "reserva", "socio"]):
            return "administracion"
        elif any(kw in query_lower for kw in ["ejercicio", "entrenamiento", "rutina", "serie"]):
            return "entrenamiento"
        elif any(kw in query_lower for kw in ["dieta", "proteína", "calorías", "nutrición"]):
            return "nutricion"
        else:
            return "general"

    def seleccionar_modelo(self, intencion: str) -> tuple[str, str]:
        """Selecciona modelo y configuración según la intención."""
        configuraciones = {
            "administracion": ("gpt-35-turbo", "Eres el asistente administrativo del club."),
            "entrenamiento":  ("gpt-4o", "Eres un entrenador personal certificado."),
            "nutricion":      ("gpt-4o", "Eres un nutricionista deportivo certificado."),
            "general":        ("gpt-35-turbo", "Eres el asistente general del club deportivo.")
        }
        return configuraciones.get(intencion, configuraciones["general"])

    def procesar(self, query: str) -> dict:
        # 1. Clasificación por reglas (sin LLM)
        intencion = self.clasificar_intencion(query)

        # 2. Seleccionar modelo apropiado
        modelo, system_prompt = self.seleccionar_modelo(intencion)

        # 3. Generar respuesta con el modelo óptimo
        response = self.openai_client.chat.completions.create(
            model=modelo,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": query}
            ],
            temperature=0.3
        )

        return {
            "intencion": intencion,
            "modelo_usado": modelo,
            "respuesta": response.choices[0].message.content,
            "tokens": response.usage.total_tokens
        }

# Uso
orquestador = OrquestadorHibrido()
resultado = orquestador.procesar("¿Cuántas proteínas debo tomar después del entreno?")
print(f"[{resultado['modelo_usado']}] {resultado['respuesta']}")
```

#### Pipeline con múltiples modelos secuenciales

```python
def pipeline_multimodelo(documento_imagen: bytes, pregunta: str) -> str:
    """
    Pipeline: Visión (GPT-4o) → Razonamiento (GPT-4o) → Síntesis (GPT-3.5)
    Optimiza costos usando modelos más baratos donde es posible.
    """
    import base64

    # Paso 1: Extraer contenido de imagen con modelo multimodal
    imagen_b64 = base64.b64encode(documento_imagen).decode()

    extraccion = openai_client.chat.completions.create(
        model="gpt-4o",  # multimodal
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": "Extrae todo el texto e información de esta imagen. Sé exhaustivo."},
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{imagen_b64}"}}
            ]
        }],
        max_tokens=1000
    ).choices[0].message.content

    # Paso 2: Razonamiento sobre el contenido extraído
    razonamiento = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "Analiza el contenido y responde la pregunta con razonamiento detallado."},
            {"role": "user", "content": f"Contenido del documento:\n{extraccion}\n\nPregunta: {pregunta}"}
        ],
        temperature=0
    ).choices[0].message.content

    # Paso 3: Síntesis concisa con modelo más económico
    sintesis = openai_client.chat.completions.create(
        model="gpt-35-turbo",  # más barato para síntesis simple
        messages=[
            {"role": "system", "content": "Resume la respuesta de forma concisa en máximo 3 oraciones."},
            {"role": "user", "content": razonamiento}
        ],
        max_tokens=200
    ).choices[0].message.content

    return sintesis
```

### Puntos clave para el examen — Sección 2.3

- ✅ `temperature=0` → determinístico; `temperature=1.0` → creativo. No combinar con `top_p` alto simultáneamente.
- ✅ `response_format={"type": "json_object"}` garantiza JSON válido en la respuesta.
- ✅ **Reflexion pattern**: generar → criticar → mejorar (3 llamadas al LLM).
- ✅ **Chain-of-Thought**: incluir razonamiento paso a paso en el prompt mejora la precisión.
- ✅ `client.telemetry.enable()` activa el tracing automático en Foundry.
- ✅ Content Safety: 4 categorías (Hate, SelfHarm, Sexual, Violence); severidad 0-6.
- ✅ Híbrido LLM+reglas: usar reglas simples para clasificar antes de llamar al LLM (ahorra costos).
- ✅ `tiktoken` permite calcular tokens antes de llamar al API.
- ✅ Application Insights + OpenTelemetry para métricas personalizadas de agentes.

---

## Resumen visual — Arquitectura completa

```
┌─────────────────────────────────────────────────────────────────┐
│                    AZURE AI FOUNDRY                              │
│                                                                  │
│  ┌─────────────────┐    ┌──────────────────┐                   │
│  │   Modelos        │    │  Prompt Flow      │                   │
│  │  - GPT-4o        │    │  - Standard flow  │                   │
│  │  - Phi-3         │    │  - Chat flow      │                   │
│  │  - Embeddings    │    │  - Eval flow      │                   │
│  └────────┬─────────┘    └────────┬─────────┘                   │
│           │                       │                              │
│  ┌────────▼───────────────────────▼──────────┐                  │
│  │              AI PROJECT CLIENT             │                  │
│  │      (AIProjectClient.from_connection_string)                 │
│  └────────┬───────────────────────────────────┘                  │
│           │                                                      │
│  ┌────────▼──────────┐  ┌─────────────┐  ┌──────────────┐      │
│  │    AGENTES         │  │  EVALUACIÓN  │  │ OBSERVAB.    │      │
│  │  - Agent           │  │  - Ground.   │  │ - Tracing    │      │
│  │  - Thread          │  │  - Relevance │  │ - App Insights│      │
│  │  - Run             │  │  - Coherence │  │ - Safety     │      │
│  │  - Tools:          │  │  - Safety    │  │ - Tokens     │      │
│  │    code_interp     │  └─────────────┘  └──────────────┘      │
│  │    file_search     │                                          │
│  │    bing_grounding  │  ┌─────────────────────────────────┐    │
│  │    functions       │  │         RAG PIPELINE             │    │
│  │    ai_search       │  │  Docs → Embed → Index → Retrieve │    │
│  └───────────────────┘  │  → Augment Prompt → Generate     │    │
│                          └─────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Checklist de repaso — Sección 2 completa

### Antes del examen, asegúrate de poder responder:

**Sobre RAG:**
- [ ] ¿Cuáles son los 4 pasos del pipeline RAG?
- [ ] ¿Qué diferencia hay entre búsqueda vectorial e híbrida?
- [ ] ¿Por qué usar `temperature=0` en RAG?
- [ ] ¿Qué es groundedness y cómo se mide?

**Sobre Agentes:**
- [ ] ¿Qué diferencia hay entre Agent, Thread y Run?
- [ ] ¿Cuándo se produce el status `REQUIRES_ACTION`?
- [ ] ¿Qué herramienta ejecuta código Python en sandbox?
- [ ] ¿Cómo se implementa Human-in-the-Loop en Azure AI Agent Service?
- [ ] ¿Qué es Semantic Kernel y cómo orquesta agentes?

**Sobre optimización:**
- [ ] ¿Para qué sirve `frequency_penalty`?
- [ ] ¿Cómo forzar respuestas en JSON?
- [ ] ¿Qué es el patrón Reflexion?
- [ ] ¿Cómo activar el tracing en Foundry con una sola línea?
- [ ] ¿Qué categorías evalúa Azure Content Safety?

---

## Recursos

| Recurso | URL |
|---------|-----|
| Azure AI Foundry Docs | https://learn.microsoft.com/azure/ai-studio/ |
| Azure AI Agent Service | https://learn.microsoft.com/azure/ai-services/agents/ |
| Azure AI Evaluation SDK | https://learn.microsoft.com/azure/ai-studio/how-to/evaluate-sdk |
| Prompt Flow | https://microsoft.github.io/promptflow/ |
| Semantic Kernel Python | https://learn.microsoft.com/semantic-kernel/overview/ |
| Azure AI Search Vector | https://learn.microsoft.com/azure/search/vector-search-overview |
| Content Safety API | https://learn.microsoft.com/azure/ai-services/content-safety/ |
| AI-103 Exam Skills | https://learn.microsoft.com/credentials/certifications/exams/ai-103/ |

---

*Sección 2 de 4 — Guía de estudio AI-103 en español*
*Peso en examen: 30-35% — Prioridad: MÁXIMA*
