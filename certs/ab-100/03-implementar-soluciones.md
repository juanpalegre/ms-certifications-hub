# Sección 3 — Implementar soluciones de negocio con IA (40–45%)

> **La sección de mayor peso del examen.** Cubre: supervisión y tuning, testing, ALM y seguridad/gobernanza/IA Responsable.
> Módulos del learning path: M08 (monitoreo), M09 (testing), M10 (ALM), M11 (seguridad y gobernanza).

---

## 1. Supervisión, análisis y optimización

### El ciclo continuo

```
① Recopilar (logs, telemetría, feedback) → ② Analizar (métricas vs baseline) → ③ Diagnosticar → ④ Ajustar (versionado + rollback) ↺
```

### Las 4 dimensiones de métricas

| Dimensión | Métricas |
|---|---|
| **Operativas** | Latencia, throughput, tasa de errores, consumo de tokens, uptime |
| **Calidad y razonamiento** | Precisión de respuesta, cobertura de conocimiento, eficacia de acción, coherencia |
| **Centradas en usuario** | Satisfacción, tasa de abandono, task completion, escalaciones |
| **Seguridad y cumplimiento** | Guardrail violations, model drift, actividad anómala, eventos DLP |

**Baselines se definen ANTES del deploy** — sin valores de referencia no se detecta degradación.

**Model drift** ocurre aunque la lógica del agente no cambie (el modelo subyacente se actualiza). Se detecta con **golden datasets** y evaluaciones programadas.

### Herramienta → problema

| Herramienta | Para qué |
|---|---|
| **Azure Monitor** (+ KQL en Log Analytics) | Telemetría principal: latencia, errores, alertas, diagnóstico profundo |
| M365 Admin Analytics | Adopción, engagement por departamento |
| Copilot & Agent Analytics | Invocaciones, finalización de tareas, guardrail events |
| Power Platform Admin Center | Salud de entorno, conectores, flows, impacto DLP |
| Foundry Dashboard | Agentes pro-code, ejecución del modelo, drift |
| Power BI custom | KPIs, heatmaps, reportes de cumplimiento |

### Diagnóstico → ajuste

| Problema | Causa típica | Ajuste |
|---|---|---|
| Respuestas incorrectas | Grounding desactualizado | Actualizar y re-indexar fuentes |
| Respuestas lentas | Flujos pesados, dependencias | Simplificar lógica, ajustar orquestación |
| Errores de acción | Conectores/permisos | Revisar conector, least privilege |
| Alto abandono | Instrucciones confusas | Mejorar UX y claridad del flujo |
| Guardrail violations | DLP mal configurada | Ajustar DLP, etiquetas, acciones permitidas |
| Model drift | Modelo subyacente cambió | Re-evaluar con golden dataset, actualizar prompts |

Modelo operativo de supervisión: roles definidos + flujos de incidentes + métricas estandarizadas con baselines + cadencias (diaria/semanal/mensual) + gestión de cambios versionada.

---

## 2. Testing de soluciones con IA

Las pruebas de agentes difieren del software tradicional: **outputs probabilísticos, no determinísticos** → marcos que contemplen variabilidad, ambigüedad y multi-turno.

### Los 4 tipos de prueba

| Tipo | Qué valida |
|---|---|
| **Basadas en escenarios** | Flujos reales, inputs ambiguos/incompletos, razonamiento multi-turno. **La más importante** |
| **Rendimiento y confiabilidad** | Volumen, interacciones largas, sesiones simultáneas, latencia bajo carga |
| **Seguridad y cumplimiento** | Datos sensibles, RBAC, triggers DLP, rechazo de instrucciones no permitidas |
| **Usabilidad** | Claridad de respuestas, fricción, accesibilidad |

### Criterios de validación = umbrales medibles

| Métrica | Umbral típico |
|---|---|
| Precisión | ≥ 90% |
| Tasa de éxito sin intervención | ≥ 85% |
| Tiempo de respuesta | P95 < 3s |
| Tasa de error | < 5% |
| Satisfacción | ≥ 4/5 |

Nunca "el agente responde bien" — siempre números. Cualitativas (coherencia, cobertura de dominio) se validan con revisión de SMEs y golden datasets.

### Blueprint de prueba unificado

**Un** plan estandarizado para todos los agentes de la organización (no uno por agente), gestionado por el CoE: ámbito + datos de prueba + roles + criterios de éxito + registro centralizado de resultados por versión.

### Copilot como generador de casos de prueba

Describir el flujo + criterios + guardrails → Copilot genera casos (happy path, edge cases, inputs maliciosos, fallos de integración). **Siempre validar con SMEs** — los casos generados son punto de partida, no producto final.

### Escenarios end-to-end multi-app D365

Cruzan límites de aplicación (Customer Service + Finance para reembolsos, Sales + SCM para stock). Documentar input, output esperado y criterios de aceptación; probar casos límite y fallos de integración.

Ciclo de testing en 8 pasos: planeación → diseño → ejecución → medición → análisis → optimización → re-test → **go/no-go**.

---

## 3. ALM para soluciones con IA

ALM de IA versiona **más que código**: datasets (entrenamiento y golden), knowledge sources, prompts e instrucciones, directivas/guardrails y telemetría de evaluación.

### Las 7 fases del ALM de datos (con puertas de control)

| Fase | Actividades | Puerta de salida |
|---|---|---|
| **A. Plan y catálogo** | Contratos de datos, etiquetas de sensibilidad, KPIs, riesgo | Contrato aprobado, activos catalogados |
| **B. Ingesta y preparación** | Calidad, linaje, versionado de esquema con hashes | Informe de calidad + linaje firmado |
| **C. Desarrollar y evaluar** | Entrenar con datos Dev/Test (**nunca producción**), baterías de evaluación | Umbrales cumplidos, riesgos mitigados |
| **D. Stage y aprobación** | Revisión privacidad/DLP/RAI, canary, **certificar golden dataset inmutable** | Aprobación del **CAB** + rollback validado |
| **E. Implementar y servir** | Publicar corpus/índices, **configurar residencia por región**, catálogo | — (pasa a operar) |
| **F. Operar y supervisar** | P95, costo/token, drift, reevaluaciones nightly, **circuit breaker** | Drift sobre límite → incidencia + pausa |
| **G. Evolución y retiro** | Reentrenar, retención/purga, conservar auditoría y linaje | Notificar consumidores, revocar acceso |

**Datos Red vs Gold**: Red = experimental, mutable, solo en Dev. Gold = finalizado, versionado, inmutable — lo único que toca producción.

### ALM por tipo de artefacto

| Artefacto | Herramientas | Consideración |
|---|---|---|
| Datos | Microsoft Purview, catálogo, Azure DevOps | Red→Gold, linaje obligatorio, residencia |
| Agentes Copilot Studio | Power Platform ALM, DevOps/GitHub | Entornos separados, no compartir identidades, promover como soluciones |
| Agentes Foundry | DevOps, GitHub Actions, Foundry SDK | CI/CD con evaluation gates |
| Modelos custom | Azure ML, MLflow, Model Registry | Nunca entrenar con datos de prod, golden dataset |
| D365 Finance/SCM | **LCS (Lifecycle Services)**, DevOps | Respetar límites de solución D365 |
| D365 Customer Service | Power Platform ALM, Solutions | OOB con config vs custom con código |

---

## 4. Seguridad, gobernanza, riesgos e IA Responsable

### Defensa en profundidad — 5 capas (ninguna alcanza sola)

1. **Identidad y acceso** — identidad administrada única por agente y entorno (sin secretos en código), RBAC de ámbito mínimo, separación de tareas (Maker → Publisher → Admin → Security), revisiones de acceso
2. **Gobernanza de datos** — DLP, etiquetas de confidencialidad, límites internos/públicos, residencia, retención con purga
3. **Evaluación y testing** — pipelines pre-producción, **red-teaming** (simulación adversarial), evaluaciones post-cambio
4. **Observabilidad y detección** — logs centralizados (prompts, tool calls, eventos de seguridad), alertas de anomalías
5. **Respuesta a incidentes** — runbook para deshabilitar/revertir, preservación de evidencia, simulacros

### Vulnerabilidades específicas de IA → mitigaciones

| Vulnerabilidad | Qué es | Mitigación |
|---|---|---|
| **Prompt injection** | Manipular el agente para salir de sus guardrails (override, contexto engañoso, instrucciones ocultas en documentos, multi-paso) | Filtrado entrada/salida, instrucciones robustas, red-teaming |
| Comportamiento no seguro del modelo | Alucinaciones, toxicidad, sesgo | Evaluaciones continuas, moderación de outputs |
| Exposición de datos | Datos sensibles en respuestas/logs/memoria | Minimización de datos, RBAC estricto |
| Brechas de identidad | Escalación de privilegios vía conectores | Identidades administradas, least privilege, segmentación |
| Uso incorrecto de herramientas | Acciones autónomas sin guardrails | Límites de herramientas, auditoría E2E, rollback |
| Flujos no seguros | Endpoints externos no aprobados | Allowlist de endpoints, MCP/A2A documentados |

### Pistas de auditoría — requisitos no negociables

**Inmutables**, timestamps verificables, **centralizadas en workspace separado del agente**, completas (quién/qué/cuándo/por qué), retención según regulación. Se auditan: cambios en prompts, knowledge sources, modelos, permisos, eventos de seguridad y decisiones de alto impacto.

### Revisión de IA Responsable (técnica, no filosófica)

| Principio | Control técnico verificable |
|---|---|
| Equidad | Tests de sesgo ejecutados |
| Confiabilidad | Validación bajo condiciones adversas |
| Privacidad | Datos protegidos, regulaciones locales cumplidas |
| Inclusión | Accesibilidad para todos los perfiles |
| Transparencia | El usuario sabe que es IA + el agente explica decisiones |
| Responsabilidad | Supervisión humana y escalación definidas |

### Checklist go/no-go antes de producción

Identidades administradas ✓ · RBAC mínimo ✓ · DLP + etiquetas ✓ · Residencia + retención ✓ · Logs centralizados + alertas ✓ · Red-teaming ejecutado ✓ · Entornos separados ✓ · Runbook de incidentes validado ✓

---

## Tips de examen — Sección 3

- **4 dimensiones de métricas**: escenario → dimensión. Herramienta → problema (lentitud = Azure Monitor; adopción = M365 Admin; drift = Foundry Dashboard).
- **Baselines antes del deploy**; model drift ocurre sin cambios en el agente.
- **4 tipos de prueba**: outputs probabilísticos = escenarios; carga = rendimiento; DLP/injection = seguridad.
- **Copilot genera casos, SMEs validan** — Copilot nunca es la autoridad final.
- **Blueprint de prueba único** para toda la org, no por agente.
- **Red vs Gold**: el golden dataset se certifica inmutable en fase D (Stage); el CAB aprueba D→E.
- **NUNCA entrenar con datos de producción** — opción automáticamente incorrecta.
- **Residencia de datos**: en ALM es fase E; en seguridad es control de cumplimiento — el examen pregunta desde ambos ángulos.
- **Prompt injection ≠ SQL injection**: técnicas y mitigaciones propias de IA.
- **Identidades administradas siempre** — nunca secretos en código.
- **Defensa en profundidad**: una solución con una sola capa (solo RBAC) es respuesta incorrecta.
- **Logs de auditoría inmutables y separados del agente** — si el agente puede modificarlos, no valen.

---

⬅️ [Sección 2 — Diseñar](./02-disenar-soluciones.md) · ➡️ [Cheat-sheet](./cheat-sheet.md)
