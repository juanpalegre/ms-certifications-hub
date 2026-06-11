# Sección 1 — Planear soluciones de negocio con IA (25–30%)

> Cubre: análisis de requisitos, diseño de la estrategia general de IA y evaluación económica (ROI/TCO).
> Módulos del learning path: M01 (rol del arquitecto), M02 (análisis de requisitos), M03 (estrategia), M04 (costos y beneficios).

---

## 1. El rol del arquitecto AB-100

El arquitecto de soluciones de IA es el **puente entre la estrategia de negocio y la implementación técnica**. Define, alinea y decide — no es quien codifica (eso es AI-103) ni quien opera el ciclo de vida (GH-600).

### Las 5 responsabilidades del arquitecto

| # | Responsabilidad | Qué implica |
|---|---|---|
| 1 | **Visión y hoja de ruta** | Estrategia de adopción de IA alineada a prioridades del negocio |
| 2 | **Arquitectura de datos** | Preparación y gobernanza de los datos |
| 3 | **Integración** | Conectar servicios de IA con flujos de trabajo empresariales |
| 4 | **Seguridad y ética** | Implementar principios de IA Responsable |
| 5 | **Supervisión del rendimiento** | Establecer KPIs y medir la eficacia |

### El marco de transformación (iterativo, no lineal)

```
① Objetivos empresariales → ② Estrategia de IA → ③ Diseño de arquitectura → ④ Implementación → ⑤ Supervisar y optimizar ↺ (retroalimenta al inicio)
```

### Ecosistema Microsoft AI — qué usa el arquitecto y cuándo

| Tecnología | Categoría | Cuándo la usa el arquitecto |
|---|---|---|
| Azure AI Services | APIs precompiladas | Capacidades de IA puntuales (visión, voz, lenguaje) sin entrenar modelos |
| Azure Machine Learning | Plataforma ML | Modelos personalizados para casos específicos |
| Azure OpenAI Service | IA generativa | Chatbots, generación de contenido, análisis semántico |
| **Azure AI Foundry** | Plataforma de agentes | Soluciones complejas multi-agente, orquestación pro-code |
| **Microsoft 365 Copilot** | IA embebida | Productividad del knowledge worker sin desarrollo custom |
| **Copilot Studio** | Low-code agentes | Lógica propia sin código extenso |
| Dynamics 365 Copilot | IA de negocio | Procesos en CRM/ERP (Sales, Service, Finance, SCM) |
| Power Platform AI | Low-code IA | AI Builder, Copilot en Power Apps/Automate para usuarios de negocio |

### Recursos OOB (out-of-the-box)

Regla central del arquitecto: **OOB primero, custom cuando es necesario**. Microsoft provee agentes precompilados (Copilot for Sales, Copilot Customer Service, Copilot Finance/SCM, M365 Copilot Chat, plantillas de Copilot Studio), herramientas y plantillas de escenarios. Beneficios: implementación más rápida, escalabilidad nativa con M365/D365 y cumplimiento de IA Responsable garantizado.

### Los 6 principios de IA Responsable de Microsoft

| Principio | Significado |
|---|---|
| Equidad | Sin sesgo ni discriminación |
| Confiabilidad y seguridad | Funciona como se espera |
| Privacidad y seguridad | Protege los datos |
| Inclusión | Accesible para todos |
| Transparencia | Explicable y comprensible |
| Responsabilidad | Rendición de cuentas |

---

## 2. Análisis de requisitos

### Las 3 áreas de valor de los agentes

| Área | Qué hacen los agentes | Herramientas |
|---|---|---|
| **Automatización de tareas** | Redactar documentos, resumir conversaciones, disparar flujos multi-paso | M365 Copilot, Copilot Studio, Power Automate |
| **Análisis de datos** | Lenguaje natural → insights accionables, tendencias, visualizaciones | Copilot, Fabric, Power BI |
| **Toma de decisiones** | Recomendaciones basadas en datos, identificación de riesgos, contexto resumido | Agentes con grounding |

Regla: empezar siempre por el **resultado empresarial**, no por la tecnología.

### Grounding y las 5 dimensiones de calidad de datos

**Grounding** = conectar el agente a datos organizacionales de confianza para que sus respuestas sean precisas y contextualmente correctas (≠ fine-tuning, que modifica los pesos del modelo).

Mnemotecnia **PRALID** — si alguna dimensión falla, la calidad del agente se degrada:

| Dimensión | Qué exige | Si falla… |
|---|---|---|
| **P**recisión | Datos validados por SMEs, coincidentes con fuentes autorizadas | Respuestas incorrectas o dañinas |
| **R**elevancia | Alineados al caso de uso específico | Respuestas conceptualmente similares pero contextualmente erróneas |
| **A**ctualidad | Información reciente, ciclos de refresh | Decisiones sobre información obsoleta |
| **L**impieza | Sin duplicados, estructura clara, formato estable | "Data contamination": embeddings y retrieval degradados |
| **D**isponibilidad | El usuario tiene acceso; datos indexados en Graph | El agente no puede recuperar el contenido |

La disponibilidad está ligada a **permisos**: la Copilot Retrieval API respeta los permisos del usuario. Si los datos existen pero el usuario no tiene acceso, el agente no los ve.

### Arquitectura de datos para IA — las 4 capas (CAF)

| Capa | Propósito | Tecnologías |
|---|---|---|
| Bases de datos operativas | Datos estructurados de apps | Azure SQL, Cosmos DB, Dynamics 365 |
| Analíticos (Lakehouse) | Datos curados para IA/ML | Microsoft Fabric, Synapse, Data Lake |
| Capa de inteligencia | Grounding, retrieval, búsqueda semántica | Azure AI Search, índice semántico M365, vector search |
| IA Apps + Agentes | Consumir datos para generar valor | M365 Copilot, Copilot Studio, Foundry, apps RAG |

**RAG** es el patrón arquitectónico de referencia: separa las fuentes de confianza del modelo; recupera, inyecta en el prompt y responde grounded. Ventajas: datos en tiempo real, privacidad conservada, menos alucinaciones.

Buenas prácticas de organización de datos: centralizar (Azure/Dataverse/Fabric), normalizar y estructurar, indexación semántica, múltiples rutas de acceso (APIs, índices, Graph Connectors), gobernanza temprana con **Microsoft Purview**, datos autoritativos y actualizados.

---

## 3. Estrategia general de IA

### CAF de Azure mapeado al ciclo de vida del agente

| Fase CAF | Ciclo de vida del agente | Output clave |
|---|---|---|
| Strategy | Plan Agents | AI Strategy brief + Agent Tech Plan |
| Plan | Plan Agents (cont.) | Adoption plan + Agent Readiness Assessment |
| Ready | Govern & Secure (base) | AI Landing Zone + Data Access Model |
| Govern + Secure | Govern & Secure (ejecución) | Policy set + Risk register + Security controls |
| Build (implícito) | Build Agents | Agent templates + Evaluation gates |
| Manage | Operate Agents | Observability baseline + Agent Ops Playbook |

### Decisión de plataforma — la escalera SaaS → Low-code → Pro-code

| Plataforma | Tipo | Mejor para | Limitaciones |
|---|---|---|---|
| M365 Copilot | SaaS OOB | Productividad M365, bajo esfuerzo, IA Responsable incluida | Autonomía baja, customización limitada |
| Copilot Studio | Low-code | Agentes de negocio custom, conectores, iteración rápida | Orquestación compleja limitada |
| Azure AI Foundry | Pro-code | Multi-agente complejo, tools custom, control del runtime | Mayor inversión y expertise |

Subir de escalón **solo cuando hay razones técnicas claras**. Empezar simple, escalar con evidencia.

### Single agent vs multi-agente

Multi-agente **solo** si se cumple al menos una:
- Se cruzan límites de seguridad o cumplimiento
- Múltiples equipos con ciclos de release independientes
- El roadmap expande hacia 3-5+ funciones
- Acciones con dependencias en 2+ workstreams

Si no: un solo agente. Muchas necesidades "multi-agente" se resuelven con persona switching, mejor retrieval o más contexto.

### Los 5 patrones de orquestación (Microsoft Agent Framework SDK)

| Patrón | Mecánica | Ideal para |
|---|---|---|
| **Sequential** | Pipeline A → B → C | Tareas por etapas con dependencias ordenadas |
| **Concurrent** | Paralelo + agregación | Subtareas independientes, análisis multi-fuente |
| **Group Chat** | Conversación mediada por moderador | Decisiones que requieren múltiples perspectivas |
| **Handoff** | Transferencia a especialista por regla/umbral | Escalación de soporte, casos que superan al agente primario |
| **Magentic** | Orquestador atrae expertos dinámicamente en runtime | Problemas no estructurados, expertos no conocidos de antemano |

### Custom agent vs extender M365 Copilot

| Señal del escenario | Decisión |
|---|---|
| Retrieval simple, flujos solo M365, low-code, heredar RAI | **Extender Copilot** |
| Workflow complejo multi-paso, sistemas externos, autonomía/triggers, multi-agente | **Custom agent** |

Híbrido frecuente: extender + algún agente custom para casos de borde.

### Prompt engineering a escala organizacional

- Estructura: contexto → instrucción específica → formato de salida → ejemplo si hace falta.
- Técnicas: zero-shot, few-shot (2-5 ejemplos), chain of thought, role prompting, retrieval-augmented.
- La **biblioteca de prompts es un activo organizacional**: gestionada como código (versionado, revisión por SMEs, gobernanza del CoE).

### AI Center of Excellence (CoE)

El CoE **no construye soluciones — gobierna**. Pilares: estándares y políticas, gobernanza (aprobación + auditoría de agentes), habilitación (formación + biblioteca de prompts), métricas (adopción, ROI, calidad) y roadmap. Previene el *agent sprawl*.

### Cumplimiento regulatorio

GDPR, EU AI Act, CCPA, LGPD: el arquitecto evalúa regulaciones de datos **por región antes de definir la arquitectura** (compliance by design). Incluye residencia de datos, clasificación de riesgo del modelo y evaluaciones de impacto en privacidad.

---

## 4. Evaluación económica: ROI, TCO y Build/Buy/Extend

### Las 5 categorías de ROI

| Categoría | Métricas típicas |
|---|---|
| Productividad | Tiempo ahorrado por tarea, tasa de automatización, adopción |
| Ahorro de costos | Horas reducidas, menos tickets de soporte |
| Impacto en ingresos | Conversión, calificación de leads, retención |
| Reducción de riesgos | Menos infracciones, menor tasa de error |
| Valor estratégico | Nuevos modelos de negocio, diferenciación (cualitativo) |

### TCO — los 5 dominios de costo

1. **Infraestructura** (compute, storage, redes)
2. **Desarrollo e integración** (diseño de agentes, orquestación)
3. **Calidad y preparación de datos** (limpieza, grounding, indexación)
4. **Experiencia y recursos** (ingenieros, MLOps, formación)
5. **Operaciones continuas** (monitoreo, reentrenamiento, licencias)

\+ gestión del cambio y costos de desmantelamiento. Proyectar a 3-5 años. **TCO responde "¿cuánto cuesta?"; ROI responde "¿vale la pena?"** — TCO es el denominador del ROI.

### El análisis de ROI en 6 pasos

1. Definir outcomes empresariales medibles ("reducir handle time 20%")
2. Identificar drivers de ROI (mapear a las 5 categorías)
3. Calcular TCO completo (5 dominios, 3-5 años)
4. Cuantificar beneficios (**Copilot Studio ROI Analytics**, time-and-motion, KPIs)
5. Comparar: payback period, NPV, ratio costo-beneficio, ROI anualizado
6. Validar con stakeholders (Finanzas, Operations, **el CoE valida las métricas**)

### Build / Buy / Extend

| Criterio | BUILD | BUY | EXTEND |
|---|---|---|---|
| Cuándo | Ventaja competitiva diferencial, compliance estricto, control total de datos | Time-to-value prioritario, proceso commodity, baja madurez interna | Agregar lógica interna a Copilot/plataforma existente |
| Time-to-value | Largo (meses) | Corto (semanas) | Medio |
| TCO inicial / operativo | Alto / Alto | Bajo / Medio | Medio / Medio |
| Riesgo principal | Costo y tiempo excesivos | Vendor lock-in | Límites de personalización |

**Extend es la respuesta más frecuente** en escenarios de madurez media (empresa con M365/D365 que quiere IA sin proyecto masivo).

### Model Router (Azure AI Foundry)

Un único endpoint que enruta cada solicitud al modelo más adecuado. Concepto nuevo y preguntado en AB-100.

- **Criterios de enrutamiento**: tipo de tarea, costo, latencia, dominio. Simple → SLM (barato); complejo → LLM; dominio → modelo ajustado.
- **4 tipos de reglas**: estáticas, ponderadas (A/B testing), fallback (confiabilidad), por versión (rollout seguro).
- **Implementación en 5 pasos**: estrategia → registrar modelos → configurar reglas → integrar → monitorear.

---

## Tips de examen — Sección 1

- **Las 5 responsabilidades del arquitecto** se preguntan por escenario: mapear situación → responsabilidad.
- **PRALID**: el examen da un escenario de datos y pregunta qué dimensión falla (datos viejos = Actualidad).
- **Grounding ≠ fine-tuning** — distinción directa.
- **Single agent por defecto**; multi-agente solo con evidencia (límites de seguridad, equipos independientes).
- **Escalera de plataformas**: M365 Copilot → Copilot Studio → Foundry. No saltar escalones sin razón técnica.
- **El CoE no construye, gobierna** — pregunta trampa clásica.
- **OOB primero, custom después** — en decisiones arquitectónicas, la opción correcta casi siempre arranca con lo existente.
- **Extend** gana en escenarios de madurez media; **ROI ≠ TCO**; los 5 dominios de TCO son pregunta directa.
- **Model Router**: propósito, 4 tipos de reglas, beneficios (costo + performance + confiabilidad).

---

⬅️ [README AB-100](./README.md) · ➡️ [Sección 2 — Diseñar soluciones](./02-disenar-soluciones.md)
