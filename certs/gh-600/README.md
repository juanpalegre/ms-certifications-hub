# GH-600 — GitHub Agentic AI Engineer (Beta)

> Operación, integración, supervisión y administración de agentes de IA dentro de flujos SDLC en GitHub.

📅 **Estado:** examen rendido (beta, 2026)  
🔗 **Study Guide:** [learn.microsoft.com](https://learn.microsoft.com/es-mx/credentials/certifications/resources/study-guides/gh-600)

---

## 🧪 Caso práctico

| Proyecto | Descripción |
|---|---|
| [🤖 Bot WhatsApp para club deportivo](../../casos-practicos/bot-whatsapp/) | Bot de IA integrado con WhatsApp para consultas y reservas de un club de pádel. Aplica las 6 secciones del examen en un proyecto real con Azure AI Foundry, MCP servers, multi-agente y CI/CD con GitHub Actions. |

---

## 📚 Guías de estudio

| # | Sección | Peso | Archivo |
|---|---|---|---|
| 1 | Arquitectura del agente y procesos SDLC | 15-20% | [01-arquitectura-agente-sdlc.md](./01-arquitectura-agente-sdlc.md) |
| 2 | Herramientas e interacción del entorno | 20-25% ⚠️ | [02-herramientas-entorno.md](./02-herramientas-entorno.md) |
| 3 | Memoria, estado y ejecución | 10-15% | [03-memoria-estado-ejecucion.md](./03-memoria-estado-ejecucion.md) |
| 4 | Evaluación, errores y ajuste | 15-20% | [04-evaluacion-errores-ajuste.md](./04-evaluacion-errores-ajuste.md) |
| 5 | Orquestación multiagente | 15-20% ⚠️ | [05-orquestacion-multiagente.md](./05-orquestacion-multiagente.md) |
| 6 | Protección y responsabilidad | 10-15% | [06-proteccion-responsabilidad.md](./06-proteccion-responsabilidad.md) |
| — | Cheat sheet (repaso rápido) | — | [cheat-sheet.md](./cheat-sheet.md) |

---

## Aptitudes evaluadas

### 1. Preparar arquitectura del agente y procesos SDLC (15–20%)
- Integrar agentes en el SDLC
- Definir límites entre planificación, razonamiento y acción
- Configurar observabilidad y control de agentes autónomos

### 2. Implementar uso de herramientas e interacción del entorno (20–25%)
- Selección y configuración de herramientas del agente
- Configuración de servidores MCP
- Integración de agentes en entornos de desarrollo
- Rutas de ejecución seguras y control de errores

### 3. Administrar memoria, estado y ejecución (10–15%)
- Estrategias de memoria (externa, corto plazo, largo plazo)
- Persistir estado y administrar deriva de contexto
- Continuidad entre herramientas y entornos

### 4. Evaluación, análisis de errores y ajuste (15–20%)
- Criterios de éxito y señales de evaluación
- Análisis de errores e identificación de causas
- Ajuste de comportamiento basado en evaluación

### 5. Orquestar coordinación de múltiples agentes (15–20%)
- Flujos de trabajo multi-agente
- Observabilidad multi-agente
- Detección y respuesta a errores multi-agente
- Ciclo de vida de agentes en flujos multi-agente

### 6. Implementar límites de protección y responsabilidad (10–15%)
- Definición de niveles de autonomía
- Barandillas de seguridad y flujos human-in-the-loop

---

## Recursos de estudio

| Recurso | Link |
|---|---|
| Foundations of Agentic AI in GitHub | [Microsoft Learn](https://learn.microsoft.com/es-es/training/modules/foundations-agentic-ai/) |
| Designing Agent Architecture and SDLC Integration | [Microsoft Learn](https://learn.microsoft.com/es-es/training/modules/design-agent-architecture-integration/) |
| Tooling, MCP y Agent Execution Environments | [Microsoft Learn](https://learn.microsoft.com/es-es/training/modules/agent-tooling-mcp-execution-environments/) |
| GitHub Docs — Custom Agents | [docs.github.com](https://docs.github.com/en/copilot/how-tos/copilot-sdk/use-copilot-sdk/custom-agents) |
| GitHub Docs — Copilot Memory | [docs.github.com](https://docs.github.com/en/copilot/concepts/agents/copilot-memory) |

---

## Perfil del candidato

Experiencia en:
- Flujos de trabajo de agentes dentro del SDLC
- Supervisión de comportamiento autónomo con controles de GitHub
- Evaluación y ajuste de resultados del agente
- Configuración de agentes personalizados (instrucciones, agentes, herramientas, MCP)
- Coordinación de ejecución multi-agente de forma segura

---

## 📁 Contenido

```
certs/gh-600/
├── README.md                          ← este archivo
├── 01-arquitectura-agente-sdlc.md     ← Sección 1 (15-20%)
├── 02-herramientas-entorno.md         ← Sección 2 (20-25%)
├── 03-memoria-estado-ejecucion.md     ← Sección 3 (10-15%)
├── 04-evaluacion-errores-ajuste.md    ← Sección 4 (15-20%)
├── 05-orquestacion-multiagente.md     ← Sección 5 (15-20%)
├── 06-proteccion-responsabilidad.md   ← Sección 6 (10-15%)
└── cheat-sheet.md                     ← Repaso rápido
```

Caso práctico completo en [`casos-practicos/bot-whatsapp/`](../../casos-practicos/bot-whatsapp/).
