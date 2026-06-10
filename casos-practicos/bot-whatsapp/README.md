# 🎾 ClubBot WhatsApp — Asistente IA para Clubes Deportivos

> **Caso Práctico GH-600 · GitHub Agentic AI Engineer Certification**
> Este proyecto es material de estudio para la certificación GH-600. Demuestra los 6 bloques del examen
> a través de un proyecto real y completo: un asistente de WhatsApp con IA para clubes de pádel/tenis.

---

## ¿Qué es esto?

Un bot de WhatsApp que actúa como **asistente inteligente del club deportivo**. Los socios pueden escribir
por WhatsApp y consultar horarios, disponibilidad de canchas, precios, torneos y hacer reservas —
exactamente igual que hablar con un recepcionista, pero disponible 24/7 y sin tiempo de espera.

El sistema usa **Azure OpenAI GPT-4o** como cerebro, está orquestado por múltiples agentes especializados,
y toda la configuración, CI/CD y auditoría vive en GitHub.

---

## 📐 Las tres certificaciones en este proyecto

El mismo bot de WhatsApp es el hilo conductor de las tres certs. Cada una lo mira desde un ángulo diferente:

| Ángulo | Certificación | Pregunta central |
|---|---|---|
| 🏛️ Arquitectura | **AB-100** Agentic AI Business Solutions Architect | *¿Qué construimos y por qué?* |
| 🔧 Implementación | **AI-103** Azure AI Apps and Agents Developer | *¿Cómo lo construimos técnicamente?* |
| 🚀 Operación | **GH-600** GitHub Agentic AI Engineer | *¿Cómo lo operamos en producción?* |

### Guías de aplicación por certificación

| Certificación | Documento |
|---|---|
| GH-600 | [gh600-aplicacion-certificacion.md](./gh600-aplicacion-certificacion.md) |
| AI-103 | [ai103-aplicacion-certificacion.md](./ai103-aplicacion-certificacion.md) |

---

## 🗺️ Arquitectura Visual

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           FLUJO COMPLETO DEL SISTEMA                            │
└─────────────────────────────────────────────────────────────────────────────────┘

  📱 USUARIO (WhatsApp)
      │
      │  "Quiero reservar una cancha para mañana a las 18hs"
      ▼
  ┌──────────────┐        Webhook POST         ┌─────────────────────────────┐
  │  WhatsApp    │ ─────────────────────────►  │   Twilio Handler            │
  │  Business    │                             │   (twilio-handler.ts)       │
  └──────────────┘                             └────────────┬────────────────┘
  (Meta / Twilio)                                           │
                                                            │ Mensaje normalizado
                                                            ▼
                                               ┌─────────────────────────────┐
                                               │   ConversationAgent          │
                                               │   (orquestador principal)    │
                                               │                             │
                                               │  1. Recupera contexto        │
                                               │     ┌────────────────┐      │
                                               │     │   Redis Store  │      │
                                               │     │  (memoria por  │      │
                                               │     │   usuario)     │      │
                                               │     └────────────────┘      │
                                               │                             │
                                               │  2. Llama al modelo         │
                                               └────────────┬────────────────┘
                                                            │
                              ┌─────────────────────────────▼──────────────────────┐
                              │            AZURE AI FOUNDRY                         │
                              │                                                      │
                              │   ┌──────────────────────────────────────────────┐  │
                              │   │           Azure OpenAI GPT-4o                │  │
                              │   │                                              │  │
                              │   │  System Prompt + Historial + Tool Schemas   │  │
                              │   └──────────────────┬───────────────────────────┘  │
                              │                      │                               │
                              │           Content Safety · Monitoring · Eval         │
                              └──────────────────────┼────────────────────────────── ┘
                                                     │
                                    ┌────────────────┴────────────────┐
                                    │     GPT-4o decide qué agente    │
                                    │     especializado invocar        │
                                    └────────┬───────────┬────────────┘
                                             │           │
                          ┌──────────────────┘           └───────────────────┐
                          │                                                   │
               ┌──────────▼──────────┐                         ┌─────────────▼──────────┐
               │   ScheduleAgent     │                         │     MemberAgent         │
               │  Horarios y canchas │                         │  Membresías y socios    │
               └──────────┬──────────┘                         └─────────────┬───────────┘
                          │                                                   │
                          │                                                   │
               ┌──────────▼──────────┐              ┌────────────────────────▼────────┐
               │  MCP Calendar       │              │  MCP Member Server              │
               │  Server             │              │  (socios, membresías activas)   │
               │  (actividades,      │              └─────────────────────────────────┘
               │   canchas,          │
               │   disponibilidad)   │         ┌────────────────────────────────────────┐
               └─────────────────────┘         │           BookingAgent                 │
                                               │  ⚠️  ALTA SENSIBILIDAD — pagos         │
                                               │  Requiere confirmación humana          │
                                               │  antes de procesar reservas con pago   │
                                               └──────────────┬─────────────────────────┘
                                                              │
                                               ┌──────────────▼─────────────────────────┐
                                               │  MCP Booking Server                    │
                                               │  (reservas, disponibilidad real-time)  │
                                               └────────────────────────────────────────┘

  ─────────────────────────────────────────────────────────────────────────────────────

  🔧 CI/CD & OBSERVABILIDAD

  ┌───────────────────────────────┐       ┌──────────────────────────────────────────┐
  │       GitHub Repository       │       │          Azure Container Apps             │
  │                               │       │                                          │
  │  • copilot-instructions.md    │       │  ┌────────────────────────────────────┐  │
  │  • *.agent.md                 │       │  │  clubbot-service (Node.js/TS)      │  │
  │  • src/ (agentes, MCP, etc.)  │       │  └────────────────────────────────────┘  │
  │                               │       │                                          │
  │  ┌──────────────────────────┐ │       │  ┌────────────────────────────────────┐  │
  │  │  GitHub Actions          │ │       │  │  Application Insights              │  │
  │  │                          │ │       │  │  (trazas, errores, uso del modelo) │  │
  │  │  ci.yml  → tests + lint  │ │──────►│  └────────────────────────────────────┘  │
  │  │  deploy.yml → Azure ACA  │ │       │                                          │
  │  └──────────────────────────┘ │       │  ┌──────────┐  ┌────────────────────┐   │
  │                               │       │  │  Redis   │  │   PostgreSQL        │   │
  │  PR Audit Trail:              │       │  │ (memoria)│  │  (datos del club)   │   │
  │  cada cambio en agentes       │       │  └──────────┘  └────────────────────┘   │
  │  queda trazado en Git         │       └──────────────────────────────────────────┘
  └───────────────────────────────┘
```

---

## 🛠️ Stack Tecnológico

| Capa | Tecnología | Por qué |
|------|------------|---------|
| **Runtime** | Node.js 20 + TypeScript | Ecosistema maduro para bots, tipado fuerte para tool schemas |
| **Modelo IA** | Azure OpenAI GPT-4o (vía Azure AI Foundry) | El más capaz para conversación + function calling; Azure da control empresarial |
| **WhatsApp** | Twilio WhatsApp Business API | Abstrae la complejidad de la API de Meta; webhooks confiables |
| **Memoria** | Redis (Azure Cache for Redis) | Estado de conversación por usuario con TTL automático |
| **Base de datos** | PostgreSQL (Azure Database for PostgreSQL) | Datos del club: socios, canchas, precios, reservas |
| **Hosting** | Azure Container Apps | Escalado automático, serverless-friendly, fácil integración con Azure AI |
| **CI/CD** | GitHub Actions | Nativo en GitHub; trazabilidad completa en el repo |
| **Observabilidad** | Azure Application Insights | Trazas distribuidas, alertas, dashboard de uso del modelo |
| **Config agentes** | GitHub Copilot custom agents (`.agent.md`, `copilot-instructions.md`) | Los agentes de desarrollo viven en el mismo repo; reproducibles y versionados |
| **MCP** | MCP Servers (TypeScript SDK) | Protocolo estándar para exponer herramientas del club al modelo |

---

## ☁️ Por qué Azure AI Foundry

Azure AI Foundry es el **hub central** desde donde se gestiona todo lo relacionado con el modelo de IA.
No es solo un proveedor de API — es una plataforma completa de gestión del ciclo de vida de modelos.

| Capacidad | Beneficio para este proyecto |
|-----------|------------------------------|
| **Deploy de GPT-4o** | Un endpoint HTTPS con API key gestionada por Azure, sin servidor propio |
| **Gestión de endpoints** | Se puede cambiar la versión del modelo (GPT-4o → GPT-4.1) sin tocar el código |
| **Evaluaciones integradas** | Mide calidad de respuestas del bot antes de hacer deploy a producción |
| **Content Safety** | Filtrado automático de contenido inapropiado — crítico en un bot de WhatsApp público |
| **Monitoreo de costes** | Dashboard de tokens consumidos, coste por conversación, alertas de gasto |
| **Prompt management** | Versiona los system prompts de los agentes como artefactos gestionados |

> **Resumen**: Azure AI Foundry nos da lo que necesitamos para llevar un agente IA a producción de forma
> responsable: control, trazabilidad, seguridad y la capacidad de iterar sobre el modelo sin riesgo.

---

## 📁 Estructura del Repositorio

```
clubbot-whatsapp/
├── .github/
│   ├── copilot-instructions.md          ← instrucciones globales para el agente de desarrollo
│   ├── workflows/
│   │   ├── ci.yml                       ← tests + lint en cada PR (push a main bloqueado)
│   │   └── deploy.yml                   ← deploy a Azure Container Apps tras merge a main
│   └── CODEOWNERS                       ← booking.agent.ts requiere aprobación de tech lead
│
├── src/
│   ├── agents/
│   │   ├── conversation.agent.ts        ← orquestador: recibe mensaje, enruta al especialista
│   │   ├── schedule.agent.ts            ← consulta horarios, disponibilidad de canchas
│   │   ├── member.agent.ts              ← verifica membresías, datos del socio
│   │   └── booking.agent.ts             ← gestiona reservas ⚠️ alta sensibilidad (pagos)
│   │
│   ├── mcp/
│   │   ├── calendar-server.ts           ← MCP server: expone calendario del club como tools
│   │   ├── booking-server.ts            ← MCP server: sistema de reservas (requiere auth)
│   │   └── member-server.ts             ← MCP server: gestión de socios y membresías
│   │
│   ├── memory/
│   │   ├── redis-store.ts               ← persiste contexto de conversación por número de WhatsApp
│   │   └── memory-manager.ts            ← expiración (TTL), cleanup, resumen de historial largo
│   │
│   ├── webhooks/
│   │   └── twilio-handler.ts            ← recibe y valida webhooks de Twilio/WhatsApp
│   │
│   └── config/
│       └── agents/
│           ├── conversation.agent.md    ← system prompt del agente orquestador
│           ├── schedule.agent.md        ← system prompt del agente de horarios
│           └── booking.agent.md         ← system prompt del agente de reservas
│
├── .agents/
│   └── *.agent.md                       ← definiciones de custom agents para desarrollo (GH Copilot)
│
├── tests/
│   ├── agents/                          ← tests unitarios de cada agente
│   ├── mcp/                             ← tests de los MCP servers (mocks de APIs)
│   └── integration/                     ← tests end-to-end del flujo completo
│
└── docs/
    └── architecture.md                  ← decisiones de arquitectura (ADRs)
```

### Archivos clave explicados

| Archivo | Qué hace | Por qué importa para GH-600 |
|---------|----------|-----------------------------|
| `.github/copilot-instructions.md` | Define cómo Copilot debe ayudar en este repo | Sección 1: GitHub Copilot como asistente de desarrollo |
| `src/agents/*.ts` | Lógica de cada agente especializado | Sección 5: Multi-agent orchestration |
| `src/mcp/*.ts` | MCP servers que exponen datos del club | Sección 4: MCP y tool use |
| `src/config/agents/*.md` | System prompts de los agentes | Sección 2: Prompt engineering |
| `.github/workflows/ci.yml` | Pipeline de tests y lint | Sección 6: CI/CD y gobernanza |
| `booking.agent.ts` + CODEOWNERS | Agente de alto riesgo con revisión obligatoria | Sección 6: Human-in-the-loop |

---

## 💬 Escenario de Uso — Conversación de Ejemplo

Este flujo muestra cómo los agentes colaboran para responder una solicitud real:

```
┌─────────────────────────────────────────────────────────────┐
│  WhatsApp — Club Pádel Los Pinos                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  👤 María García                              18:03         │
│  "Hola! Quiero reservar una cancha de pádel  │             │
│   para mañana a las 18hs"                    ✓✓            │
│                                                             │
│                          🤖 ClubBot           18:03        │
│                   "¡Hola María! 👋 Déjame     │            │
│                    verificar tu membresía     │            │
│                    y la disponibilidad..."   ✓✓            │
│                                                             │
│  ─── [interno: MemberAgent verifica socio #4821] ───       │
│  ─── [interno: ScheduleAgent consulta disponibilidad] ───  │
│                                                             │
│                          🤖 ClubBot           18:03        │
│                   "✅ Tu membresía Premium    │            │
│                    está activa (vence Mar     │            │
│                    2026). Para mañana         │            │
│                    miércoles 15/01 a las      │            │
│                    18:00hs tengo disponible:  │            │
│                                               │            │
│                    🎾 Cancha 2 — Cubierta     │            │
│                    🎾 Cancha 4 — Semicubierta │            │
│                                               │            │
│                    ¿Cuál preferís?           ✓✓            │
│                                                             │
│  👤 María García                              18:04         │
│  "La cancha 2 cubierta, gracias"             ✓✓            │
│                                                             │
│  ─── [interno: BookingAgent prepara reserva] ───           │
│  ─── [interno: solicita confirmación antes de pagar] ───   │
│                                                             │
│                          🤖 ClubBot           18:04        │
│                   "Perfecto! Te confirmo la   │            │
│                    reserva:                   │            │
│                                               │            │
│                    📋 Cancha 2 Cubierta       │            │
│                    📅 Mié 15/01 · 18:00–19:00 │            │
│                    💰 $2.500 (incluido en     │            │
│                       membresía Premium)      │            │
│                                               │            │
│                    ¿Confirmás la reserva?    ✓✓            │
│                    Respondé SÍ o NO"          │            │
│                                                             │
│  👤 María García                              18:04         │
│  "Sí!"                                       ✓✓            │
│                                                             │
│  ─── [BookingAgent crea reserva en sistema] ───            │
│  ─── [envía confirmación + recordatorio programado] ───    │
│                                                             │
│                          🤖 ClubBot           18:04        │
│                   "✅ ¡Reserva confirmada!    │            │
│                    Te mandamos un mail con    │            │
│                    los detalles y recibirás   │            │
│                    un recordatorio 1 hora     │            │
│                    antes. ¡Hasta mañana! 🎾  ✓✓            │
└─────────────────────────────────────────────────────────────┘
```

### ¿Qué ocurrió internamente?

```
1. ConversationAgent recibe el mensaje → recupera contexto de Redis (María ya había escrito antes)
2. GPT-4o analiza la intención: "reservar cancha" → invoca MemberAgent
3. MemberAgent llama MCP Member Server → verifica membresía activa, obtiene ID de socio
4. ScheduleAgent llama MCP Calendar Server → consulta slots libres para mañana 18hs
5. GPT-4o genera respuesta con opciones → WhatsApp la muestra
6. María elige cancha → ConversationAgent vuelve a llamar a GPT-4o con la selección
7. BookingAgent prepara la reserva pero NO la confirma → pide confirmación explícita (human-in-the-loop)
8. María confirma → BookingAgent llama MCP Booking Server → reserva creada
9. Respuesta final + almacenamiento del estado en Redis
```

> **Nota GH-600**: El paso 7 es deliberado. `BookingAgent` es un agente de **alta sensibilidad** porque
> puede generar cargos económicos. Sigue el principio de *human-in-the-loop* requerido por la sección 5
> del examen para acciones con consecuencias irreversibles.

---

## 🧠 Cómo se Consume el Modelo de IA

Esta sección es clave para entender la integración con Azure AI Foundry.

### Flujo de llamada al modelo

```typescript
// src/agents/conversation.agent.ts (simplificado)

import { AzureOpenAI } from "openai";

const client = new AzureOpenAI({
  endpoint: process.env.AZURE_OPENAI_ENDPOINT,  // desde Azure AI Foundry
  apiKey: process.env.AZURE_OPENAI_API_KEY,
  apiVersion: "2024-10-21",
});

async function runConversationAgent(userMessage: string, userId: string) {
  // 1. Recuperar historial de Redis
  const history = await redisStore.getHistory(userId);

  // 2. Definir tools disponibles (MCP tools expuestas como function calling)
  const tools: ChatCompletionTool[] = [
    {
      type: "function",
      function: {
        name: "check_member_status",
        description: "Verifica si el número de WhatsApp corresponde a un socio activo",
        parameters: {
          type: "object",
          properties: {
            phone_number: { type: "string", description: "Número en formato E.164" }
          },
          required: ["phone_number"]
        }
      }
    },
    {
      type: "function",
      function: {
        name: "get_court_availability",
        description: "Consulta canchas disponibles para una fecha y hora",
        parameters: {
          type: "object",
          properties: {
            date: { type: "string", description: "Fecha en formato YYYY-MM-DD" },
            time_slot: { type: "string", description: "Hora en formato HH:MM" },
            sport: { type: "string", enum: ["padel", "tennis"] }
          },
          required: ["date", "time_slot"]
        }
      }
    },
    // ... más tools
  ];

  // 3. Llamar a GPT-4o con system prompt + historial + mensaje actual
  const response = await client.chat.completions.create({
    model: "gpt-4o",                    // nombre del deployment en Azure AI Foundry
    messages: [
      { role: "system", content: CONVERSATION_SYSTEM_PROMPT },
      ...history,
      { role: "user", content: userMessage }
    ],
    tools,
    tool_choice: "auto",                // el modelo decide si usar tools o responder directo
    temperature: 0.3,                   // bajo para respuestas consistentes y factuales
    max_tokens: 500,
  });

  // 4. Si el modelo quiere usar una tool, ejecutarla y volver a llamar al modelo
  if (response.choices[0].finish_reason === "tool_calls") {
    const toolResults = await executeToolCalls(response.choices[0].message.tool_calls);
    return runConversationAgent_withResults(history, userMessage, toolResults);
  }

  // 5. Guardar en Redis y devolver respuesta
  await redisStore.appendToHistory(userId, userMessage, response.choices[0].message.content);
  return response.choices[0].message.content;
}
```

### System Prompts por agente

Cada agente tiene su propio system prompt enfocado en su especialidad:

```markdown
<!-- src/config/agents/conversation.agent.md -->

Sos el asistente virtual del Club de Pádel Los Pinos. Tu nombre es ClubBot.
Sos amigable, conciso y siempre verificás que el socio esté habilitado antes
de realizar cualquier gestión.

REGLAS:
- Siempre saludá al inicio de cada nueva conversación
- Verificá la membresía antes de mostrar precios o disponibilidad
- Para reservas con costo, pedí confirmación explícita antes de proceder
- Respondé siempre en el idioma en que te escriben
- Si no sabés algo, decilo claramente — no inventes información del club

LÍMITES:
- No proceses pagos directamente — derivá al sistema de cobro del club
- No des información de otros socios
- Ante dudas sobre precios, consultá siempre la Pricing API (no uses datos hardcodeados)
```

### Por qué function calling y no RAG simple

| Enfoque | Cuándo usarlo |
|---------|--------------|
| **RAG** (documentos + embeddings) | Cuando la información es estática y textual (FAQs, reglamentos) |
| **Function Calling / Tool Use** | Cuando necesitás datos en tiempo real (disponibilidad, membresías activas) |
| **Combinación** | Este proyecto usa ambos: tools para datos dinámicos, RAG para preguntas sobre el reglamento del club |

> **Clave de diseño**: La disponibilidad de canchas cambia constantemente — no podés embedear eso.
> Por eso usamos function calling para que el modelo consulte la API en el momento de la pregunta.

---

## 🔗 Relación con los 6 Bloques de GH-600

| Bloque | Tema | Cómo se aplica en este proyecto |
|--------|------|----------------------------------|
| **1** | GitHub Copilot como asistente de desarrollo | `copilot-instructions.md` define el contexto del proyecto; `.agent.md` para tareas repetitivas |
| **2** | Prompt engineering y system prompts | Cada agente tiene su propio system prompt versionado en `src/config/agents/` |
| **3** | Integración de modelos IA en apps | El servicio Node.js consume GPT-4o via Azure AI Foundry con el SDK oficial |
| **4** | MCP y extensión de agentes con tools | MCP servers para Calendar, Booking y Members exponen APIs del club como tools estándar |
| **5** | Multi-agent orchestration | 4 agentes especializados con BookingAgent bajo human-in-the-loop por alto riesgo |
| **6** | CI/CD, gobernanza y seguridad | GitHub Actions + CODEOWNERS + audit trail en PRs + Content Safety en Azure AI Foundry |

---

## 🚀 Primeros Pasos (Setup)

```bash
# 1. Clonar e instalar dependencias
git clone https://github.com/tu-org/clubbot-whatsapp
cd clubbot-whatsapp
npm install

# 2. Configurar variables de entorno
cp .env.example .env
# Completar: AZURE_OPENAI_ENDPOINT, AZURE_OPENAI_API_KEY, TWILIO_AUTH_TOKEN, REDIS_URL, etc.

# 3. Ejecutar en desarrollo
npm run dev

# 4. Exponer webhook local (para recibir mensajes de Twilio en local)
npx ngrok http 3000
# Configurar la URL de ngrok en el dashboard de Twilio

# 5. Correr tests
npm test
```

---

## 📚 Documentación Relacionada

- [`docs/architecture.md`](./docs/architecture.md) — Decisiones de arquitectura (ADRs)
- [`src/config/agents/`](./src/config/agents/) — System prompts de cada agente
- [`gh600-aplicacion-certificacion.md`](./gh600-aplicacion-certificacion.md) — Guía de aplicación GH-600
- [`ai103-aplicacion-certificacion.md`](./ai103-aplicacion-certificacion.md) — Guía de aplicación AI-103
- [Azure AI Foundry Docs](https://learn.microsoft.com/azure/ai-foundry/) — Portal de gestión del modelo
- [Twilio WhatsApp API](https://www.twilio.com/docs/whatsapp) — Configuración del canal WhatsApp
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) — SDK para los MCP servers

---

<div align="center">

**GH-600 · GitHub Agentic AI Engineer** · Caso Práctico

*Este repositorio es material de estudio. El código es ilustrativo y puede requerir adaptaciones para producción.*

</div>
