# Sección 2 — Diseñar soluciones de negocio con IA (25–30%)

> Cubre: diseño de IA y agentes, extensibilidad de soluciones y orquestación de agentes/apps precompilados.
> Módulos del learning path: M05 (diseño de agentes), M06 (extensibilidad), M07 (orquestación OOB).

---

## 1. Diseño de agentes

### Los 3 tipos de agentes en Copilot Studio

| | Task Agent | Agente autónomo | Agente conversacional |
|---|---|---|---|
| **Propósito** | Acciones específicas en flujos estructurados | Lograr objetivos sin interacción constante | Interacción en lenguaje natural |
| **Se inicia** | Instrucción del usuario en cada paso | **Triggers** (eventos, cambios de sistema, horarios) | Mensajes/preguntas del usuario |
| **Componentes** | Objetivos, habilidades, acciones, conocimiento, contexto, guardrails | Objetivos, triggers, instrucciones, conocimiento, acciones, publicación | Topics + fallback, prompt actions, agent flows, NLP |
| **Ejemplo** | Crear registro de ventas, presentar ticket | Onboarding automático, tickets por sentimiento | Bot de atención, asistente de RRHH |

El autónomo se construye en 4 pasos (caso de uso → construir → probar/refinar → **publicar en Teams**) y lleva instrucciones de sistema explícitas.

### Arquitectura del task agent — proceso de diseño

1. Propósito con enunciado de outcome empresarial
2. Objetivos medibles, observables, testeables
3. Habilidades (language understanding, planificación, ejecución)
4. Acciones vía conectores Power Platform, APIs, Dataverse, cloud flows
5. Fuentes de conocimiento exactas, actualizadas, gobernadas e indexadas
6. **Guardrails**: reglas que definen qué puede y no puede hacer ("nunca sobrepasar el límite de crédito")

Best practice crítica: **least privilege** — limitar permisos de acción al mínimo necesario.

### IA Responsable aplicada a todo el ciclo de vida

Los 6 principios no son checklist de diseño inicial — aplican en **cada fase**: diseño (riesgos y escenarios de seguridad desde el día uno), desarrollo (pruebas de equidad, documentación de transparencia), deploy (supervisión humana, validación pre-producción), operaciones (mejora continua, reentrenamiento ante cambios de política).

### Criterios de éxito — se definen ANTES del diseño

Con stakeholders del negocio y el AI CoE, alineados con los outcomes del ROI. Categorías: adopción (usuarios activos, sesiones), automatización (tareas sin intervención), satisfacción (feedback, escalaciones), valor de negocio (tiempo ahorrado, casos resueltos). Sin métricas no hay valor demostrable.

### Power Platform Well-Architected Framework — 5 pilares

| Pilar | Qué cubre | Señal en el examen |
|---|---|---|
| Confiabilidad | Reintentos, failover, conectores redundantes | "El conector falla intermitentemente" |
| Seguridad | Least privilege (Entra ID), políticas DLP | "Acceso sin roles definidos" |
| Excelencia operativa | Admin Center Analytics, ALM con DevOps/GitHub | "No hay logs de cambios" |
| Eficiencia del rendimiento | Dataverse para alta carga, Azure Functions para offload | "La app se degrada bajo carga" |
| Optimización de la experiencia | UX consistente, Copilot-assisted, accesibilidad | "Usuarios abandonan el flujo" |

Alineado con el Azure WAF — coherencia entre arquitecturas cloud y low-code.

---

## 2. Extensibilidad de soluciones

### Las 4 capas de extensibilidad de Copilot Studio (en orden de esfuerzo)

| Capa | Qué define | Esfuerzo | Cuándo |
|---|---|---|---|
| **① Instrucción** | Propósito, rol, restricciones, reglas de escalación. *Prompt modification* sin regenerar el agente | Sin código | Solo cambia comportamiento |
| **② Habilidades** | Skills modulares y reutilizables: retrieval, acciones de dominio, flujos multi-paso | Low-code | Capacidades del dominio |
| **③ Integración** | Sistemas de registro (D365, M365, APIs externas), Power Automate, eventos | Conectores/APIs | Integrar SAP u otro sistema externo |
| **④ Pro-code (VS Code)** | Lógica custom, herramientas en código, orquestación personalizada, ALM con GitHub/DevOps | Pro-code | Alta complejidad |

Principio del arquitecto: skills modulares compartidas por el CoE — no duplicar la misma skill en varios agentes.

### Cuándo usar cada plataforma extendida

- **Azure AI Foundry**: modelos custom o fine-tuned, *connected agents* (agente principal + especialistas con permisos mínimos), evaluaciones propias, control total del pipeline.
- **Agentes declarativos de M365 Copilot**: **no modifican el modelo base** — definen ámbito vía instrucciones, conocimiento y acciones. Heredan IA Responsable de M365. Para productividad en Word/Excel/Teams/Outlook. Ámbito acotado: agentes declarativos demasiado amplios generan respuestas inconsistentes.
- **Computer use (Copilot Studio)**: RPA inteligente — el agente "ve" la pantalla e interactúa como un humano. **Para apps legacy sin API** y formularios web complejos. Alto riesgo: acceso visual a datos en pantalla → gobernanza y auditoría estrictas.

### Model Context Protocol (MCP)

Contrato estructurado que define **qué contexto puede acceder un agente y cómo interpretarlo**. En AB-100 aparece especialmente ligado a **Dynamics 365 Finance & Operations**:

| Contexto MCP expone | Permite al agente |
|---|---|
| Entidades de datos (clientes, proveedores, productos) | Responder con datos reales del ERP sin alucinar |
| Metadatos de proceso (workflows, aprobaciones) | Entender el estado del proceso y actuar correcto |
| Modelos de dominio (dimensiones financieras) | Respuestas alineadas a las reglas contables |
| Reglas de localización | Adaptar comportamiento por jurisdicción |

Beneficios: semántica empresarial consistente entre agentes, menos alucinaciones, interoperabilidad multi-app, auditabilidad. Distinto del grounding genérico.

---

## 3. Orquestación de agentes y apps precompilados

El arquitecto **configura y orquesta lo que ya existe** — no diseña desde cero. OOB siempre antes que custom.

### Los 3 modelos de experiencia de IA en Dynamics 365

| Modelo | Dónde vive | Ideal para |
|---|---|---|
| **Sidecar** | Panel de chat lateral a la app | Consultas NL, resúmenes, sin interrumpir el flujo |
| **Embedded** | Dentro del formulario/workspace, contextual | IA mientras el usuario trabaja en la página |
| **Externo** | Desde Teams u otras apps (vía Dataverse/APIs) | Cross-app, roles específicos, notificaciones |

### Capacidades OOB por producto D365

| Producto | Capacidades clave |
|---|---|
| **Finance** | Resúmenes del coordinador de colecciones, análisis de oportunidades/riesgos, chat NL con datos F&O. Extensión: plugins X++ + contexto MCP |
| **Supply Chain** | Planificación de demanda con IA, agente de comunicación con proveedores, revisión de cambios en POs confirmados |
| **Customer Service** | **Agent Hub** + 4 agentes autónomos OOB + Copilot en Contact Center (asistencia en tiempo real) |
| **Sales** | Copilot for Sales: captura CRM automática, resúmenes de reunión, próximos pasos |

### Los 4 agentes autónomos OOB de Customer Service

1. **Intención del cliente** — detecta intenciones analizando casos pasados y actuales
2. **Administración de casos** — automatiza creación, actualización, resolución, cierre
3. **Gestión de conocimiento** — mantiene y mejora la knowledge base
4. **Evaluación de calidad** — evalúa interacciones automáticamente

**Agent Hub ≠ agente**: es el panel de administración donde supervisores adoptan y monitorean agentes de forma segura.

### Agentes M365 — cuándo proponer cada uno

| Agente | Escenario | Consideración del arquitecto |
|---|---|---|
| Copilot for Sales | Ventas con D365 Sales/Salesforce + Teams/Outlook | Configurar qué datos del CRM se acceden |
| Copilot for Service | Soporte con casos y KB de D365 | Fuentes de conocimiento autorizadas y actualizadas |
| M365 Copilot Chat | Productividad general sobre Microsoft Graph | Revisar permisos del índice semántico por usuario |

### Power Platform AI Hub

Para usuarios de negocio **sin involucrar a IT**: AI Builder (modelos precompilados sin ML expertise — distinto de Azure ML, que es para data scientists), Copilot en Power Apps (NL sobre datos), Copilot en Power Automate (describir → generar flujo), Copilot en Power BI/Fabric (informes e insights NL).

---

## Tips de examen — Sección 2

- **Los 3 tipos de agentes**: escenario → tipo. Trigger sin usuario = autónomo; flujo estructurado paso a paso = task; NLP con topics = conversacional.
- **Las 4 capas de extensibilidad en orden**: comportamiento = capa 1; integrar sistema externo = capa 3; orquestación custom = capa 4.
- **Computer use = apps legacy sin API** — siempre con advertencia de gobernanza estricta.
- **Agentes declarativos NO modifican el modelo base** (≠ fine-tuning) y heredan RAI de M365.
- **MCP**: contrato de contexto empresarial, foco en D365 F&O, reduce alucinaciones.
- **Sidecar / Embedded / Externo**: "IA mientras trabaja en el formulario" = Embedded.
- **Los 4 agentes de Customer Service**: "automatizar el ciclo de vida del caso" = Administración de Casos.
- **Agent Hub es panel de administración**, no un agente.
- **Criterios de éxito ANTES del diseño**, con CoE y stakeholders — definir métricas después del deploy es la trampa.
- **WAF de Power Platform**: mapear problema → pilar.

---

⬅️ [Sección 1 — Planear](./01-planear-soluciones.md) · ➡️ [Sección 3 — Implementar](./03-implementar-soluciones.md)
