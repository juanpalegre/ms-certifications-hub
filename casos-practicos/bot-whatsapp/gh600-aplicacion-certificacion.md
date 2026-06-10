# Cómo aplica GH-600 en el mundo real: El bot de WhatsApp para el club deportivo

> Explicación práctica de cada sección del examen aplicada a un proyecto real.
> Escrito en español coloquial, como si lo estuviéramos hablando en una reunión de trabajo.

---

## El proyecto en 30 segundos

Imaginá que sos socio de un club de pádel y tenis. Querés reservar una cancha para el sábado. Normalmente tendrías que llamar, esperar que atiendan, o meterte a una web que nadie sabe usar bien. Con este proyecto, simplemente le mandás un mensaje por WhatsApp al número del club: "Hola, ¿tienen canchas de pádel libres el sábado a las 18?", y el bot te responde en segundos, te muestra disponibilidad, te hace la reserva, y te manda la confirmación — todo sin hablar con nadie.

Por debajo, hay cuatro agentes de IA especializados corriendo en Azure Container Apps, un GPT-4o como cerebro principal (vía Azure AI Foundry), MCP servers que envuelven las APIs del club, Redis guardando el hilo de la conversación, y GitHub + GitHub Actions manejando todo el ciclo de desarrollo. Eso es lo que vamos a mapear a cada sección del examen GH-600.

---

## Sección 1: Arquitectura del agente y procesos SDLC (15–20%)

### Lo que evalúa el examen

Esta sección evalúa si entendés cómo integrar un agente de IA en un ciclo de desarrollo de software real: desde cómo planifica y ejecuta tareas hasta cómo se monitorea lo que hace.

---

### Cómo aplica en el bot del club

#### Integrar agentes en el SDLC

Acá el truco es que el agente de Copilot **no tiene acceso ilimitado** al repositorio. No puede tocar `main`, no puede hacer deployments, y no puede hablar con las APIs de producción directamente. Todo pasa por el mismo proceso que usaría cualquier desarrollador humano.

El flujo concreto que seguimos es:

1. Alguien del equipo abre un **issue en GitHub** describiendo lo que necesita. Por ejemplo: *"El bot debería poder responder sobre precios de inscripción a torneos"*.
2. El **Copilot coding agent** toma ese issue, genera un plan en lenguaje natural explicando qué va a hacer (qué archivos va a tocar, qué tests va a escribir, qué docs va a actualizar).
3. Después de que un humano **aprueba el plan**, el agente ejecuta y abre un **Pull Request** con todos los cambios.
4. Los **GitHub Actions** corren automáticamente: tests unitarios, lint, análisis de seguridad.
5. Un humano revisa el PR, lo aprueba, y recién ahí se hace merge a `main`.

El anti-patrón que explícitamente evitamos: nunca dejamos que el agente haga `push` directo a `main`. Nunca. Aunque parezca más rápido, perdés trazabilidad y control. El examen evalúa que entiendas esto.

---

#### Plan → Actuar → Evaluar en la práctica

Pensalo así: cuando implementamos el `BookingAgent` (el agente que crea reservas), el coding agent no arranca escribiendo código a ciegas. Primero genera un plan que dice algo como:

> *"Voy a crear `src/agents/booking-agent.ts`, definir el schema de la tool `createBooking` con los campos `courtId`, `memberId`, `dateTime`, `durationMinutes`, agregar validación de membresía activa antes de permitir la reserva, escribir tests en `tests/booking-agent.test.ts`, y actualizar el README con la descripción del agente."*

Ese plan queda visible en el **comentario del issue** antes de que el agente arranque a ejecutar. Si el plan tiene algo mal — por ejemplo, si se olvidó de validar que la cancha no esté ya reservada — un humano lo puede corregir **antes** de que el agente escriba una sola línea de código. Esto es el ciclo Plan → Actuar → Evaluar aplicado al SDLC.

En producción, el mismo ciclo aplica al bot en tiempo de ejecución: el ConversationAgent planifica qué agentes especialistas va a invocar antes de responder al usuario.

---

#### Observabilidad desde el inicio

Básicamente lo que hacemos acá es que cada decisión del agente queda **documentada automáticamente**:

- Los **mensajes de commit** tienen contexto real: no ponen "fix stuff", ponen "feat(booking): add membership validation before court reservation".
- Las **descripciones de PR** explican el razonamiento del agente: por qué eligió ese approach, qué alternativas consideró, qué tests cubren qué casos.
- Los **GitHub Actions logs** capturan cada paso de la CI: si un test falla, el log muestra exactamente qué herramienta llamó el agente y cuál fue el resultado.

Esto no es solo buena práctica — el examen pregunta específicamente sobre esto. La idea es que si algo sale mal en producción, podés rastrear hacia atrás hasta el PR, el plan, y el issue original.

---

## Sección 2: Herramientas y entorno (20–25%) ⚠️

### Lo que evalúa el examen

Esta es la sección con más peso del examen (25% en algunos blueprints). Evalúa que entendás cómo configurar las herramientas que usa el agente, cómo funcionan los MCP servers, y cómo se gestiona el entorno de ejecución de forma segura.

---

### Cómo aplica en el bot del club

#### Las herramientas del agente (tools / function calling)

El GPT-4o no puede hacer nada solo — necesita **herramientas** para interactuar con el mundo real. En nuestro caso, las herramientas son funciones que el modelo puede "llamar" (function calling) y que internamente hacen llamadas a las APIs del club.

Las herramientas que tenemos son:

| Tool | Qué hace | Permisos |
|------|----------|----------|
| `checkAvailability` | Verifica si una cancha está libre en un horario | Solo lectura |
| `getSchedule` | Devuelve el calendario de actividades del club | Solo lectura |
| `lookupMember` | Busca datos de un socio por número de WhatsApp | Solo lectura |
| `createBooking` | Crea una reserva en el sistema | **Escritura** |
| `getPricing` | Consulta precios de canchas y actividades | Solo lectura |

El truco acá es que **no todos los agentes tienen acceso a todas las herramientas**. El `ScheduleAgent` solo tiene `checkAvailability` y `getSchedule` — herramientas de lectura. El `BookingAgent` tiene `createBooking`, pero para poder usarla, primero necesita una confirmación explícita del usuario. Esto es **scoping de permisos** aplicado a función calling.

---

#### MCP Servers — la estrella del show

MCP (Model Context Protocol) es, básicamente, un estándar para que los agentes de IA descubran y usen herramientas de forma dinámica. En lugar de que hardcodees cada endpoint de la API del club en el código del agente, **envolvés cada API en un MCP server**, y el agente descubre qué puede hacer preguntándole al servidor.

Pensalo así: es como la diferencia entre darle a alguien una lista de teléfonos fijos vs darle acceso a la guía telefónica completa. Con MCP, el agente puede explorar qué herramientas están disponibles, cuáles son sus parámetros, y cuáles tiene permitido usar.

En nuestro proyecto tenemos tres MCP servers:

**`calendar-mcp-server`**
- Expone herramientas de lectura: disponibilidad de canchas, calendario de clases, reservas existentes.
- Nivel de riesgo: bajo. El agente puede usarlo sin aprobación adicional.
- Cualquier agente del sistema puede conectarse a él.

**`booking-mcp-server`**
- Expone herramientas de escritura: crear reservas, modificar turnos, cancelar con política de cargo.
- Nivel de riesgo: **alto**. Requiere aprobación explícita en el allowlist de la organización.
- Solo el `BookingAgent` tiene acceso, y solo después de que el usuario confirmó la acción.

**`member-mcp-server`**
- Expone datos de socios: estado de membresía, historial de reservas, datos de contacto.
- Nivel de riesgo: medio. Datos personales, logging obligatorio.
- Accesible por `MemberAgent` y `ConversationAgent` (para identificar al usuario).

**MCP Allowlists** — este concepto entra directo en el examen. La organización configura qué MCP servers están permitidos. El `booking-mcp-server` no está disponible por defecto para todos los agentes: hay que aprobarlo explícitamente. Si alguien intenta conectar un MCP server no autorizado, el sistema lo bloquea. Esto evita que un agente rogue empiece a crear reservas fantasma.

**MCP Logs** — cada llamada a una herramienta MCP queda registrada: qué agente la llamó, con qué parámetros, cuál fue la respuesta, y en qué timestamp. Esto es clave para auditoría.

---

#### El entorno de ejecución

Hay dos entornos que hay que entender bien, porque el examen los confunde:

**El entorno del coding agent (desarrollo):**
- Corre en una **VM sandboxeada** de GitHub.
- Tiene un firewall que le **impide llamar a las APIs de producción** directamente. No puede hablar con el `booking-mcp-server` de producción mientras desarrolla. Trabaja contra un ambiente de staging o mocks.
- Solo trabaja en **ramas de feature**, nunca en `main`.
- Puede ser invocado desde un GitHub Actions workflow: cuando se abre un issue con el label `copilot`, el workflow dispara el coding agent automáticamente.

**El entorno de producción (runtime):**
- El bot corre en **Azure Container Apps** — un servicio de contenedores serverless que escala automáticamente.
- Los mensajes de WhatsApp llegan como webhooks HTTP al Container App.
- El Container App llama a los MCP servers (que también corren como containers separados).
- Redis (Azure Cache for Redis) guarda el estado de la conversación.
- Las credenciales se manejan con **Azure Key Vault + Managed Identity** — ningún secreto está hardcodeado en el código.

---

#### Error handling en producción

El examen pregunta sobre resiliencia, y acá tenemos casos concretos:

**Retry logic para WhatsApp API**: si el envío de un mensaje falla (timeout, rate limit), el sistema reintenta hasta 3 veces con backoff exponencial. WhatsApp tiene rate limits que hay que respetar.

**Compensation pattern para reservas**: imaginá que el `BookingAgent` llama a `createBooking`, la reserva se crea en el sistema del club, pero después falla el envío del mensaje de confirmación al usuario. El usuario no sabe si quedó la reserva o no. Sin compensación, el socio llama al club confundido. Con el patrón de compensación, si el mensaje de confirmación falla, el sistema **cancela la reserva automáticamente** y notifica al staff para que llamen al socio. Siempre estado consistente.

**Escalación a humano**: si el bot no puede resolver algo después de 2 intentos, responde: *"Disculpá, no puedo completar esto automáticamente. Te conecto con un asesor del club."* Y deriva a un staff member vía WhatsApp Business. Nunca deja al socio colgado.

---

## Sección 3: Memoria, estado y ejecución (10–15%)

### Lo que evalúa el examen

Esta sección evalúa que entiendas cómo los agentes gestionan información a lo largo del tiempo: qué recuerdan entre mensajes, cómo manejan conversaciones largas, y qué pasa cuando el contexto se pone complicado.

---

### Cómo aplica en el bot del club

#### El problema del WhatsApp bot sin memoria

Este es el problema fundamental que cualquiera que haya hecho un chatbot sabe bien. WhatsApp no tiene concepto de "sesión" — cada mensaje llega como un **HTTP webhook independiente**. Sin gestión de estado, el bot sería completamente amnésico:

```
Socio: "Hola, soy Juan Pérez, quiero reservar una cancha"
Bot: "¡Hola! ¿En qué te puedo ayudar?"
Socio: "Para el sábado a las 18"
Bot: "¿Reservar qué cosa? No sé de qué hablás."
```

Eso es exactamente lo que pasa si mandás cada webhook al modelo sin contexto. Cada llamada al GPT-4o es independiente. El modelo no sabe lo que pasó antes.

---

#### Memoria de corto plazo: Redis

La solución es Redis. Básicamente lo que hacemos es:

1. Cuando llega un webhook de WhatsApp, leemos el número de teléfono del remitente.
2. Usamos ese número como **clave en Redis** (hasheada por privacidad).
3. En Redis guardamos el **historial de la conversación**: los últimos N turnos (mensaje del usuario + respuesta del bot).
4. Cuando construimos el prompt para GPT-4o, metemos ese historial como contexto.
5. Después de recibir la respuesta del modelo, actualizamos el historial en Redis.
6. El registro en Redis tiene un **TTL de 4 horas**. Si el socio no escribió nada en 4 horas, la próxima vez arranca conversación nueva.

El TTL de 4 horas no es arbitrario — viene de analizar los patrones de uso del club. La mayoría de las interacciones se completan en menos de 20 minutos. 4 horas cubre incluso los casos de "lo dejo para más tarde".

El historial no guarda mensajes infinitos — usamos una **ventana deslizante de 20 turnos** (10 del usuario + 10 del bot). Más que eso empieza a pesar demasiado en el prompt y el modelo se pierde.

---

#### Memoria de largo plazo: Copilot Memory + instrucciones personalizadas

Acá es donde entra algo específico de GH-600. El **Copilot coding agent** también tiene memoria — no para la conversación del bot, sino para el conocimiento del codebase.

En el archivo `.github/copilot-instructions.md` documentamos las convenciones del proyecto:

```markdown
# Convenciones del bot del club deportivo

## Reglas de negocio importantes
- Siempre verificar membresía activa ANTES de permitir una reserva
- Las reservas requieren confirmación explícita del usuario antes de crearse
- El BookingAgent nunca procesa pagos — siempre redirige al link de pago seguro

## Convenciones de código
- Todos los agentes extienden la clase BaseAgent
- Las tools deben tener tipos TypeScript estrictos con Zod schemas
- Los logs usan el correlation ID del webhook de WhatsApp
```

Esto le da al coding agent **conocimiento persistente** sobre cómo funciona el proyecto. Cuando el agente genera código nuevo, "recuerda" que debe validar membresía antes de reservar, incluso si el issue no lo menciona explícitamente.

La **Copilot Memory** (almacenamiento automático de hechos del codebase) complementa esto: el agente aprende que `BookingAgent` vive en `src/agents/booking-agent.ts`, que los schemas de Zod están en `src/schemas/`, y que los tests van en `tests/` con el mismo nombre que el archivo que testean.

---

#### Deriva de contexto

Este es un problema real que el examen toca. Imaginá esta conversación:

```
Socio: "¿Tienen clases de pádel los martes?"
Bot: "Sí, hay clases a las 19h y 21h con el Profe García"
Socio: "¿Y cuánto salen?"
Bot: "$3.500 la clase"
Socio: "Perfecto. ¿Y las piletas abren en verano?"
Bot: [responde sobre piletas]
Socio: "Buenísimo. Me apunto a la de pádel del martes a las 19"
Bot: ... ¿a cuál pádel del martes? ¿A las 19h? ¿Con qué profe?
```

En una conversación larga con cambios de tema, el modelo puede perder el hilo. Este es el **context drift** (deriva de contexto).

Nuestras soluciones:

1. **Ventana deslizante** de 20 turnos — los turnos más viejos se descartan, así el contexto reciente siempre está fresco.
2. **Resumen automático**: antes de descartar turnos viejos, el sistema le pide al GPT-4o que genere un resumen de los puntos clave mencionados (qué actividad, qué horario, qué precio). Ese resumen reemplaza los turnos viejos en el contexto.
3. **Reset explícito**: si el usuario escribe "empezar de nuevo" o "olvidate de lo anterior", el sistema limpia el contexto en Redis y arranca fresco.
4. **Detección de cambio de tema**: el ConversationAgent analiza si el nuevo mensaje es coherente con el contexto actual. Si detecta un cambio brusco, inserta una nota en el prompt: *"El usuario cambió de tema. El contexto anterior era sobre pádel; ahora pregunta sobre natación."*

---

## Sección 4: Evaluación, errores y ajuste (15–20%)

### Lo que evalúa el examen

Esta sección evalúa que sepas cómo medir si el agente está funcionando bien, cómo diagnosticar cuando algo falla, y cómo mejorar el sistema de forma iterativa.

---

### Cómo aplica en el bot del club

#### Definir "éxito" para el bot

Antes de hablar de métricas, hay que tener claro qué significa que el bot funcione bien. Para nosotros hay cinco criterios:

1. **Tasa de respuesta correcta**: ¿cuántas respuestas del bot son factualmente correctas sobre el club? (horarios reales, precios actualizados, disponibilidad precisa)
2. **Tasa de completitud de reservas**: de los usuarios que intentaron reservar, ¿cuántos lo completaron sin ayuda humana?
3. **Satisfacción del usuario**: los socios pueden reaccionar con 👍 o 👎 a las respuestas. Monitoreamos la proporción.
4. **Tasa de escalación**: ¿cuántas conversaciones terminan con *"te comunico con un asesor"*? Queremos esto bajo, pero no cero — si nunca escala, probablemente el bot esté respondiendo cosas que debería escalar.
5. **Latencia de respuesta**: tiempo desde que llega el webhook hasta que el socio recibe la respuesta. Target: menos de 3 segundos para consultas simples, menos de 8 para reservas.

---

#### Señales cuantitativas vs cualitativas

**Cuantitativas** (las que miden solas en Application Insights):
- % de reservas completadas sin error en el primer intento
- % de mensajes que requirieron intervención humana
- Tiempo promedio de respuesta por tipo de consulta
- Tasa de éxito de cada tool call (checkAvailability, createBooking, etc.)
- Cantidad de retries por fallo de API

**Cualitativas** (las que requieren ojos humanos):
- ¿Las respuestas suenan naturales en español, o tienen el tono de un manual de instrucciones?
- ¿El bot está dando información del club que ya no aplica? (ejemplo: precios de temporada pasada)
- ¿El bot recomienda actividades que tienen sentido para el perfil del socio?
- Revisión aleatoria por parte del staff: 5% de las conversaciones se revisan manualmente cada semana.

---

#### Analizar errores: herramientas concretas de GitHub

Cuando algo falla, hay una cadena de herramientas para diagnosticar:

**Escenario**: el `BookingAgent` falló al crear una reserva y el socio recibió un error.

1. **Application Insights**: buscamos el correlation ID del webhook de WhatsApp. Encontramos el trace completo: qué tools llamó el agente, en qué orden, cuál fue el response de cada una.

2. **GitHub Actions logs**: si el problema surgió después de un deploy reciente, revisamos el log del workflow de deploy. ¿Pasaron todos los tests? ¿Hubo algún warning en el análisis de dependencias?

3. **Timeline del PR**: ¿qué cambió en el `BookingAgent` en el último PR? El PR description (generado por el coding agent) explica qué modificó y por qué. Podemos ver si el cambio introdujo el bug.

4. **MCP Server logs**: revisamos los logs del `booking-mcp-server`. ¿Recibió la llamada? ¿La API del club respondió con error? ¿Fue un timeout?

Con estas cuatro fuentes, en general encontramos la causa raíz en menos de 15 minutos.

---

#### Causa raíz: las 4 categorías aplicadas al bot

El examen categoriza los errores en cuatro tipos. Veamos cada uno con ejemplos reales de nuestro bot:

**1. Error de razonamiento (Reasoning Error)**
El GPT-4o malinterpretó la consulta.

*Ejemplo*: El socio preguntó "¿puedo reservar la 4 para mañana?". "La 4" es la cancha número 4, pero el modelo entendió "las 4 de la tarde". Resultado: le mostró disponibilidad horaria en vez de preguntarle a qué hora quería jugar.

*Fix*: actualizar el system prompt del ConversationAgent para que, cuando el usuario mencione un número sin contexto claro, pregunte de desambiguación antes de continuar.

**2. Error de uso de herramientas (Tool Misuse)**
El agente llamó a las herramientas en el orden equivocado o con parámetros incorrectos.

*Ejemplo*: el `BookingAgent` llamó a `createBooking` antes de llamar a `checkAvailability`. La cancha ya estaba reservada, y el sistema del club devolvió un error de conflicto.

*Fix*: agregar una regla explícita en el prompt del BookingAgent: "SIEMPRE llamar a checkAvailability antes de createBooking. Nunca crear una reserva sin confirmar disponibilidad primero." También agregar un test que verifica este orden.

**3. Error de contexto (Context Error)**
El agente usó información desactualizada de la conversación.

*Ejemplo*: el socio primero preguntó por la cancha 2 los martes, después cambió de tema a precios de natación, y finalmente dijo "bueno, la reservo". El bot reservó... la cancha 2 del martes, cuando el socio en realidad quería hablar de la pileta.

*Fix*: implementar la detección de cambio de tema (descrita en Sección 3) y hacer que el bot siempre confirme el qué, cuándo y dónde antes de llamar a createBooking.

**4. Error de entorno (Environment Error)**
El problema no fue el agente — fue la infraestructura.

*Ejemplo*: la API del club tuvo un timeout de 30 segundos. El `booking-mcp-server` esperó, el Container App también esperó, y el socio recibió silencio radio.

*Fix*: implementar circuit breaker en el MCP server. Si la API del club no responde en 5 segundos, el MCP server responde con error inmediatamente y el bot dice: *"El sistema de reservas está lento en este momento. ¿Te aviso cuando esté disponible?"*. Además, retry con backoff para las llamadas al club.

---

#### Ajuste iterativo

El ciclo de mejora que usamos:

1. **Review semanal**: el equipo revisa las conversaciones donde el bot escaló a humano o donde recibió 👎.
2. **Identificar patrón**: ¿es siempre el mismo tipo de pregunta? ¿Siempre el mismo horario? ¿Siempre el mismo agente?
3. **Actualizar instrucciones**: si es un problema de razonamiento, se actualiza el system prompt o el `.github/copilot-instructions.md`. Si es un bug en el código, se abre un issue.
4. **Coding agent genera el fix**: el coding agent lee el issue, genera el plan, lo ejecuta, abre el PR.
5. **CI valida**: los tests corren, incluyendo tests de regresión para los casos que fallaron.
6. **Deploy via GitHub Actions**: si los tests pasan y el PR es aprobado, el deploy es automático.
7. **Monitor**: Application Insights muestra si la tasa de ese tipo de error bajó.

La clave es que el ciclo es corto: de "detectamos el problema" a "el fix está en producción" puede ser cuestión de horas.

---

## Sección 5: Orquestación multiagente (15–20%) ⚠️

### Lo que evalúa el examen

Esta sección evalúa que entiendas cómo coordinar múltiples agentes especializados, cómo evitar conflictos entre ellos, y cómo mantener visibilidad sobre lo que hace cada uno.

---

### Cómo aplica en el bot del club

#### Los 4 agentes del sistema

No armamos un agente gigante que haga todo. Armamos cuatro agentes especializados, cada uno experto en su área. Pensalo como un equipo de recepcionistas del club, cada uno especializado en algo diferente.

**ConversationAgent** — *el recepcionista principal*
- Recibe **todos** los mensajes de WhatsApp.
- No conoce los detalles de canchas ni membresías — su único trabajo es entender qué quiere el socio y derivar al especialista correcto.
- Clasifica la intención: ¿es una consulta de horarios? ¿De precios? ¿Quiere reservar? ¿Es una queja?
- Es el **orchestrator** del sistema.
- Herramientas que tiene: `lookupMember` (para identificar al socio), `classifyIntent` (interna).

**ScheduleAgent** — *el especialista en canchas y horarios*
- Solo sabe de disponibilidad de canchas, calendarios de clases, actividades del club.
- Herramientas: `checkAvailability`, `getSchedule`.
- **Solo lectura** — nunca puede modificar nada.
- Responde preguntas como: *"¿Hay cancha de pádel el jueves a las 20?"*, *"¿A qué hora son las clases de yoga?"*.

**MemberAgent** — *el especialista en socios*
- Maneja todo lo relacionado con la membresía: estado, vencimiento, categoría, historial.
- Herramientas: `lookupMember`, `getMembershipStatus`, `getBookingHistory`.
- Lectura principalmente, con logging obligatorio (datos personales).
- Responde preguntas como: *"¿Mi membresía sigue activa?"*, *"¿Cuántas reservas hice este mes?"*.

**BookingAgent** — *el especialista en reservas (alto riesgo)*
- Es el único que puede crear o modificar reservas.
- Herramientas: `checkAvailability` (lectura), `createBooking` (escritura), `cancelBooking` (escritura con cargo).
- **Alto riesgo** — siempre requiere confirmación explícita del usuario antes de ejecutar una acción de escritura.
- Nunca procesa pagos — genera un link seguro de pago y se lo manda al socio.

---

#### El patrón de orquestación: Orchestrator + Specialists

El `ConversationAgent` es el cerebro que coordina. El flujo para una reserva es:

```
Socio: "Quiero reservar cancha de pádel el sábado a las 18"

1. ConversationAgent recibe el mensaje
2. Llama a lookupMember → identifica al socio (Juan Pérez, membresía activa)
3. Clasifica intención: BOOKING_REQUEST
4. Llama al ScheduleAgent con: {sport: "padel", day: "saturday", time: "18:00"}
5. ScheduleAgent responde: {available: true, courts: ["cancha_1", "cancha_3"]}
6. ConversationAgent muestra opciones al socio
7. Socio elige: "La cancha 1"
8. ConversationAgent llama al BookingAgent con los datos
9. BookingAgent muestra resumen y pide confirmación
10. Socio confirma
11. BookingAgent llama a createBooking
12. Socio recibe confirmación + link de pago
```

¿Por qué no un solo agente gigante? Porque:
- Más fácil de testear: podés testear el ScheduleAgent solo, sin tener que simular una conversación completa.
- Más fácil de mantener: si cambian los horarios del club, solo tocás el ScheduleAgent.
- Más seguro: el principio de mínimo privilegio aplicado a agentes. El ScheduleAgent literalmente no puede reservar nada aunque quiera.
- Más fácil de entender qué falló: si algo sale mal, sabés en qué agente buscar.

---

#### Aislamiento para ejecución paralela

El `ScheduleAgent` y el `MemberAgent` pueden correr **en paralelo** porque no comparten estado ni se afectan entre sí. Si el socio pregunta algo que requiere saber tanto su disponibilidad como su estado de membresía, el ConversationAgent puede llamar a ambos al mismo tiempo y esperar los dos resultados.

El `BookingAgent`, en cambio, **siempre corre solo** — nunca en paralelo con otra instancia del mismo agente. Esto es para evitar el problema de doble reserva: si dos mensajes del mismo socio llegan casi simultáneamente (puede pasar si el socio es ansioso y manda el mismo mensaje dos veces), no queremos que dos instancias del BookingAgent intenten crear dos reservas al mismo tiempo.

En el ciclo de desarrollo pasa algo análogo: en GitHub, cada agente tiene su **propia rama de feature** y su **propio PR**. El coding agent que trabaja en el BookingAgent no toca la rama del ScheduleAgent. Aislamiento total.

---

#### Conflictos entre agentes

Este es el problema más práctico de orquestación. Mirá este escenario:

```
Sábado 17:59:45 — ScheduleAgent verifica: cancha 3 disponible a las 18:00
Sábado 17:59:47 — Otro socio reserva cancha 3 a las 18:00 (desde la web del club)
Sábado 17:59:48 — BookingAgent intenta createBooking para nuestra cancha 3 a las 18:00
Sábado 17:59:48 — La API del club devuelve: CONFLICT - court already booked
```

En 3 segundos, la cancha pasó de disponible a ocupada. Esto es una **race condition** perfectamente normal en cualquier sistema concurrente.

Cómo lo manejamos:

1. El `booking-mcp-server` maneja el error CONFLICT de la API y lo convierte en un resultado estructurado: `{success: false, reason: "conflict", alternativeSlots: [...]}`.
2. El `BookingAgent` recibe este resultado y NO crashea. Le responde al socio: *"Uy, justo alguien reservó la cancha 3 en ese horario. Hay disponibilidad en la cancha 1 a las 18:00 o en la cancha 3 a las 19:00. ¿Cuál preferís?"*
3. El socio elige y el proceso sigue.

Esto es **optimistic locking** del lado del dominio: asumimos que no habrá conflicto, intentamos la operación, y si hay conflicto, recuperamos gracefully.

---

#### Observabilidad multi-agente

Con cuatro agentes corriendo, el diagnóstico puede volverse un infierno si no está bien instrumentado. La solución es el **correlation ID**.

Cuando llega un webhook de WhatsApp, generamos un UUID único para esa interacción. **Todos** los logs de todos los agentes que participan en responder ese mensaje incluyen ese ID. Entonces cuando queremos entender qué pasó con el mensaje del sábado a las 17:59, hacemos una sola query en Application Insights:

```kusto
traces
| where customDimensions.correlationId == "abc-123-def"
| order by timestamp asc
```

Y vemos el flujo completo: ConversationAgent recibió → clasificó como BOOKING → llamó a ScheduleAgent → ScheduleAgent verificó disponibilidad → ConversationAgent llamó a BookingAgent → BookingAgent intentó reservar → CONFLICT → BookingAgent ofreció alternativas.

En el SDLC, lo mismo aplica: cada PR del coding agent tiene en su descripción qué agente modificó, qué herramientas cambiaron, y qué tests nuevos se agregaron. Si el sábado hay un problema con el ScheduleAgent, podemos buscar en GitHub todos los PRs que tocaron ese archivo en los últimos 7 días.

---

## Sección 6: Protección y responsabilidad (10–15%)

### Lo que evalúa el examen

Esta sección evalúa que sepas implementar salvaguardas apropiadas para diferentes niveles de riesgo, y que entiendas cómo mantener responsabilidad (accountability) sobre lo que hace el agente.

---

### Cómo aplica en el bot del club

#### Niveles de autonomía por tipo de acción

No todas las acciones del bot son iguales. Acá la clave es **proporcionalidad**: el nivel de control humano debe ser proporcional al riesgo de la acción.

| Acción | Riesgo | Nivel de autonomía |
|--------|--------|-------------------|
| Consultar horarios de canchas | Bajo | Totalmente autónomo |
| Consultar precios | Bajo | Totalmente autónomo |
| Consultar actividades del club | Bajo | Totalmente autónomo |
| Consultar estado de membresía | Medio | Autónomo con logging obligatorio |
| Consultar historial de reservas | Medio | Autónomo con logging obligatorio |
| Reservar cancha | Alto | Semi-autónomo — requiere confirmación del usuario |
| Cancelar reserva sin cargo | Alto | Semi-autónomo — requiere confirmación del usuario |
| Cancelar reserva con cargo económico | Muy alto | Requiere confirmación humana del staff |
| Modificar datos de membresía | Muy alto | Redirige a canal de atención humana |
| Procesar pagos | Crítico | Nunca — siempre link de pago externo seguro |

La lógica detrás: si algo puede costarle plata al socio o modificar datos permanentes, **un humano tiene que estar en el loop**. Si solo está dando información, que sea autónomo — frenar eso solo frustra al socio y no agrega seguridad real.

---

#### Guardrails concretos en el código

Estos no son conceptos abstractos — son configuraciones reales en el repositorio:

**Branch protection rules en `main`:**
- Requiere mínimo **2 revisiones humanas** aprobadas antes de hacer merge.
- CI debe pasar al 100%: tests, lint, análisis de seguridad.
- El historial de commits no puede ser reescrito (no force push).
- El coding agent de Copilot **nunca puede hacer push directo a `main`** — siempre trabaja en ramas.

**CODEOWNERS:**
```
# Toda la lógica de booking requiere aprobación del lead developer
src/agents/booking-agent.ts @lead-dev @tech-lead
src/schemas/booking-schema.ts @lead-dev

# Los MCP servers requieren aprobación de security
src/mcp-servers/ @lead-dev @security-team
```

Esto significa que si el coding agent modifica el BookingAgent, el PR no puede ser aprobado por cualquier persona del equipo — necesita explícitamente el OK del `@lead-dev`.

**MCP Allowlists a nivel organización:**
La organización de GitHub tiene configurado qué MCP servers pueden usar los repositorios. El `booking-mcp-server` está en una lista de aprobación explícita. Si alguien intenta agregar un MCP server nuevo sin que esté en la allowlist, los GitHub Actions lo detectan y bloquean el merge.

**Firewall del coding agent:**
El coding agent de Copilot tiene bloqueado el acceso a las APIs de producción. No puede llamar al endpoint de WhatsApp Business, no puede llamar a la API del club con credenciales de producción. Solo puede usar el ambiente de staging.

**Scoped tokens:**
Cada servicio tiene exactamente los permisos mínimos que necesita. El token del `calendar-mcp-server` solo puede hacer GET a la API del club. El del `booking-mcp-server` puede hacer GET y POST, pero no DELETE. Las credenciales viven en Azure Key Vault y se inyectan vía Managed Identity — nunca están en el código ni en variables de entorno en texto plano.

---

#### Human-in-the-loop sin frenar la velocidad

El error más común al implementar HITL (human-in-the-loop) es pasarse de conservador y meter un humano en el medio de acciones que no lo necesitan. Eso mata la experiencia del usuario.

Nuestro approach: el humano no tiene que aprobar la **acción** — el humano es el **usuario mismo**.

Para reservar una cancha, el flujo es:
1. Bot verifica disponibilidad (autónomo).
2. Bot presenta opciones (autónomo).
3. Usuario elige (humano en el loop de forma natural — es el usuario tomando la decisión).
4. Bot pide confirmación explícita: *"Perfecto. Te confirmo: cancha 1 el sábado 15/03 a las 18:00. ¿Confirmamos?"*.
5. Usuario dice "sí" (humano en el loop de nuevo).
6. Bot crea la reserva (autónomo — ya hubo confirmación humana).

No hay un empleado del club en el medio. El "human-in-the-loop" es el propio socio tomando decisiones conscientes. El empleado solo entra cuando hay acciones de muy alto riesgo (cargos económicos, modificación de membresías).

Esto cumple con los requisitos del examen sin sacrificar la experiencia: el bot sigue siendo rápido y útil.

---

#### Responsabilidad y auditoría

Si algo sale mal — y eventualmente siempre sale algo mal — necesitamos poder responder: ¿qué pasó, cuándo, quién lo hizo, y qué sistema lo ejecutó?

Cada acción de escritura del bot queda registrada en Application Insights con:
- **WhatsApp User ID** (hasheado para privacidad, pero reversible con la clave correcta).
- **Timestamp** exacto del webhook y de cada acción posterior.
- **Qué agente** ejecutó la acción (BookingAgent, MemberAgent, etc.).
- **Qué tool MCP** se llamó y con qué parámetros exactos.
- **Qué respondió la API** del club.
- **Correlation ID** que une todos los eventos de esa interacción.

En GitHub, el historial es inmutable. Cada línea de código que escribió el coding agent tiene su commit, su PR, su aprobación, su timestamp. Si mañana hay una reserva errónea y el socio dice que nunca confirmó nada, podemos buscar el correlation ID en Application Insights y ver exactamente qué mensaje mandó, cuándo, y qué confirmación dio.

Esto no es solo para cumplir con el examen — es lo que te salva cuando hay un reclamo real. La auditoría completa, combinada con el historial de GitHub, significa que siempre podés reconstruir la historia completa de cualquier acción que tomó el sistema.

---

## Resumen ejecutivo: ¿Cómo estudiar esto para el examen?

Si tuvieras que quedarte con una sola idea de cada sección:

| Sección | La idea clave |
|---------|---------------|
| **SDLC** | El agente sigue el mismo proceso que un humano: issue → plan → PR → CI → review → merge. Nunca saltea pasos. |
| **Herramientas y entorno** | MCP servers son la interfaz entre el agente y el mundo. Allowlists controlan qué puede usar. El entorno está aislado. |
| **Memoria y estado** | Redis guarda el contexto de corto plazo. Las instrucciones personalizadas dan memoria de largo plazo al coding agent. |
| **Evaluación** | Definí métricas antes de deployar. Clasifica los errores en 4 categorías. El ciclo de mejora es continuo. |
| **Multi-agente** | Un orchestrator coordina especialistas con mínimo privilegio. Correlation IDs hacen auditable todo el flujo. |
| **Protección** | Autonomía proporcional al riesgo. Guardrails en el código (CODEOWNERS, allowlists, tokens). Auditoría completa. |

---

*Documento generado para preparación de certificación GH-600 — Sección de casos prácticos.*
*Proyecto: Bot de WhatsApp para club deportivo (pádel/tenis)*
*Stack: Azure OpenAI GPT-4o · Azure AI Foundry · Azure Container Apps · Redis · MCP Servers · GitHub Actions*
