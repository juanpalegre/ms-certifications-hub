# AB-100 — Agentic AI Business Solutions Architect

> Diseño de soluciones de IA agéntica enterprise sobre la plataforma Microsoft: Azure AI Foundry, Copilot Studio, Microsoft 365 Copilot, Dynamics 365, Power Platform.

📅 **Estado:** ✅ certificado (2026)
🔗 **Study guide oficial:** [learn.microsoft.com](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ab-100)

---

## Rol del candidato

El AB-100 es el **arquitecto** del stack: define qué construir, qué servicios usar, cómo gobernar costos, seguridad y Responsible AI. No implementa el código (AI-103) ni opera el ciclo de vida (GH-600).

```
AB-100  →  Define qué construir (arquitectura, diseño, gobernanza)
AI-103  →  Construye el núcleo de IA (Python, RAG, agentes)
GH-600  →  Automatiza el ciclo de vida (CI/CD, Copilot agents, MCP)
```

---

## Aptitudes de un vistazo

| # | Sección | Peso | Prioridad |
|---|---|---|---|
| 1 | Planear soluciones de negocio con IA | 25–30% | ⚠️ Alta |
| 2 | Diseñar soluciones de negocio con IA | 25–30% | ⚠️ Alta |
| 3 | Implementar soluciones de negocio con IA | 40–45% | ⚠️⚠️ Muy alta |

> La sección 3 (deploy: monitoreo, testing, ALM, seguridad) pesa casi la mitad del examen.

---

## Guías de estudio

| Archivo | Sección | Contenido |
|---|---|---|
| [01-planear-soluciones.md](./01-planear-soluciones.md) | Planear (25–30%) | Rol del arquitecto, análisis de requisitos, PRALID, CAF, multi-agente, ROI/TCO, Build/Buy/Extend, Model Router |
| [02-disenar-soluciones.md](./02-disenar-soluciones.md) | Diseñar (25–30%) | 3 tipos de agentes, extensibilidad en 4 capas, MCP, computer use, orquestación OOB en D365/M365/Power Platform |
| [03-implementar-soluciones.md](./03-implementar-soluciones.md) | Implementar (40–45%) | Monitoreo y tuning, testing de agentes, ALM de datos en 7 fases, seguridad, prompt injection, IA Responsable |
| [cheat-sheet.md](./cheat-sheet.md) | Repaso rápido | Frameworks numerados, decisiones rápidas, distinciones clave |

---

## Temas centrales del examen

- **Diseño de soluciones agénticas** — agentes de tareas, autónomos y conversacionales; orquestación multi-agente
- **Ecosistema Microsoft AI** — Azure AI Foundry, Copilot Studio, Microsoft 365 Copilot, Dynamics 365, Power Platform
- **Protocolos abiertos** — Model Context Protocol (MCP), Agent2Agent (A2A)
- **Arquitectura enterprise** — ALM para IA, estrategia de environments, Well-Architected Framework
- **Responsible AI** — gobernanza, compliance, defensa contra prompt injection, residencia de datos, auditoría
- **Alineación con negocio** — ROI/TCO, Build/Buy/Extend, Cloud Adoption Framework, AI Center of Excellence

---

## Caso práctico

➡️ [Bot de WhatsApp visto desde AB-100](../../casos-practicos/bot-whatsapp/) — arquitectura, selección de servicios y gobernanza aplicadas a un proyecto real.
