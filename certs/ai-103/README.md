# AI-103 — Azure AI Apps and Agents Developer Associate

> **Sucesor de AI-102.** Microsoft reemplazó AI-102 con AI-103 para reflejar el cambio de Cognitive Services individuales a Azure AI Foundry como plataforma unificada.
>
> 📅 **Estado:** en preparación (2026)
> 📖 **Guía de estudio oficial:** [AI-103 Study Guide (ES-MX)](https://learn.microsoft.com/es-mx/credentials/certifications/resources/study-guides/ai-103)

---

## Candidato objetivo

El ingeniero AI-103 es un **desarrollador Python** que construye aplicaciones e IA sobre **Azure AI Foundry**. No define la arquitectura (eso es AB-100), ni gestiona el CI/CD (eso es GH-600) — **implementa el núcleo de IA**: los pipelines RAG, los agentes, las integraciones de visión, lenguaje y extracción de documentos.

Trabaja en colaboración con:

| Rol | Certificación | Responsabilidad en el proyecto |
|---|---|---|
| Arquitecto de plataforma IA | AB-100 | Define la arquitectura, elige servicios, gobierna costos |
| Ingeniero DevOps / Copilot | GH-600 | CI/CD, GitHub Actions, Copilot coding agents, MCP servers |
| **Desarrollador de apps IA** | **AI-103** | **Implementa RAG, agentes, visión, lenguaje, extracción** |

---

## Conexión con AI-102: ¿Qué cambió?

AI-103 no es una actualización menor — es un rediseño que refleja cómo Microsoft reorientó toda su plataforma de IA hacia Foundry.

| AI-102 | AI-103 |
|---|---|
| Azure Cognitive Services (servicios individuales) | Azure AI Foundry (plataforma unificada) |
| Vision, Language, Speech como APIs separadas | Servicios integrados vía Foundry + SDK unificado |
| Azure Bot Service | Azure AI Agent Service |
| QnA Maker | Azure AI Search + RAG |
| Sin foco en agentes | Agentes y multi-agentes como tema central |
| SDK fragmentado por servicio | `azure-ai-projects` como punto de entrada único |

**Regla práctica:** si en AI-102 hacías `CognitiveServicesClient`, en AI-103 hacés `AIProjectClient` desde `azure-ai-projects`.

---

## Aptitudes de un vistazo

| # | Sección | Peso | Prioridad |
|---|---|---|---|
| 1 | Planear y administrar solución de IA Azure | 25–30% | ⚠️ Alta |
| 2 | IA generativa y agentes | 30–35% | ⚠️⚠️ Muy alta |
| 3 | Computer Vision | 10–15% | Media |
| 4 | Análisis de texto y Speech | 10–15% | Media |
| 5 | Extracción de información | 10–15% | Media |

> Las secciones 1 y 2 juntas suman hasta el **65%** del examen. Son el núcleo.

---

## Guías de estudio

| Archivo | Sección | Estado |
|---|---|---|
| [01-planear-administrar.md](./01-planear-administrar.md) | Planear y administrar Azure AI (25–30%) | ✅ Disponible |
| [02-ia-generativa-agentes.md](./02-ia-generativa-agentes.md) | IA generativa y agentes (30–35%) | ✅ Disponible |
| [03-computer-vision.md](./03-computer-vision.md) | Computer Vision (10–15%) | ✅ Disponible |
| [04-analisis-texto.md](./04-analisis-texto.md) | Análisis de texto y Speech (10–15%) | ✅ Disponible |
| [05-extraccion-informacion.md](./05-extraccion-informacion.md) | Extracción de información (10–15%) | ✅ Disponible |
| [cheat-sheet.md](./cheat-sheet.md) | Hoja de referencia rápida | 🔲 Pendiente |

---

## Stack técnico del examen

### Lenguaje principal

**Python** es el lenguaje que el examen da por sentado. Algunos snippets aparecen en C# o REST, pero la mayoría de la documentación oficial y los ejemplos de laboratorio son Python.

### SDKs que debés conocer

| SDK (PyPI) | Para qué sirve |
|---|---|
| `azure-ai-projects` | Punto de entrada unificado a Foundry — conexiones, agentes, evaluaciones |
| `azure-openai` | Llamadas directas a modelos OpenAI (chat, embeddings, vision) |
| `azure-search-documents` | Indexar y consultar Azure AI Search (RAG) |
| `azure-ai-documentintelligence` | Extraer texto estructurado de PDFs y documentos |
| `azure-ai-language` | NER, PII, sentiment, summarization, clasificación |
| `azure-cognitiveservices-speech` | STT (Whisper), TTS, transcripción de audio |
| `azure-ai-evaluation` | Medir groundedness, relevance, coherence de respuestas IA |
| `azure-identity` | Autenticación con Managed Identity / DefaultAzureCredential |

### Plataforma

- **Azure AI Foundry** → Hub + Project como unidad de trabajo
- **Azure AI Agent Service** → orquestación de agentes con function calling
- **Azure AI Search** → índice vectorial para RAG
- **Azure AI Content Safety** → filtros de contenido (input y output)
- **Azure AI Evaluation** → métricas de calidad para modelos en producción

### Servicios clave

```
Azure AI Foundry Hub
├── Azure AI Agent Service        ← agentes con tools
├── Azure OpenAI Service          ← GPT-4o, embeddings, Whisper
├── Azure AI Search               ← índice vectorial (RAG)
├── Azure AI Content Safety       ← filtros de entrada/salida
├── Azure AI Document Intelligence ← extracción de PDFs
├── Azure AI Language             ← NLP (NER, PII, sentiment)
└── Azure AI Evaluation           ← métricas de calidad
```

---

## Recursos de estudio

| Recurso | URL |
|---|---|
| Guía de estudio oficial AI-103 | [learn.microsoft.com/es-mx/credentials/certifications/resources/study-guides/ai-103](https://learn.microsoft.com/es-mx/credentials/certifications/resources/study-guides/ai-103) |
| Página de certificación AI-103 | [learn.microsoft.com/es-mx/credentials/certifications/azure-ai-engineer](https://learn.microsoft.com/es-mx/credentials/certifications/azure-ai-engineer) |
| Microsoft Learn — AI-103 Learning Path | [learn.microsoft.com/es-mx/training/paths/develop-ai-apps-azure](https://learn.microsoft.com/es-mx/training/paths/develop-ai-apps-azure) |
| Documentación Azure AI Foundry | [learn.microsoft.com/es-mx/azure/ai-foundry](https://learn.microsoft.com/es-mx/azure/ai-foundry) |
| Documentación Azure AI Agent Service | [learn.microsoft.com/es-mx/azure/ai-services/agents](https://learn.microsoft.com/es-mx/azure/ai-services/agents) |
| SDK azure-ai-projects (PyPI) | [pypi.org/project/azure-ai-projects](https://pypi.org/project/azure-ai-projects/) |
| Repositorio de ejemplos oficiales | [github.com/Azure-Samples/azureai-samples](https://github.com/Azure-Samples/azureai-samples) |
| Azure AI Evaluation docs | [learn.microsoft.com/es-mx/azure/ai-studio/how-to/evaluate-generative-ai-app](https://learn.microsoft.com/es-mx/azure/ai-studio/how-to/evaluate-generative-ai-app) |
| Azure AI Search — vector search | [learn.microsoft.com/es-mx/azure/search/vector-search-overview](https://learn.microsoft.com/es-mx/azure/search/vector-search-overview) |
| Azure Document Intelligence | [learn.microsoft.com/es-mx/azure/ai-services/document-intelligence](https://learn.microsoft.com/es-mx/azure/ai-services/document-intelligence) |
| Responsible AI — Content Safety | [learn.microsoft.com/es-mx/azure/ai-services/content-safety](https://learn.microsoft.com/es-mx/azure/ai-services/content-safety) |

---

## Estructura del repositorio

```
certs/ai-103/
├── README.md                       ← Este archivo
├── 01-planear-administrar.md       ← Sección 1: arquitectura, seguridad, costos, Foundry
├── 02-ia-generativa-agentes.md     ← Sección 2: RAG, agentes, evaluación, prompt engineering
├── 03-computer-vision.md           ← Sección 3: visión, análisis de imágenes, GPT-4o vision
├── 04-analisis-texto.md            ← Sección 4: NLP, Speech, Language
├── 05-extraccion-informacion.md    ← Sección 5: Document Intelligence, Content Understanding
└── cheat-sheet.md                  ← 🔲 Hoja de referencia rápida (pendiente)
```

---

## Conexión con las otras certificaciones del stack

Este repositorio vive en el contexto de un stack de tres certificaciones que trabajan sobre el mismo tipo de proyecto:

```
AB-100  →  Define qué construir (arquitectura, diseño, gobernanza)
  ↓
AI-103  →  Construye el núcleo de IA (Python, RAG, agentes, modelos)
  ↓
GH-600  →  Automatiza el ciclo de vida (CI/CD, Copilot agents, MCP)
```

Ver el caso práctico completo en: [`casos-practicos/bot-whatsapp/ai103-aplicacion-certificacion.md`](../../casos-practicos/bot-whatsapp/ai103-aplicacion-certificacion.md)
