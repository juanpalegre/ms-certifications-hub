# AB-100 — Cheat-sheet

> Repaso rápido pre-examen. Los números, frameworks y reglas que hay que tener frescos.

---

## Los números del examen

| Sección | Peso |
|---|---|
| 1. Planear soluciones de negocio con IA | 25–30% |
| 2. Diseñar soluciones de negocio con IA | 25–30% |
| 3. **Implementar** soluciones de negocio con IA | **40–45%** |

Aprobación: 700/1000.

---

## Frameworks numerados (memorizar)

| Framework | Contenido |
|---|---|
| **5 responsabilidades del arquitecto** | Visión/roadmap · Arquitectura de datos · Integración · Seguridad/ética · Supervisión |
| **6 principios de IA Responsable** | Equidad · Confiabilidad y seguridad · Privacidad y seguridad · Inclusión · Transparencia · Responsabilidad |
| **5 dimensiones de calidad de datos (PRALID)** | Precisión · Relevancia · Actualidad · Limpieza · dIsponibilidad |
| **4 capas de arquitectura de datos** | Operativas → Lakehouse → Inteligencia (AI Search) → Apps/Agentes |
| **5 patrones de orquestación** | Sequential · Concurrent · Group Chat · Handoff · Magentic |
| **5 categorías de ROI** | Productividad · Ahorro de costos · Ingresos · Riesgos · Valor estratégico |
| **5 dominios de TCO** | Infraestructura · Desarrollo · Datos · Expertise · Operaciones |
| **6 pasos del análisis de ROI** | Outcomes → Drivers → TCO → Beneficios → Comparar (payback/NPV) → Validar (CoE) |
| **3 tipos de agentes** | Task (paso a paso) · Autónomo (triggers) · Conversacional (topics/NLP) |
| **4 capas de extensibilidad** | Instrucción → Habilidades → Integración → Pro-code |
| **3 modelos de experiencia D365** | Sidecar (chat lateral) · Embedded (en el formulario) · Externo (desde Teams) |
| **4 agentes autónomos de Customer Service** | Intención · Casos · Conocimiento · Calidad |
| **5 pilares Power Platform WAF** | Confiabilidad · Seguridad · Excelencia operativa · Rendimiento · Experiencia |
| **4 dimensiones de métricas** | Operativas · Calidad · Usuario · Seguridad/cumplimiento |
| **4 tipos de prueba** | Escenarios · Rendimiento · Seguridad/cumplimiento · Usabilidad |
| **7 fases ALM de datos** | Plan → Ingesta → Desarrollar → Stage → Deploy → Operar → Evolución |
| **5 capas de defensa en profundidad** | Identidad → Datos → Evaluación → Observabilidad → Respuesta a incidentes |

---

## Decisiones rápidas (escenario → respuesta)

| Si el escenario dice… | La respuesta es… |
|---|---|
| Productividad en M365, bajo esfuerzo | M365 Copilot (extender, no construir) |
| Lógica de negocio custom, low-code | Copilot Studio |
| Multi-agente complejo, control del runtime | Azure AI Foundry |
| ¿Single o multi-agente? sin límites de seguridad ni equipos separados | **Single agent primero** |
| App legacy sin API | Computer use |
| Contexto empresarial consistente para D365 F&O | MCP |
| Empresa madurez media con M365/D365 que quiere IA | **Extend** (no Build ni Buy) |
| Tarea simple/clasificación vs razonamiento complejo | Model Router: SLM vs LLM |
| Análisis de intención en Customer Service | Agente de Intención OOB (no custom) |
| IA mientras el usuario trabaja en el formulario | Embedded (no Sidecar) |
| ¿Quién gobierna estándares y aprueba agentes? | AI CoE (**no construye, gobierna**) |
| Autenticación del agente | Identidad administrada (nunca secretos) |
| Usuario manipula al agente con instrucciones ocultas | Prompt injection → filtrado E/S + red-teaming |
| ¿Datos de producción para entrenar? | **NUNCA** |
| ¿Dónde se certifica el golden dataset? | Fase D (Stage) — lo aprueba el CAB |
| ¿Cuándo se definen baselines y criterios de éxito? | **ANTES** del deploy/diseño |

---

## Distinciones que el examen evalúa

- **Grounding ≠ fine-tuning** — contexto en runtime vs modificar pesos del modelo
- **ROI ≠ TCO** — vale la pena vs cuánto cuesta (TCO es el denominador)
- **Agent Hub ≠ agente** — panel de administración, no agente
- **Agentes declarativos ≠ fine-tuning** — definen ámbito, no modifican el modelo
- **AI Builder ≠ Azure ML** — usuarios de negocio vs data scientists
- **Foundry ≠ Copilot Studio** — pro-code vs low-code, complementarios
- **Telemetría operativa ≠ de comportamiento** — sistema vs interacción del usuario
- **Datos Red ≠ Gold** — experimental/mutable vs producción/inmutable
- **Prompt injection ≠ SQL injection** — vulnerabilidad de IA con mitigaciones propias

---

## Reglas de oro del arquitecto

1. **OOB primero, custom cuando es necesario.**
2. **Empezar simple, escalar con evidencia** (single → multi-agente; SaaS → low-code → pro-code).
3. **Resultado empresarial primero, tecnología después.**
4. **Least privilege en todo** — agentes, conectores, datos.
5. **Compliance by design** — regulaciones por región antes de la arquitectura.
6. **Métricas y baselines antes del deploy** — sin medición no hay valor demostrable.
7. **Defensa en profundidad** — nunca una sola capa de seguridad.
8. **El arquitecto diseña y coordina, no implementa solo.**

---

⬅️ [README AB-100](./README.md)
