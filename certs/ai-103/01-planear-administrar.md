# Sección 1: Planear y administrar una solución de IA de Azure (25–30%)

> ⚠️ **Sección de alto peso.** Representa entre el 25–30% del examen AI-103. Cubre la arquitectura, configuración, seguridad, monitoreo y principios de IA responsable de toda solución construida con **Azure AI Foundry**.

---

## Estructura de Azure AI Foundry (base conceptual)

Antes de entrar a cada subsección, es fundamental entender la jerarquía de Foundry:

```
Azure Subscription
└── Azure AI Foundry Hub          ← Infraestructura compartida
    ├── Conexiones (OpenAI, Search, Storage...)
    ├── Compute & Networking
    └── Azure AI Foundry Project  ← Espacio de trabajo aislado por equipo/solución
        ├── Catálogo de modelos
        ├── Deployments de modelos
        ├── Agentes
        ├── Índices de búsqueda
        └── Evaluaciones / Prompt Flow
```

| Nivel       | Responsabilidad                                              | Compartido entre proyectos |
|-------------|--------------------------------------------------------------|---------------------------|
| **Hub**     | Redes, cómputo, almacenamiento, conexiones, seguridad base   | ✅ Sí                     |
| **Project** | Modelos desplegados, agentes, índices, experimentos, código  | ❌ No (aislado)           |

> 📌 **Regla de examen:** El Hub es el recurso padre; un proyecto hereda las conexiones y la configuración de red del Hub, pero sus modelos y agentes son propios.

---

## 1.1 Elegir los servicios de Foundry adecuados para IA generativa y agentes

### Conceptos clave

#### Taxonomía de modelos disponibles en Foundry

| Categoría             | Modelos principales                           | Cuándo usarlos                                              |
|-----------------------|-----------------------------------------------|-------------------------------------------------------------|
| **LLM general**       | GPT-4o, GPT-4o-mini                           | Razonamiento complejo, chat, síntesis, análisis de documentos |
| **SLM (Small LM)**    | Phi-4, Phi-3-medium, Phi-3-mini               | Escenarios de bajo costo, edge/on-device, tasks simples      |
| **Multimodal**        | GPT-4o (vision), DALL-E 3                     | Entrada imagen+texto, generación de imágenes                 |
| **Código**            | GPT-4o, Phi-3-medium                          | Generación, explicación y revisión de código                 |
| **Embeddings**        | text-embedding-3-large, text-embedding-3-small| Búsqueda semántica, RAG, clustering                          |
| **Audio/Speech**      | Whisper (STT), Azure TTS                      | Transcripción de voz, text-to-speech                         |
| **Modelos open**      | Llama 3.x, Mistral, Cohere Command R+         | Alternativas open-source/lower cost vía serverless           |

#### Árbol de decisión: selección de modelo

```
¿La tarea requiere razonamiento complejo o contexto muy largo?
├─ Sí → GPT-4o
└─ No
   ├─ ¿Es sensible al costo o simple (extracción, clasificación)?
   │   ├─ Sí → GPT-4o-mini  o  Phi-4
   │   └─ No → continúa...
   ├─ ¿Generación o comprensión de código?
   │   └─ GPT-4o  o  Phi-3-medium
   ├─ ¿Entrada visual (imágenes + texto)?
   │   └─ GPT-4o (vision)
   ├─ ¿Generación de imágenes?
   │   └─ DALL-E 3
   ├─ ¿Transcripción de audio?
   │   └─ Whisper
   ├─ ¿Búsqueda semántica / RAG?
   │   └─ text-embedding-3-large (mayor precisión) o text-embedding-3-small (menor costo)
   └─ ¿Despliegue en edge / sin internet?
       └─ Phi-3-mini (SLM optimizado para on-device)
```

#### Servicios clave de Azure AI Foundry y su función

| Servicio                       | Función principal                                                        | Tipo de tarea                        |
|--------------------------------|--------------------------------------------------------------------------|--------------------------------------|
| **Azure AI Agent Service**     | Construir y ejecutar agentes con herramientas, memoria y flujos multi-step | Agentes autónomos, orquestación      |
| **Azure AI Search**            | Indexación vectorial, búsqueda semántica e híbrida (BM25 + vector)       | RAG, recuperación de información     |
| **Azure AI Content Safety**    | Filtrado de contenido dañino en inputs y outputs del modelo              | Moderación, guardrails               |
| **Azure AI Evaluation**        | Métricas automatizadas de calidad del modelo (coherencia, relevancia…)   | QA de respuestas generativas         |
| **Prompt Flow**                | Orquestación visual de cadenas LLM (DAG de pasos)                       | Pipelines de IA sin código intensivo |
| **Fine-tuning**                | Ajuste fino de modelos en datos propios del dominio                      | Personalización de modelos           |
| **Azure OpenAI Service**       | API para modelos GPT, DALL-E, Whisper, Embeddings de OpenAI              | Inferencia de modelos                |

#### Selección de servicio por tipo de tarea

| Tarea                                          | Servicio recomendado                                   |
|------------------------------------------------|--------------------------------------------------------|
| Generación de texto / chat                     | Azure OpenAI (GPT-4o via Foundry)                      |
| Alineación de datos / fine-tuning              | Fine-tuning en Foundry (supervisado o preferencias)    |
| Búsqueda semántica / RAG                       | Azure AI Search (vector + híbrido) + embeddings        |
| Flujos de agentes multi-step                   | Azure AI Agent Service + Prompt Flow (opcional)        |
| Procesamiento multimodal                       | GPT-4o vision / DALL-E 3 / Whisper                     |
| Moderación de contenido                        | Azure AI Content Safety                                |
| Evaluación de calidad                          | Azure AI Evaluation                                    |

#### Recuperación e indexación: métodos disponibles

| Método                    | Descripción                                                        | Cuándo elegirlo                                    |
|---------------------------|--------------------------------------------------------------------|----------------------------------------------------|
| **Búsqueda vectorial pura** | Usa embeddings + distancia coseno/euclidiana                      | Alta semántica, queries en lenguaje natural         |
| **BM25 (keyword)**        | Búsqueda léxica clásica basada en frecuencia de términos           | Términos técnicos exactos, IDs, nombres propios     |
| **Búsqueda híbrida**      | Combina vectorial + BM25 con Reciprocal Rank Fusion (RRF)         | ✅ Recomendado por defecto para RAG                |
| **Semantic Reranker**     | Reordena resultados con un modelo de lenguaje secundario           | Cuando la precisión de ranking es crítica           |
| **Integrated vectorization** | Foundry vectoriza automáticamente al ingestar documentos       | Simplificar el pipeline ETL de documentos           |

#### Memoria, herramientas y conocimiento para agentes

| Componente             | Descripción                                                              | Ejemplo en Foundry                          |
|------------------------|--------------------------------------------------------------------------|---------------------------------------------|
| **Memoria de corto plazo** | Historial de conversación en el contexto del modelo                  | Thread de conversación en Agent Service     |
| **Memoria de largo plazo** | Datos persistidos y recuperados vía búsqueda vectorial               | Azure AI Search como knowledge store        |
| **Herramientas (Tools)**   | Funciones externas que el agente puede invocar                       | Code Interpreter, Function Calling, Bing    |
| **Knowledge grounding**    | Documentos base para fundamentar respuestas (RAG)                    | Índice de Azure AI Search vinculado         |
| **Integración de datos**   | Conectores a fuentes externas (SharePoint, SQL, APIs)                | Azure AI Search data sources / connectors   |

### Aplicación práctica

**Escenario RAG (Retrieval-Augmented Generation):**
1. Ingestar documentos → Azure AI Search con integrated vectorization (usa text-embedding-3-large).
2. Configurar búsqueda híbrida + semantic reranker.
3. En cada consulta: recuperar top-K chunks → concatenar al prompt → GPT-4o genera respuesta fundamentada.

**Escenario de agente multi-herramienta:**
1. Crear agente en Azure AI Agent Service.
2. Definir herramientas: `code_interpreter`, `file_search`, funciones personalizadas (Function Calling).
3. Conectar memory store (AI Search) para persistencia.
4. El agente decide qué herramienta usar en cada turno.

### Puntos clave para el examen

- ✅ **Phi-4 / Phi-3** = SLM → bajo costo, tareas simples, edge deployment.
- ✅ **text-embedding-3-large** = mayor precisión para RAG de alta calidad.
- ✅ **Búsqueda híbrida** es el patrón RAG recomendado en Azure.
- ✅ El **Semantic Reranker** mejora el ranking pero añade latencia y costo.
- ✅ **Whisper** = STT (Speech-to-Text); **Azure TTS** = Text-to-Speech — son servicios distintos.
- ✅ **DALL-E 3** = generación de imágenes; **GPT-4o vision** = comprensión de imágenes.

---

## 1.2 Configurar soluciones de IA en Foundry

### Conceptos clave

#### Diseño de infraestructura Azure para soluciones de IA

```
┌─────────────────────────────────────────────────┐
│              Azure Subscription                 │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │         Azure AI Foundry Hub             │   │
│  │  • Azure Storage (artefactos, datasets)  │   │
│  │  • Azure Key Vault (secretos)            │   │
│  │  • Azure Container Registry (imágenes)   │   │
│  │  • Log Analytics Workspace               │   │
│  │  • VNet / Private Endpoints              │   │
│  │                                          │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │       AI Foundry Project           │  │   │
│  │  │  • Connections (OpenAI, Search...) │  │   │
│  │  │  • Model deployments               │  │   │
│  │  │  • Agents & Threads                │  │   │
│  │  │  • Indexes (AI Search)             │  │   │
│  │  │  • Evaluations                     │  │   │
│  │  └────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  Azure OpenAI Resource  │  Azure AI Search      │
│  (gestionado por Hub)   │  (vinculado al Hub)   │
└─────────────────────────────────────────────────┘
```

**Recursos que se crean automáticamente con el Hub:**
- Azure Storage Account
- Azure Key Vault
- Azure Container Registry (opcional)
- Application Insights (opcional pero recomendado)

#### Opciones de deployment de modelos

| Tipo de Deployment        | Descripción                                                         | Caso de uso                                    | Costo           |
|---------------------------|---------------------------------------------------------------------|------------------------------------------------|-----------------|
| **Standard**              | Capacidad compartida, pay-per-token                                 | Desarrollo, cargas variables                   | 💲 Bajo (por uso)|
| **Global Standard**       | Enrutamiento global a regiones con capacidad disponible             | Cargas globales, mayor disponibilidad           | 💲💲 Medio       |
| **Provisioned (PTU)**     | Throughput garantizado, capacidad reservada (Provisioned Throughput Units) | Producción, latencia predecible, alto volumen | 💲💲💲 Alto (fijo) |
| **Serverless (MaaS)**     | Modelos de terceros (Llama, Mistral) sin gestionar infraestructura  | Open-source models, pay-per-token              | 💲 Variable      |

> 📌 **Clave de examen:** PTU (Provisioned) = latencia predecible + capacidad garantizada, pero compromiso de costo fijo. Standard = flexible pero sujeto a throttling en picos.

#### Matriz de selección de deployment

| Criterio                          | Standard | Global Standard | Provisioned (PTU) |
|-----------------------------------|----------|-----------------|-------------------|
| Latencia predecible               | ❌        | ⚠️ Parcial      | ✅                 |
| Sin compromisos de costo          | ✅        | ✅               | ❌                 |
| Alta disponibilidad global        | ❌        | ✅               | ⚠️ Por región      |
| Adecuado para producción crítica  | ⚠️        | ✅               | ✅                 |
| Modelos open-source               | ❌        | ❌               | ❌ (usar Serverless)|

#### Configuración de deployments de modelos y agentes

**Deployment de modelo (via portal o bicep/ARM):**
```json
{
  "name": "gpt-4o-prod",
  "model": {
    "format": "OpenAI",
    "name": "gpt-4o",
    "version": "2024-11-20"
  },
  "sku": {
    "name": "Standard",
    "capacity": 100
  }
}
```

**Parámetros clave al configurar un deployment:**
- `capacity`: TPM (Tokens Per Minute) en Standard; PTUs en Provisioned.
- `version`: versión específica del modelo (importante para reproducibilidad).
- `content_filter_policy`: filtro de contenido asociado.

**Configuración de agente (Azure AI Agent Service):**
```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

client = AIProjectClient.from_connection_string(
    conn_str="<connection_string>",
    credential=DefaultAzureCredential()
)

agent = client.agents.create_agent(
    model="gpt-4o-prod",           # nombre del deployment
    name="mi-agente",
    instructions="Eres un asistente experto en ...",
    tools=[{"type": "code_interpreter"}, {"type": "file_search"}]
)
```

#### Integración con pipelines CI/CD

| Componente CI/CD              | Herramienta Azure                              | Descripción                                              |
|-------------------------------|------------------------------------------------|----------------------------------------------------------|
| **Control de versiones**      | Azure DevOps Repos / GitHub                    | Código de Prompt Flow, definiciones de agentes, configs  |
| **Pipelines de evaluación**   | Azure DevOps Pipelines / GitHub Actions        | Ejecutar evaluaciones automáticas antes de promover       |
| **Despliegue de modelos**     | Azure CLI / Bicep / Terraform                  | IaC para deployments en Foundry                          |
| **Registro de artefactos**    | Azure AI Foundry Model Registry                | Versionar modelos fine-tuned y flujos                    |
| **Aprobación de cambios**     | Azure DevOps Gates / GitHub Environments       | Aprobación humana antes de producción                    |

**Flujo CI/CD recomendado para soluciones AI:**
```
Commit código → CI: lint + unit tests
             → Ejecutar evaluaciones AI (coherencia, relevancia, groundedness)
             → Umbral de calidad: ¿pasa el score mínimo?
                ├─ Sí → CD: desplegar a staging → aprobación humana → producción
                └─ No → Bloquear despliegue + notificar
```

**Ejemplo GitHub Actions para evaluación AI:**
```yaml
- name: Run AI Evaluation
  run: |
    az ml job create --file eval_job.yaml \
      --resource-group $RG \
      --workspace-name $HUB_NAME
```

### Aplicación práctica

- Usar **Bicep / ARM templates** para provisionar Hub + Project de forma reproducible.
- Definir **connections** en el Hub (Azure OpenAI, AI Search, Storage) para que todos los proyectos las hereden.
- Usar **Prompt Flow SDK** o REST API en pipelines para evaluaciones automatizadas antes de promover cambios.
- Mantener **versiones de deployment**: nunca sobreescribir un deployment en producción; crear uno nuevo y redirigir tráfico.

### Puntos clave para el examen

- ✅ **PTU** = Provisioned Throughput Units = capacidad reservada = costo fijo + baja latencia predecible.
- ✅ **Global Standard** = enrutamiento global automático, mayor disponibilidad que Standard.
- ✅ Las **connections** se definen en el Hub y son accesibles por todos los proyectos hijo.
- ✅ El agente usa el **nombre del deployment** (no el nombre del modelo base) al configurarse.
- ✅ CI/CD debe incluir **evaluaciones automáticas** de calidad como gate de calidad, no solo tests de código.
- ✅ `DefaultAzureCredential` permite autenticación sin API keys usando Managed Identity o credenciales del entorno.

---

## 1.3 Administrar, supervisar y proteger sistemas de IA

### Conceptos clave

#### Gestión de cuotas y rate limits

| Concepto             | Descripción                                                                   |
|----------------------|-------------------------------------------------------------------------------|
| **TPM**              | Tokens Per Minute — límite de velocidad de procesamiento por deployment        |
| **RPM**              | Requests Per Minute — límite de solicitudes por deployment                     |
| **Cuota regional**   | Límite total de TPM disponibles en una región para una suscripción             |
| **PTU**              | Provisioned Throughput Units — capacidad fija comprada, sin límites dinámicos  |

**Estrategias para gestionar cuotas:**
- Distribuir deployments en **múltiples regiones** para aprovechar cuotas regionales independientes.
- Implementar **retry con exponential backoff** para errores 429 (rate limit exceeded).
- Usar **Global Standard** para enrutamiento automático a regiones con capacidad.
- Monitorear uso de tokens con **Azure Monitor** para anticipar límites.

#### Estructura de costos — comparativa

| Componente                  | Modelo de costo                             | Herramienta de control             |
|-----------------------------|---------------------------------------------|------------------------------------|
| Modelos Standard/Serverless | Por token (input + output, precio diferente)| Azure Cost Management              |
| Modelos Provisioned (PTU)   | Costo fijo por hora/mes                     | Planificación de capacidad          |
| Azure AI Search             | Por unidad de búsqueda (SU) + documentos    | Scaling de replicas/particiones     |
| Storage (embeddings, datos) | Por GB almacenado                           | Políticas de retención              |
| Compute (fine-tuning)       | Por hora de GPU                             | Limitar trabajos de entrenamiento   |

> 💡 **Optimización de costos:** GPT-4o-mini puede reducir costos hasta un 90% vs GPT-4o en tasks simples. Evaluar si la calidad es aceptable antes de usar el modelo más potente.

#### Supervisión del sistema

**Stack de monitoreo:**
```
Azure AI Foundry
    └── Diagnósticos → Log Analytics Workspace
                           └── Application Insights
                                   └── Azure Monitor Workbooks
                                           └── Alertas (Azure Monitor Alerts)
```

**Métricas y logs críticos a monitorear:**

| Qué monitorear                    | Métrica / Log                                      | Herramienta                    |
|-----------------------------------|----------------------------------------------------|--------------------------------|
| **Uso de tokens**                 | `TokensConsumed` por deployment                    | Azure Monitor Metrics          |
| **Latencia de inferencia**        | `DurationMs` por request                           | Application Insights           |
| **Errores 429 (throttling)**      | `HttpStatusCode = 429`                             | Log Analytics (KQL)            |
| **Calidad de respuestas**         | Scores de evaluación (coherencia, relevancia)      | Azure AI Evaluation             |
| **Drift del modelo**              | Degradación de métricas de calidad en el tiempo    | Evaluaciones periódicas         |
| **Contenido bloqueado**           | Eventos de filtro de contenido (blocked_by_filter) | Azure AI Content Safety logs   |
| **Groundedness / alucinaciones**  | `groundedness_score` en evaluaciones               | Azure AI Evaluation             |
| **Estado índice de búsqueda**     | `DocumentCount`, errores de indexación             | Azure AI Search métricas        |
| **Calidad de relevancia**         | NDCG, MRR en búsquedas de prueba                  | Azure AI Evaluation (retrieval) |
| **Ingesta de datos**              | Documentos indexados, fallos de parsing            | AI Search indexer status        |

**Consulta KQL de ejemplo — detectar throttling:**
```kql
AzureDiagnostics
| where ResourceType == "MICROSOFT.COGNITIVESERVICES/ACCOUNTS"
| where ResultSignature == "429"
| summarize Count = count() by bin(TimeGenerated, 5m), Resource
| render timechart
```

**Consulta KQL — tokens consumidos por deployment:**
```kql
AzureMetrics
| where MetricName == "TokensConsumed"
| summarize TotalTokens = sum(Total) by bin(TimeGenerated, 1h), Resource
| render timechart
```

#### Drift y degradación de calidad

El **drift** ocurre cuando la calidad de las respuestas del modelo se degrada con el tiempo, causado por:
- Cambios en los datos de entrada (distribución drift).
- Actualizaciones del modelo base por el proveedor.
- Cambios en el dominio de negocio.

**Estrategia de detección:**
1. Definir **dataset de evaluación** de referencia (golden dataset).
2. Ejecutar evaluaciones periódicas (semanales/mensuales) con el mismo dataset.
3. Alertar cuando los scores caen por debajo de umbrales definidos.
4. Registrar histórico de scores en Log Analytics para tendencias.

#### Configuración de seguridad

##### Managed Identity (sin API keys)

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

# Sin API key — usa Managed Identity o credenciales del ambiente
credential = DefaultAzureCredential()
client = AIProjectClient.from_connection_string(
    conn_str="<connection_string>",
    credential=credential
)
```

**`DefaultAzureCredential` intenta en orden:**
1. Variables de entorno (`AZURE_CLIENT_ID`, etc.)
2. Managed Identity (en Azure: VMs, App Service, Functions, AKS…)
3. Azure CLI credentials (local dev)
4. Visual Studio / VS Code credentials

##### Redes privadas y endpoints

| Configuración                    | Descripción                                                        |
|----------------------------------|--------------------------------------------------------------------|
| **Private Endpoint**             | Conecta el Hub/Project a la VNet, sin tráfico público              |
| **VNet Integration**             | Integra App Service / Functions con la VNet para llamadas internas |
| **Service Endpoint**             | Acceso optimizado sin IP pública desde subnets específicas         |
| **Network isolation (Hub)**      | Modo "Disabled" / "Enabled" / "AllowedByPolicy" de acceso público |

> 📌 Para producción: Hub con `publicNetworkAccess: Disabled` + Private Endpoints + DNS privado.

##### Azure Key Vault

- Almacenar: connection strings, API keys de terceros, certificados.
- El Hub/Project acceden al Key Vault vía **Managed Identity** (sin guardar secretos en código).
- Configurar **soft-delete** y **purge protection** para prevenir borrado accidental.

##### RBAC — Roles de Azure AI

| Rol                                          | Permisos principales                                                |
|----------------------------------------------|---------------------------------------------------------------------|
| **AI Developer**                             | Crear proyectos, deployments, agentes, ejecutar evaluaciones        |
| **AI Inference Deployment Operator**         | Gestionar deployments de modelos (sin crear proyectos)              |
| **Cognitive Services User**                  | Invocar endpoints de modelos (solo inferencia)                      |
| **Cognitive Services OpenAI Contributor**    | Gestionar recursos OpenAI, fine-tuning                              |
| **Search Index Data Contributor**            | Escribir en índices de Azure AI Search                              |
| **Search Index Data Reader**                 | Leer/consultar índices de Azure AI Search                           |

**Principio de mínimo privilegio:**
- App de producción → `Cognitive Services User` (solo inferencia).
- Pipeline CI/CD de deployments → `AI Inference Deployment Operator`.
- Desarrolladores → `AI Developer` (en entornos de dev/staging).

### Aplicación práctica

**Arquitectura segura de referencia:**
```
Internet → Azure Front Door (WAF)
              → App Service (VNet Integrated)
                   ↓ Private Endpoint
              Azure AI Foundry Hub (publicNetworkAccess: Disabled)
                   ↓ Managed Identity
              Azure OpenAI Resource
                   ↓ Managed Identity
              Azure AI Search (Private Endpoint)
                   ↓ Managed Identity
              Azure Key Vault (Private Endpoint)
```

### Puntos clave para el examen

- ✅ **Managed Identity** = sin API keys = credencial de Azure AD automática para recursos Azure.
- ✅ `DefaultAzureCredential` funciona tanto en local (via Azure CLI) como en Azure (via Managed Identity).
- ✅ **PTU** = Provisioned Throughput → costo fijo + sin throttling dinámico.
- ✅ Para detectar **drift**: evaluaciones periódicas con dataset de referencia + alertas en Monitor.
- ✅ **RBAC**: separar rol de "invocar modelo" (`Cognitive Services User`) de "gestionar deployment" (`AI Inference Deployment Operator`).
- ✅ **Private Endpoints** = tráfico no sale a internet → necesario para compliance en producción.
- ✅ **Log Analytics + KQL** = herramienta estándar para consultar logs de Azure AI.
- ✅ Los **indexer status** de AI Search muestran cuántos documentos se procesaron y los errores de ingesta.

---

## 1.4 Implementación de IA responsable en sistemas generativos y agentes

### Conceptos clave

La IA Responsable en Azure AI Foundry se implementa en múltiples capas:

```
┌─────────────────────────────────────────────────┐
│              IA Responsable (capas)             │
│                                                 │
│  1. Filtros de contenido (Content Filters)      │
│  2. Prompt Shield (detección de ataques)        │
│  3. Groundedness detection (alucinaciones)      │
│  4. Evaluaciones de calidad y seguridad         │
│  5. Auditoría y trazabilidad                    │
│  6. Controles de comportamiento del agente      │
└─────────────────────────────────────────────────┘
```

#### Filtros de contenido (Content Filters)

Azure AI Content Safety proporciona filtros configurables para 4 categorías de daño:

| Categoría        | Descripción                                              | Niveles de severidad     |
|------------------|----------------------------------------------------------|--------------------------|
| **Hate**         | Discurso de odio, discriminación                         | Safe / Low / Medium / High |
| **Violence**     | Contenido violento, instrucciones de daño físico         | Safe / Low / Medium / High |
| **Sexual**       | Contenido de naturaleza sexual                           | Safe / Low / Medium / High |
| **Self-harm**    | Contenido relacionado con autolesiones                   | Safe / Low / Medium / High |

**Configuración de política de filtrado:**
- Se aplica tanto a **inputs** (prompt del usuario) como a **outputs** (respuesta del modelo).
- Umbral configurable: por defecto en **Medium** para producción.
- Acción cuando se supera el umbral: `block` (devuelve error) o `annotate` (solo registra).

```json
{
  "contentFilterConfig": {
    "hate": { "filterEnabled": true, "severityThreshold": "medium" },
    "violence": { "filterEnabled": true, "severityThreshold": "medium" },
    "sexual": { "filterEnabled": true, "severityThreshold": "high" },
    "selfHarm": { "filterEnabled": true, "severityThreshold": "medium" }
  }
}
```

> 📌 Los filtros de contenido **no se pueden desactivar completamente** en la mayoría de deployments Standard — se requiere aprobación especial de Microsoft para ajustar umbrales a niveles muy bajos.

#### Prompt Shield — Detección de ataques de inyección

**Prompt injection**: intento del usuario o de datos externos de manipular el comportamiento del modelo, eludiendo las instrucciones del sistema.

| Tipo de ataque               | Descripción                                              | Ejemplo                                    |
|------------------------------|----------------------------------------------------------|--------------------------------------------|
| **Direct injection**         | El usuario introduce instrucciones maliciosas en el prompt | "Ignora las instrucciones anteriores y..."  |
| **Indirect injection**       | Instrucciones maliciosas en documentos/datos recuperados | Un documento RAG que contiene "...olvida tu rol..." |

**Prompt Shield** detecta ambos tipos y puede:
- Bloquear el request.
- Añadir metadatos de riesgo para decisión downstream.

#### Groundedness Detection — Detección de alucinaciones

| Concepto                  | Descripción                                                        |
|---------------------------|--------------------------------------------------------------------|
| **Alucinación**           | El modelo genera información falsa o no fundamentada en el contexto|
| **Groundedness**          | La respuesta está fundamentada en los documentos/contexto provistos|
| **Groundedness score**    | Métrica 0–5: qué tan bien la respuesta refleja las fuentes         |

**Cómo funciona:**
- Azure AI Evaluation compara la respuesta generada vs. los documentos de contexto.
- Detecta afirmaciones en la respuesta que NO están respaldadas por el contexto.
- Output: score + lista de frases "no fundamentadas" identificadas.

#### Evaluación de IA responsable — Métricas de calidad

| Métrica              | Qué mide                                                            | Rango  |
|----------------------|---------------------------------------------------------------------|--------|
| **Coherence**        | Consistencia lógica y fluidez de la respuesta                       | 1–5    |
| **Fluency**          | Calidad gramatical y lingüística                                    | 1–5    |
| **Relevance**        | Qué tan relevante es la respuesta a la pregunta original            | 1–5    |
| **Groundedness**     | Fundamento de la respuesta en el contexto/documentos provistos      | 1–5    |
| **Similarity (F1)**  | Similitud semántica con una respuesta de referencia (ground truth)  | 0–1    |
| **Violence / Hate**  | Detección de contenido dañino en respuestas                         | 0–7    |

**Evaluación via SDK:**
```python
from azure.ai.evaluation import evaluate, CoherenceEvaluator, GroundednessEvaluator

results = evaluate(
    data="eval_dataset.jsonl",
    evaluators={
        "coherence": CoherenceEvaluator(model_config=model_config),
        "groundedness": GroundednessEvaluator(model_config=model_config)
    },
    output_path="./eval_results"
)
```

**Formato del dataset de evaluación (JSONL):**
```jsonl
{"query": "¿Cuál es la política de devoluciones?", "response": "La política es...", "context": "Según el manual..."}
{"query": "¿Cómo contacto soporte?", "response": "Puede contactar...", "context": "El equipo de soporte..."}
```

#### Auditoría, trazabilidad y metadatos de procedencia

| Elemento de auditoría           | Cómo implementarlo en Azure                                         |
|---------------------------------|---------------------------------------------------------------------|
| **Logs de inputs/outputs**      | Habilitar diagnósticos en Azure OpenAI → Log Analytics              |
| **Metadatos de procedencia**    | Incluir fuentes en la respuesta del agente (citations)              |
| **Thread IDs de agentes**       | Cada conversación tiene un `thread_id` único → trazabilidad         |
| **Run IDs**                     | Cada ejecución del agente tiene un `run_id` → auditar pasos         |
| **Flujos de aprobación**        | Azure DevOps Gates / Logic Apps para aprobación humana en workflows |
| **Inmutabilidad de logs**       | Log Analytics tiene retención configurable y es immutable por diseño|

> ⚠️ **Privacidad:** Habilitar logging de inputs/outputs puede capturar **PII**. Considerar anonimización o filtrado antes de almacenar. Documentar en la política de privacidad.

**Trazabilidad de agentes:**
```python
# Recuperar pasos de ejecución de un agente para auditoría
run_steps = client.agents.list_run_steps(
    thread_id=thread.id,
    run_id=run.id
)
for step in run_steps:
    print(f"Step: {step.type} | Status: {step.status} | Tool: {step.step_details}")
```

#### Control de comportamiento del agente

| Control                          | Descripción                                                          |
|----------------------------------|----------------------------------------------------------------------|
| **Instrucciones del sistema**    | Definir rol, límites y comportamiento esperado del agente            |
| **Modos de supervisión**         | Human-in-the-loop: el agente solicita aprobación antes de actuar     |
| **Restricciones de herramientas**| Limitar qué tools puede usar el agente (ej: solo `file_search`)      |
| **Permisos de herramientas**     | Configurar scope de Code Interpreter (sin acceso a red externa)      |
| **Max iterations**               | Limitar el número de pasos del agente para evitar bucles infinitos   |
| **Timeout de runs**              | Cancelar runs que excedan tiempo límite                              |

**Ejemplo: agente con supervisión humana (HITL):**
```python
# Configurar requires_action → pausa hasta aprobación humana
run = client.agents.create_and_process_run(
    thread_id=thread.id,
    agent_id=agent.id
)

# Si el run requiere acción humana:
if run.status == "requires_action":
    # Revisar y aprobar/rechazar la acción propuesta
    client.agents.submit_tool_outputs_to_run(
        thread_id=thread.id,
        run_id=run.id,
        tool_outputs=[{"tool_call_id": call.id, "output": "approved"}]
    )
```

**Buenas prácticas para agentes seguros:**

| Práctica                              | Descripción                                                        |
|---------------------------------------|--------------------------------------------------------------------|
| **Principio de mínimo privilegio**    | El agente solo tiene acceso a las herramientas que necesita        |
| **Validación de outputs**             | Verificar salidas del agente antes de ejecutar acciones externas   |
| **Sandboxing de Code Interpreter**    | Ejecuta código en entorno aislado — no puede acceder a red externa |
| **Logging de todas las tool calls**   | Registrar inputs/outputs de cada herramienta invocada              |
| **Límites de tokens y steps**         | Prevenir consumo excesivo y comportamientos inesperados            |

### Aplicación práctica

**Checklist de IA Responsable para poner en producción:**

```
☐ Filtros de contenido configurados (hate, violence, sexual, self-harm)
☐ Prompt Shield habilitado (direct + indirect injection)
☐ Evaluaciones de calidad definidas (coherence, groundedness, relevance)
☐ Dataset de evaluación (golden set) creado y versionado
☐ Evaluaciones automáticas integradas en el pipeline CI/CD
☐ Logging de inputs/outputs habilitado (con política de PII)
☐ Thread IDs y Run IDs registrados para trazabilidad
☐ Instrucciones de sistema claras con restricciones explícitas
☐ Herramientas del agente limitadas al mínimo necesario
☐ Proceso de aprobación humana definido para acciones de alto riesgo
☐ Política de revisión periódica de logs y evaluaciones
```

### Puntos clave para el examen

- ✅ Filtros de contenido aplican a **inputs Y outputs** — son bidireccionales.
- ✅ **Prompt Shield** protege contra inyección directa (usuario) e indirecta (documentos/datos externos).
- ✅ **Groundedness** mide si la respuesta está fundamentada en el contexto RAG — la herramienta para detectar alucinaciones.
- ✅ Las **evaluaciones** de Azure AI Evaluation usan un modelo juez (LLM-as-judge) para puntuar métricas como coherencia y relevancia.
- ✅ `thread_id` + `run_id` = trazabilidad de conversaciones y ejecuciones de agentes.
- ✅ **Mínimo privilegio en agentes**: configurar solo las tools necesarias, con permisos mínimos.
- ✅ El logging de inputs/outputs puede exponer **PII** — requiere política de privacidad explícita.
- ✅ **Human-in-the-loop** = el run entra en estado `requires_action` → aprobación manual antes de continuar.
- ✅ Los **filtros de contenido no se pueden desactivar completamente** sin aprobación especial de Microsoft.

---

## Resumen visual de la sección

```
PLANEAR Y ADMINISTRAR UNA SOLUCIÓN DE IA AZURE (25-30%)
│
├── 1.1 ELEGIR SERVICIOS
│     ├── Modelos: GPT-4o (complejo) | GPT-4o-mini/Phi-4 (costo) | DALL-E (imágenes)
│     ├── Servicios: Agent Service | AI Search | Content Safety | Evaluation | Prompt Flow
│     └── RAG: Búsqueda Híbrida (BM25 + Vector) + Semantic Reranker
│
├── 1.2 CONFIGURAR
│     ├── Hub → Project → Resources (jerarquía fundamental)
│     ├── Deployments: Standard | Global Standard | Provisioned (PTU)
│     └── CI/CD: Evaluaciones como gate de calidad antes de producción
│
├── 1.3 ADMINISTRAR / SUPERVISAR / PROTEGER
│     ├── Cuotas: TPM/RPM | PTU para capacidad garantizada
│     ├── Monitor: KQL en Log Analytics | Application Insights | Alertas
│     └── Seguridad: Managed Identity | Private Endpoints | RBAC | Key Vault
│
└── 1.4 IA RESPONSABLE
      ├── Content Filters: Hate/Violence/Sexual/Self-harm (input + output)
      ├── Prompt Shield: Inyección directa e indirecta
      ├── Evaluation: Coherence/Relevance/Groundedness/Fluency
      └── Agentes: HITL | Mínimo privilegio | thread_id + run_id
```

---

## Recursos

### Documentación oficial Microsoft

- [Azure AI Foundry documentation](https://learn.microsoft.com/azure/ai-foundry/)
- [Azure AI Foundry Hub & Projects](https://learn.microsoft.com/azure/ai-foundry/concepts/ai-resources)
- [Model catalog in Azure AI Foundry](https://learn.microsoft.com/azure/ai-foundry/how-to/model-catalog-overview)
- [Deployment types for Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/how-to/deployment-types)
- [Azure AI Search - Hybrid search](https://learn.microsoft.com/azure/search/hybrid-search-overview)
- [Azure AI Agent Service](https://learn.microsoft.com/azure/ai-services/agents/overview)
- [Azure AI Content Safety](https://learn.microsoft.com/azure/ai-services/content-safety/overview)
- [Azure AI Evaluation SDK](https://learn.microsoft.com/azure/ai-foundry/how-to/evaluate-sdk)
- [Prompt Shield](https://learn.microsoft.com/azure/ai-services/content-safety/concepts/jailbreak-detection)
- [Managed Identity for Azure AI](https://learn.microsoft.com/azure/ai-services/authentication)
- [RBAC roles for Azure AI Foundry](https://learn.microsoft.com/azure/ai-foundry/concepts/rbac-ai-foundry)
- [Responsible AI in Azure AI Foundry](https://learn.microsoft.com/azure/ai-foundry/responsible-ai)
- [Monitor Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/how-to/monitor-openai)

### Servicios relacionados para el examen

- [Phi-3 / Phi-4 model family](https://learn.microsoft.com/azure/ai-foundry/how-to/deploy-models-phi-4)
- [Azure AI Evaluation - metrics reference](https://learn.microsoft.com/azure/ai-foundry/concepts/evaluation-metrics-built-in)
- [Prompt Flow overview](https://learn.microsoft.com/azure/machine-learning/prompt-flow/overview-what-is-prompt-flow)
- [DefaultAzureCredential reference](https://learn.microsoft.com/dotnet/azure/sdk/authentication/credential-chains)

---

> 📝 **Última actualización:** Junio 2025 | Basado en objetivos del examen AI-103
