# Pasos de implementación — Bot WhatsApp para club deportivo

> Guía completa de implementación alineada a las secciones del examen GH-600.
> **Stack:** Azure OpenAI GPT-4o · Twilio WhatsApp Business API · Node.js/TypeScript · Redis · PostgreSQL · Azure Container Apps · GitHub Actions

---

## Fase 0: Preparación del entorno (antes de escribir una línea de código)

> **GH-600 §S1 — SDLC Integration & §S6 — Guardrails setup**

Esta fase establece GitHub como sistema de registro central y configura el entorno de IA antes de tocar código. El orden importa: primero las reglas, después la ejecución.

---

### Paso 0.1: Configurar GitHub como sistema de registro

```bash
# Crear el repositorio
gh repo create clubbot-whatsapp \
  --private \
  --description "Bot de WhatsApp con IA para club de pádel/tenis" \
  --clone

cd clubbot-whatsapp
```

**Crear estructura inicial:**

```bash
mkdir -p .github .agents src/{agents,mcp,webhooks,memory} tests
```

**Configurar branch protection en `main`:**

```bash
gh api repos/{owner}/clubbot-whatsapp/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["ci / lint","ci / type-check","ci / unit-tests"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null
```

**Crear `CODEOWNERS`:**

```
# .github/CODEOWNERS

# Regla general
*                   @lead-dev

# BookingAgent requiere 2 revisores por su sensibilidad
src/agents/booking* @lead-dev @senior-dev
src/mcp/booking*    @lead-dev @senior-dev
```

**Crear `.github/copilot-instructions.md`:**

```markdown
# ClubBot WhatsApp — Copilot Instructions

## Arquitectura
Este proyecto es un bot de WhatsApp para clubes deportivos (pádel/tenis).
Usa Azure OpenAI GPT-4o para IA (vía Azure AI Foundry) y Twilio para WhatsApp.

## Stack
- Runtime: Node.js 20 + TypeScript (strict: true)
- IA: @azure/openai SDK → deployment "clubbot-gpt4o"
- Cache/contexto: Redis (TTL 4 horas por conversación)
- DB: PostgreSQL (Prisma como ORM)
- MCP Servers: calendar-mcp, member-mcp, booking-mcp

## Estructura de carpetas
- src/agents/       → ConversationAgent, ScheduleAgent, MemberAgent, BookingAgent
- src/mcp/          → calendar-server.ts, member-server.ts, booking-server.ts
- src/webhooks/     → twilio-handler.ts
- src/memory/       → redis-store.ts
- tests/            → espejo de src/

## Convenciones de código
- TypeScript estricto (strict: true, noImplicitAny: true)
- Tests con Jest, coverage mínimo 80%
- Cada agente en src/agents/, cada MCP server en src/mcp/
- Variables de entorno siempre desde process.env — NUNCA valores hardcoded
- Errores tipados: nunca `catch(e: any)`, siempre `catch(e: unknown)`

## Reglas de negocio críticas
- BookingAgent NUNCA debe ejecutar createBooking sin confirmación explícita del usuario
- La confirmación debe ser "Sí" o "Si" (case-insensitive) — no inferida
- Los MCP servers de escritura (booking) requieren logging obligatorio de cada operación
- El contexto de conversación por usuario expira a las 4 horas (sliding window)
- Máximo 20 mensajes en el contexto activo (descartar los más antiguos)

## Escenarios de escalada
- Si el agente no puede resolver en 3 intentos → escalar a humano
- Si el usuario escribe "hablar con persona" o "asesor" → escalar inmediatamente
```

---

### Paso 0.2: Crear agentes de desarrollo personalizados

Los archivos `.agent.md` definen cómo Copilot debe comportarse al trabajar en partes específicas del sistema.

**Crear `.agents/booking-specialist.agent.md`:**

```markdown
---
name: Booking Specialist Agent
description: Agente especializado en el módulo de reservas. Aplica máxima precaución en cambios de lógica de negocio.
---

Sos un especialista en sistemas de reservas críticas. Cuando desarrolles cambios en BookingAgent o booking-mcp-server, seguí estas reglas sin excepción:

1. **TDD obligatorio**: siempre escribí el test antes que el código de producción.
2. **Mocks en tests**: nunca ejecutes createBooking en tests sin mockear completamente la API externa.
3. **Comentario en lógica de confirmación**: todo cambio en el flujo de confirmación requiere un comentario JSDoc explicando el "por qué".
4. **Errores visibles**: los errores deben escalar (throw o log + notificación), nunca silenciarse con try/catch vacío.
5. **Revisión de edge cases**: para cada función, listá mínimo 3 casos borde antes de implementar.
6. **Idempotencia**: createBooking debe ser idempotente — verificá duplicados antes de insertar.

Antes de cualquier cambio, preguntate: "¿Qué pasa si esto falla a mitad de camino?"
```

**Crear `.agents/backend-agent.agent.md`:**

```markdown
---
name: Backend Agent
description: Agente para desarrollo general de backend. Enfocado en TypeScript idiomático y buenas prácticas de API.
---

Sos un desarrollador backend senior especializado en Node.js/TypeScript. Al trabajar en este proyecto:

1. Preferí tipos explícitos sobre `any` o `unknown` cuando sea posible.
2. Usá async/await consistentemente — no mezcles con `.then()`.
3. Toda función que acceda a la DB o Redis debe tener manejo de errores.
4. Los endpoints de webhook deben responder en < 200ms (Twilio timeout = 15s).
5. Para Redis: siempre usá TTL — nunca keys sin expiración.
```

---

### Paso 0.3: Configurar Azure AI Foundry

```bash
# 1. Crear Azure AI Foundry project (vía Azure Portal o CLI)
az cognitiveservices account create \
  --name clubbot-ai \
  --resource-group rg-clubbot \
  --kind OpenAI \
  --sku S0 \
  --location eastus

# 2. Deployar modelo gpt-4o
az cognitiveservices account deployment create \
  --name clubbot-ai \
  --resource-group rg-clubbot \
  --deployment-name clubbot-gpt4o \
  --model-name gpt-4o \
  --model-version "2024-08-06" \
  --model-format OpenAI \
  --sku-capacity 10 \
  --sku-name Standard

# 3. Obtener endpoint y key
az cognitiveservices account show \
  --name clubbot-ai \
  --resource-group rg-clubbot \
  --query properties.endpoint

az cognitiveservices account keys list \
  --name clubbot-ai \
  --resource-group rg-clubbot
```

**Guardar en GitHub Secrets:**

```bash
gh secret set AZURE_OPENAI_ENDPOINT --body "https://clubbot-ai.openai.azure.com/"
gh secret set AZURE_OPENAI_KEY      --body "<tu-api-key>"
gh secret set AZURE_OPENAI_DEPLOYMENT --body "clubbot-gpt4o"
```

> **Nota GH-600:** Habilitar Content Safety filter en Azure AI Foundry Studio → tu proyecto → Content Filters → Create. Configurar severidad "Medium" para hate, violence, self-harm, sexual en input y output.

---

### Paso 0.4: Configurar Twilio WhatsApp Sandbox

1. Crear cuenta en [twilio.com](https://twilio.com)
2. Ir a **Messaging → Try it out → Send a WhatsApp message**
3. Activar Sandbox siguiendo las instrucciones
4. En **Sandbox Settings**, configurar:
   - **When a message comes in:** `https://<tu-dominio>/webhook/whatsapp`
   - Para desarrollo local usar ngrok: `ngrok http 3000`

```bash
gh secret set TWILIO_ACCOUNT_SID  --body "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
gh secret set TWILIO_AUTH_TOKEN   --body "<tu-auth-token>"
gh secret set TWILIO_PHONE_NUMBER --body "whatsapp:+14155238886"
```

---

## Fase 1: Implementar el núcleo del bot (ConversationAgent + webhook)

> **GH-600 §S1 — Plan → Act → Evaluate + §S3 — Memoria**

---

### Paso 1.1: Crear issue + asignar al Copilot coding agent

Crear el issue en GitHub con template estructurado para que el agente tenga contexto suficiente:

```
Título: feat: implementar webhook handler y ConversationAgent base

## Descripción
Crear el handler de webhooks de Twilio y el ConversationAgent que recibe mensajes,
carga contexto desde Redis y llama a Azure OpenAI.

## Criterios de éxito
- [ ] Recibe mensajes de WhatsApp vía POST /webhook/whatsapp
- [ ] Valida firma de Twilio (X-Twilio-Signature)
- [ ] Contexto de conversación persiste en Redis con TTL 4 horas
- [ ] Devuelve respuesta TwiML válida
- [ ] Tests unitarios cubren el flujo principal (mock de OpenAI y Redis)
- [ ] CI pasa (lint + type-check + tests)

## Archivos a crear
- src/webhooks/twilio-handler.ts
- src/memory/redis-store.ts
- src/agents/conversation.agent.ts
- tests/conversation.agent.test.ts

## Notas para el agente
- Usar el SDK @azure/openai para llamar a GPT-4o
- El endpoint y API key están en process.env.AZURE_OPENAI_ENDPOINT y AZURE_OPENAI_KEY
- NO usar valores hardcoded bajo ninguna circunstancia
- El sistema prompt se carga desde src/prompts/system.ts
- Validar firma de Twilio antes de procesar cualquier mensaje
```

**Asignar al Copilot coding agent:**

```bash
# Vía GitHub CLI (una vez creado el issue)
gh issue edit <número> --add-assignee "Copilot"

# O desde la UI: Issues → issue → Assignees → @Copilot
```

---

### Paso 1.2: El coding agent genera el plan

El agente analiza el issue y propone un plan antes de ejecutar. Ejemplo de plan típico:

```
## Plan generado por Copilot coding agent

Analicé el issue y el repositorio. Propongo el siguiente plan:

### Archivos nuevos
1. `src/webhooks/twilio-handler.ts`
   - Middleware Express que valida X-Twilio-Signature
   - Extrae `Body` y `From` del payload de Twilio
   - Llama a ConversationAgent y responde con TwiML

2. `src/memory/redis-store.ts`
   - `getContext(phone: string): Message[]` → recupera historial
   - `saveContext(phone: string, messages: Message[]): void` → guarda con TTL 4h
   - Key pattern: `conversation:{phoneNumber}`

3. `src/agents/conversation.agent.ts`
   - Carga contexto desde Redis
   - Agrega mensaje del usuario al contexto
   - Llama a Azure OpenAI con el sistema prompt + contexto
   - Guarda respuesta en Redis
   - Retorna texto de respuesta

4. `tests/conversation.agent.test.ts`
   - Mock de Redis y Azure OpenAI SDK
   - Test: flujo feliz (usuario envía mensaje → respuesta)
   - Test: contexto vacío (primera conversación)
   - Test: contexto con historial previo

### Dependencias a instalar
- @azure/openai
- ioredis
- twilio
- express

¿Procedo con este plan?
```

---

### Paso 1.3: Revisar y aprobar el plan

> **Concepto GH-600: Human-in-the-Loop**

El desarrollador revisa el plan **antes** de que el agente ejecute. Esto es fundamental: el agente actúa como colaborador, no como autónomo sin supervisión.

Checklist de revisión del plan:
- [ ] Los archivos propuestos coinciden con la arquitectura definida
- [ ] El agente no propone hardcodear valores sensibles
- [ ] Los tests mencionados son suficientes para el criterio de aceptación
- [ ] No hay dependencias inesperadas o innecesarias

Si el plan está bien → aprobar ("LGTM, proceed").
Si hay algo incorrecto → dar feedback específico: "En redis-store.ts, el TTL debe ser 14400 (4 horas en segundos), no 3600."

---

### Paso 1.4: El agente genera código y abre PR

El agente crea una feature branch, hace commits atómicos y abre el PR:

```
Branch: feature/conversation-agent-base
PR título: feat: webhook handler y ConversationAgent base (#1)

## ¿Qué hace este PR?
Implementa el núcleo del bot: recepción de mensajes de WhatsApp vía Twilio y
respuesta usando Azure OpenAI GPT-4o con contexto persistente en Redis.

## Cambios
- `src/webhooks/twilio-handler.ts`: valida firma Twilio, extrae mensaje, responde TwiML
- `src/memory/redis-store.ts`: manejo de contexto con TTL 4h
- `src/agents/conversation.agent.ts`: orquestador, llama a GPT-4o
- `tests/conversation.agent.test.ts`: 6 tests unitarios (todos pasan)

## Cómo probar localmente
1. cp .env.example .env (y completar valores)
2. npm install
3. npm run dev
4. Enviar mensaje al sandbox de Twilio

Closes #1
```

---

### Paso 1.5: CI valida automáticamente

El pipeline de CI corre automáticamente en el PR (ver Fase 5 para el YAML completo):

```
✅ lint          (eslint)          — 0 errores
✅ type-check    (tsc --noEmit)    — 0 errores
✅ unit-tests    (jest)            — 6/6 tests pasan
✅ coverage      (jest --coverage) — 84% (> 80% mínimo)
```

Si algún check falla, el agente recibe el error y puede corregir con un nuevo commit.

---

### Paso 1.6: Human review + merge

Revisión humana final antes del merge:

```bash
# Ver el diff del PR
gh pr diff <número>

# Aprobar y hacer merge
gh pr review <número> --approve --body "Implementación limpia, tests completos. ✅"
gh pr merge <número> --squash --delete-branch
```

> **Regla GH-600:** No hacer merge sin que el CI esté verde y con al menos 1 aprobación humana.

---

## Fase 2: Implementar MCP Servers (herramientas del agente de IA)

> **GH-600 §S2 — MCP Configuration & Tool Use**

Los MCP Servers son las "herramientas" que el agente de IA puede invocar. Cada servidor expone funciones específicas con permisos acotados.

---

### Paso 2.1: Diseñar los MCP Servers

**Principio de menor privilegio:** cada servidor tiene acceso solo a lo que necesita.

| Servidor | Tipo | Acceso DB | Herramientas |
|---|---|---|---|
| `calendar-mcp` | Solo lectura | `SELECT` en `courts`, `activities`, `slots` | `getActivities`, `getCourts`, `getAvailability` |
| `member-mcp` | Lectura + perfil | `SELECT` en `members`, `memberships` | `lookupMember`, `getMembershipStatus` |
| `booking-mcp` | Escritura (alta sensibilidad) | `SELECT + INSERT + UPDATE` en `bookings` | `createBooking`, `cancelBooking` |

**Definición detallada de herramientas:**

```typescript
// calendar-mcp: herramientas disponibles
getActivities()
// → Activity[] { id, name, description, schedule, instructor }

getCourts()
// → Court[] { id, name, type: 'padel'|'tennis', indoor: boolean }

getAvailability(date: string, courtId: string)
// → Slot[] { startTime, endTime, available: boolean }

// member-mcp: herramientas disponibles
lookupMember(phone: string)
// → Member | null { id, name, phone, email }

getMembershipStatus(memberId: string)
// → MembershipStatus { active: boolean, plan: string, expiresAt: Date }

// booking-mcp: herramientas disponibles (alta sensibilidad)
createBooking(memberId: string, courtId: string, slot: string)
// → Booking { id, confirmationCode, details }

cancelBooking(bookingId: string)
// → { success: boolean, message: string }
```

---

### Paso 2.2: Configurar MCP en el repositorio

**Crear `.vscode/mcp.json`:**

```json
{
  "servers": {
    "calendar": {
      "type": "stdio",
      "command": "node",
      "args": ["./dist/mcp/calendar-server.js"],
      "env": {
        "DB_URL": "${env:DATABASE_URL}",
        "LOG_LEVEL": "info"
      }
    },
    "member": {
      "type": "stdio",
      "command": "node",
      "args": ["./dist/mcp/member-server.js"],
      "env": {
        "DB_URL": "${env:DATABASE_URL}"
      }
    },
    "booking": {
      "type": "stdio",
      "command": "node",
      "args": ["./dist/mcp/booking-server.js"],
      "env": {
        "DB_URL": "${env:DATABASE_URL}",
        "REQUIRE_CONFIRMATION": "true",
        "AUDIT_LOG_ENABLED": "true"
      }
    }
  }
}
```

**Esquema de un MCP Server (ejemplo: `src/mcp/calendar-server.ts`):**

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { db } from "../db/client.js";

const server = new McpServer({
  name: "calendar-mcp",
  version: "1.0.0",
});

server.tool(
  "getAvailability",
  "Obtiene los turnos disponibles para una cancha en una fecha",
  {
    date: z.string().describe("Fecha en formato YYYY-MM-DD"),
    courtId: z.string().describe("ID de la cancha"),
  },
  async ({ date, courtId }) => {
    const slots = await db.slot.findMany({
      where: { date, courtId, available: true },
      orderBy: { startTime: "asc" },
    });
    return {
      content: [{ type: "text", text: JSON.stringify(slots) }],
    };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

---

### Paso 2.3: Configurar logging de MCP

**Regla GH-600:** Toda llamada a un MCP tool debe quedar registrada. Especialmente crítico para herramientas de escritura.

```typescript
// src/mcp/middleware/audit-logger.ts

interface ToolCallLog {
  timestamp: string;
  server: string;
  tool: string;
  params: Record<string, unknown>;   // sanitizado — sin datos sensibles
  durationMs: number;
  success: boolean;
  error?: string;
}

export function withAuditLog(
  serverName: string,
  toolName: string,
  handler: (params: unknown) => Promise<unknown>
) {
  return async (params: unknown) => {
    const start = Date.now();
    const log: ToolCallLog = {
      timestamp: new Date().toISOString(),
      server: serverName,
      tool: toolName,
      params: sanitize(params),   // eliminar PII antes de loggear
      durationMs: 0,
      success: false,
    };

    try {
      const result = await handler(params);
      log.success = true;
      log.durationMs = Date.now() - start;
      await writeAuditLog(log);
      return result;
    } catch (error: unknown) {
      log.durationMs = Date.now() - start;
      log.error = error instanceof Error ? error.message : "Unknown error";
      await writeAuditLog(log);
      throw error;
    }
  };
}
```

---

## Fase 3: Implementar agentes especializados (arquitectura multi-agente)

> **GH-600 §S5 — Orquestación multi-agente**

---

### Paso 3.1: ScheduleAgent

**Feature branch:** `feature/schedule-agent`

**Issue para el coding agent:**

```
Título: feat: ScheduleAgent — consulta de actividades y disponibilidad

## Herramientas MCP disponibles
- calendar-mcp: getActivities, getCourts, getAvailability

## Sistema prompt
"Sos el asistente de horarios del club. Respondés preguntas sobre actividades,
clases, entrenadores y disponibilidad de canchas. Siempre mostrás los horarios
en formato claro (HH:MM). Si no hay disponibilidad, ofrecés la próxima fecha libre."

## Criterios de aceptación
- [ ] Responde correctamente a: "¿Qué canchas hay disponibles mañana a las 18?"
- [ ] Responde correctamente a: "¿Cuándo da clases el profe Martín?"
- [ ] Maneja el caso de ninguna disponibilidad con mensaje amigable
- [ ] Tests con mocks de calendar-mcp
```

**Estructura del agente:**

```typescript
// src/agents/schedule.agent.ts
export class ScheduleAgent {
  private tools = ["getActivities", "getCourts", "getAvailability"];
  private systemPrompt = loadPrompt("schedule");

  async handle(userMessage: string, context: Message[]): Promise<string> {
    // Llama a GPT-4o con tool_choice para MCP tools de calendar
    // Interpreta la respuesta y formatea para WhatsApp
  }
}
```

---

### Paso 3.2: MemberAgent

**Feature branch:** `feature/member-agent`

**Sistema prompt:**

```
Sos el asistente de socios del club. Tenés acceso a información de membresía.
NUNCA compartas datos personales de un socio con otra persona que no sea el propio socio.
Verificá siempre que el número de teléfono que consulta coincide con el socio.
Si la membresía está vencida, informalo amablemente y ofrecé opciones de renovación.
```

**Herramientas MCP:** `lookupMember`, `getMembershipStatus`

---

### Paso 3.3: BookingAgent (alta sensibilidad — proceso especial)

**Feature branch:** `feature/booking-agent`

**Usar el agente especializado:** Activar `.agents/booking-specialist.agent.md` para este desarrollo.

**Proceso de desarrollo con pasos adicionales:**

#### 3.3.1 — TDD: escribir tests primero

```typescript
// tests/booking.agent.test.ts (se escribe ANTES del código)

describe("BookingAgent", () => {
  it("NO ejecuta createBooking sin confirmación del usuario", async () => {
    const agent = new BookingAgent();
    const response = await agent.handle("quiero reservar cancha 3 mañana a las 18", []);
    
    // Debe pedir confirmación, NO llamar createBooking
    expect(mockCreateBooking).not.toHaveBeenCalled();
    expect(response).toContain("Confirmás");
  });

  it("ejecuta createBooking cuando el usuario confirma con 'Sí'", async () => {
    const agent = new BookingAgent();
    const context = [/* ... historial con la reserva propuesta ... */];
    
    await agent.handle("Sí", context);
    expect(mockCreateBooking).toHaveBeenCalledTimes(1);
  });

  it("NO ejecuta createBooking con respuestas ambiguas ('dale', 'ok')", async () => {
    const agent = new BookingAgent();
    await agent.handle("dale", contextWithPendingBooking);
    expect(mockCreateBooking).not.toHaveBeenCalled();
  });

  it("maneja error de createBooking y notifica al usuario", async () => {
    mockCreateBooking.mockRejectedValue(new Error("Cancha no disponible"));
    const agent = new BookingAgent();
    const response = await agent.handle("Sí", contextWithPendingBooking);
    expect(response).toContain("no se pudo confirmar");
  });
});
```

#### 3.3.2 — Implementar el agente con gate de confirmación

```typescript
// src/agents/booking.agent.ts

export class BookingAgent {
  private systemPrompt = `
    Sos el asistente de reservas del club. 

    REGLA CRÍTICA: Nunca ejecutes createBooking directamente.
    Flujo obligatorio:
    1. Identificar la intención de reserva
    2. Consultar disponibilidad con getAvailability
    3. Proponer la reserva con TODOS los detalles (cancha, fecha, hora, precio)
    4. Pedir confirmación EXPLÍCITA: "¿Confirmás esta reserva? Respondé Sí o No."
    5. Solo si el usuario responde exactamente "Sí" o "Si" → ejecutar createBooking
    6. Confirmar la reserva con el código de confirmación
  `;

  async handle(message: string, context: Message[]): Promise<string> {
    const pendingBooking = this.extractPendingBooking(context);
    
    // Si hay una reserva pendiente de confirmación y el usuario dice "Sí"
    if (pendingBooking && this.isExplicitConfirmation(message)) {
      return await this.executeBooking(pendingBooking);
    }

    // En cualquier otro caso, el agente propone pero no ejecuta
    return await this.proposeBooking(message, context);
  }

  private isExplicitConfirmation(message: string): boolean {
    return /^s[ií]$/i.test(message.trim());
  }
}
```

#### 3.3.3 — CI adicional para BookingAgent

```yaml
# .github/workflows/booking-integration.yml
# Corre solo cuando hay cambios en src/agents/booking* o src/mcp/booking*

- name: Integration tests — BookingAgent
  run: npm run test:integration -- --testPathPattern=booking
  env:
    DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
```

---

### Paso 3.4: Conectar el orquestador (ConversationAgent)

El `ConversationAgent` clasifica la intención y delega al agente especializado:

```typescript
// src/agents/conversation.agent.ts

type Intent = "schedule" | "booking" | "member" | "general" | "escalate";

export class ConversationAgent {
  private agents = {
    schedule: new ScheduleAgent(),
    booking: new BookingAgent(),
    member: new MemberAgent(),
  };

  async handle(phone: string, message: string): Promise<string> {
    const context = await redisStore.getContext(phone);

    // Detectar intención con GPT-4o
    const intent = await this.classifyIntent(message, context);

    // Escalar a humano si es necesario
    if (intent === "escalate" || this.needsEscalation(context)) {
      return await this.escalateToHuman(phone);
    }

    // Delegar al agente especializado
    const specialist = this.agents[intent] ?? new GeneralAgent();
    const response = await specialist.handle(message, context);

    // Guardar en contexto
    await redisStore.saveContext(phone, [
      ...context,
      { role: "user", content: message },
      { role: "assistant", content: response },
    ]);

    return response;
  }

  private needsEscalation(context: Message[]): boolean {
    // Escalar si el agente falló 3 veces seguidas
    const lastThree = context.slice(-6);
    const agentMessages = lastThree.filter((m) => m.role === "assistant");
    return agentMessages.every((m) => m.content.includes("no pude"));
  }
}
```

---

## Fase 4: Gestión de memoria y estado

> **GH-600 §S3 — Estado, memoria y contexto de conversación**

---

### Paso 4.1: Implementar sliding window de contexto

**Estructura de key en Redis:**
- Key: `conversation:{phoneNumber}` (ej: `conversation:+5491112345678`)
- TTL: 14400 segundos (4 horas)
- Tipo: JSON array de mensajes

```typescript
// src/memory/redis-store.ts

const MAX_MESSAGES = 20;
const TTL_SECONDS = 14400; // 4 horas

export class RedisConversationStore {
  constructor(private client: Redis) {}

  async getContext(phone: string): Promise<Message[]> {
    const key = `conversation:${phone}`;
    const raw = await this.client.get(key);
    return raw ? JSON.parse(raw) : [];
  }

  async saveContext(phone: string, messages: Message[]): Promise<void> {
    const key = `conversation:${phone}`;
    // Sliding window: mantener solo los últimos MAX_MESSAGES
    const window = messages.slice(-MAX_MESSAGES);
    await this.client.setex(key, TTL_SECONDS, JSON.stringify(window));
  }

  async clearContext(phone: string): Promise<void> {
    await this.client.del(`conversation:${phone}`);
  }
}
```

---

### Paso 4.2: Configurar Copilot Memory para el repositorio

Copilot Memory permite que el coding agent recuerde decisiones y contexto del proyecto entre sesiones.

```
GitHub org settings → Copilot → Policies → Copilot Memory → Enabled
```

El agente irá acumulando conocimiento sobre el proyecto: patrones preferidos, decisiones de arquitectura, bugs conocidos.

---

### Paso 4.3: Manejo de deriva de contexto

Triggers para resetear el contexto de conversación:

```typescript
// src/memory/context-manager.ts

const RESET_TRIGGERS = [
  /empezar de nuevo/i,
  /comenzar de cero/i,
  /olvidá todo/i,
  /nueva consulta/i,
];

export function shouldResetContext(
  message: string,
  lastActivity: Date
): boolean {
  // Reset explícito del usuario
  if (RESET_TRIGGERS.some((t) => t.test(message))) return true;

  // Inactividad de más de 30 minutos → reset suave (mantener perfil, limpiar intención)
  const inactiveMs = Date.now() - lastActivity.getTime();
  if (inactiveMs > 30 * 60 * 1000) return true;

  return false;
}
```

---

## Fase 5: CI/CD con GitHub Actions

> **GH-600 §S1 — SDLC Integration & §S2 — Entornos**

---

### Paso 5.1: Pipeline de CI (cada PR)

**Crear `.github/workflows/ci.yml`:**

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    name: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npm run lint

  type-check:
    name: type-check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npx tsc --noEmit

  unit-tests:
    name: unit-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npm test -- --coverage --coverageThreshold='{"global":{"lines":80}}'

  integration-tests:
    name: integration-tests
    runs-on: ubuntu-latest
    # Solo corre cuando cambia código de booking
    if: |
      contains(github.event.pull_request.changed_files, 'src/agents/booking') ||
      contains(github.event.pull_request.changed_files, 'src/mcp/booking')
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: clubbot_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/clubbot_test
```

---

### Paso 5.2: Pipeline de Deploy (merge a main)

**Crear `.github/workflows/deploy.yml`:**

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    name: Build & Push Docker image
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Login to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: clubbot.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build & Push
        id: meta
        run: |
          TAG=clubbot.azurecr.io/clubbot-whatsapp:${{ github.sha }}
          docker build -t $TAG .
          docker push $TAG
          echo "tags=$TAG" >> $GITHUB_OUTPUT

  deploy-staging:
    name: Deploy to Staging
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging        # auto-deploy, sin aprobación manual
    steps:
      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: rg-clubbot
          containerAppName: clubbot-staging
          imageToDeploy: ${{ needs.build-and-push.outputs.image-tag }}

  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production      # requiere aprobación manual (ver Paso 5.3)
    steps:
      - name: Deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: rg-clubbot
          containerAppName: clubbot-production
          imageToDeploy: ${{ needs.build-and-push.outputs.image-tag }}
```

---

### Paso 5.3: Configurar Environments con aprobación

```
GitHub repo → Settings → Environments

1. Crear environment "staging"
   - No required reviewers
   - Auto-deploy

2. Crear environment "production"  
   - Required reviewers: @lead-dev, @senior-dev
   - Wait timer: 0 minutos (puede subir a 5 si quieren pausa reflexiva)
   - Deployment branches: main only
```

**Secrets por environment:**

```bash
# Staging
gh secret set AZURE_OPENAI_ENDPOINT --env staging --body "https://clubbot-staging.openai.azure.com/"

# Production
gh secret set AZURE_OPENAI_ENDPOINT --env production --body "https://clubbot-prod.openai.azure.com/"
```

---

## Fase 6: Observabilidad y evaluación

> **GH-600 §S4 — Evaluación, métricas y loop de ajuste**

---

### Paso 6.1: Instrumentación con Application Insights

```bash
npm install @azure/monitor-opentelemetry applicationinsights
```

```typescript
// src/telemetry/instrumentation.ts

import appInsights from "applicationinsights";

appInsights
  .setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING)
  .setAutoCollectRequests(true)
  .setAutoCollectExceptions(true)
  .start();

export const client = appInsights.defaultClient;

// Loggear cada mensaje entrante (sanitizado — sin contenido personal)
export function trackIncomingMessage(phone: string, intent: string) {
  client.trackEvent({
    name: "IncomingMessage",
    properties: {
      phoneHash: hash(phone),   // NUNCA el teléfono real
      intent,
      timestamp: new Date().toISOString(),
    },
  });
}

// Loggear cada llamada a agente especializado
export function trackAgentCall(
  agent: string,
  durationMs: number,
  success: boolean
) {
  client.trackDependency({
    name: `Agent:${agent}`,
    duration: durationMs,
    success,
    dependencyTypeName: "AI Agent",
    data: agent,
  });
}

// Loggear errores
export function trackError(error: Error, context: Record<string, string>) {
  client.trackException({ exception: error, properties: context });
}
```

---

### Paso 6.2: Definir métricas de éxito

| Métrica | Descripción | Target | Umbral de alerta |
|---|---|---|---|
| **Booking completion rate** | % de intenciones de reserva que terminan en reserva confirmada | > 85% | < 75% |
| **Escalation rate** | % de conversaciones que escalan a humano involuntariamente | < 10% | > 15% |
| **Response time p95** | Percentil 95 de tiempo de respuesta del bot | < 5s | > 8s |
| **Context hit rate** | % de mensajes que encuentran contexto en Redis | > 70% | < 50% |
| **MCP tool error rate** | % de llamadas a MCP tools que fallan | < 2% | > 5% |

**Query KQL en Application Insights:**

```kql
// Booking completion rate — últimas 24h
customEvents
| where name == "IncomingMessage" and timestamp > ago(24h)
| summarize
    totalBookingIntents = countif(properties.intent == "booking"),
    completedBookings   = countif(properties.intent == "booking_confirmed")
| extend completionRate = todouble(completedBookings) / todouble(totalBookingIntents) * 100
| project completionRate
```

---

### Paso 6.3: Loop de ajuste semanal

```
Cada lunes — 30 minutos — proceso de ajuste:

1. PULL datos de la semana anterior desde Application Insights
   → ¿Cuál fue el booking completion rate?
   → ¿Cuáles fueron los top 5 errores?
   → ¿En qué paso del flujo se abandonan más conversaciones?

2. IDENTIFICAR el problema principal
   → Si booking completion < 85%: revisar el flujo de confirmación
   → Si escalation rate > 10%: revisar el prompt del ConversationAgent
   → Si response time > 5s: revisar llamadas a MCP o contexto de Redis

3. ACTUALIZAR system prompts o lógica
   → Crear issue en GitHub con los cambios propuestos
   → Asignar al coding agent para que implemente
   → Revisar el PR antes de hacer merge

4. DESPLEGAR y medir impacto en la semana siguiente
```

---

## Fase 7: Guardrails y responsabilidad

> **GH-600 §S6 — Protección, permisos mínimos y auditoría**

---

### Paso 7.1: Configurar Content Safety en Azure AI Foundry

```
Azure AI Foundry Studio → tu proyecto → Content Filters → New filter

Configuración recomendada:
- Hate speech:   Input Medium, Output Medium
- Violence:      Input Medium, Output Medium  
- Self-harm:     Input High,   Output High    ← más estricto
- Sexual:        Input Medium, Output Medium

Aplicar a: deployment "clubbot-gpt4o"
```

**Código de manejo de bloqueos:**

```typescript
// Cuando Azure rechaza por content safety
if (error.code === "content_filter") {
  logger.warn("Content Safety bloqueó una respuesta", { phone: hash(phone) });
  return "Lo siento, no puedo responder eso. ¿En qué otra cosa puedo ayudarte con el club?";
}
```

---

### Paso 7.2: Configurar permisos mínimos

**GitHub Actions — token de mínimo privilegio:**

```yaml
# En cada workflow, declarar permisos explícitos
permissions:
  contents: read        # leer el repo
  packages: write       # push de imágenes (solo en deploy)
  # NO declarar: pull-requests: write, issues: write, etc.
```

**Azure — Managed Identity (sin contraseñas hardcodeadas):**

```bash
# Crear managed identity para la Container App
az identity create \
  --name clubbot-identity \
  --resource-group rg-clubbot

# Asignar rol de lectura a Cognitive Services (solo lo necesario)
az role assignment create \
  --assignee <identity-client-id> \
  --role "Cognitive Services OpenAI User" \
  --scope /subscriptions/<sub>/resourceGroups/rg-clubbot/providers/Microsoft.CognitiveServices/accounts/clubbot-ai
```

**Base de datos — usuarios con privilegios separados:**

```sql
-- Usuario de solo lectura (para calendar-mcp y member-mcp)
CREATE USER clubbot_reader WITH PASSWORD '<strong-password>';
GRANT SELECT ON courts, slots, activities, members, memberships TO clubbot_reader;

-- Usuario de lectura-escritura (SOLO para booking-mcp)
CREATE USER clubbot_booking WITH PASSWORD '<strong-password>';
GRANT SELECT, INSERT, UPDATE ON bookings TO clubbot_booking;
GRANT SELECT ON members, courts, slots TO clubbot_booking;
-- NO conceder DELETE, DROP, CREATE, etc.
```

---

### Paso 7.3: Audit trail completo

```sql
-- Tabla de auditoría (PostgreSQL)
CREATE TABLE booking_audit (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  member_id     UUID NOT NULL,
  action        TEXT NOT NULL,        -- 'create', 'cancel'
  booking_id    UUID,
  agent_name    TEXT NOT NULL,        -- 'BookingAgent'
  mcp_tool      TEXT NOT NULL,        -- 'createBooking'
  mcp_params    JSONB NOT NULL,       -- parámetros (sin datos sensibles)
  result        TEXT NOT NULL,        -- 'success', 'error'
  error_message TEXT,
  session_id    UUID NOT NULL         -- ID de la sesión de conversación
);

-- Índices para consultas frecuentes
CREATE INDEX idx_booking_audit_member ON booking_audit(member_id, created_at DESC);
CREATE INDEX idx_booking_audit_action ON booking_audit(action, created_at DESC);
```

```typescript
// Insertar en audit trail en booking-mcp-server.ts
await db.bookingAudit.create({
  data: {
    memberId,
    action: "create",
    bookingId: result.id,
    agentName: "BookingAgent",
    mcpTool: "createBooking",
    mcpParams: { courtId, slot },    // NO incluir datos personales
    result: "success",
    sessionId,
  },
});
```

---

### Paso 7.4: Human escalation path

```typescript
// src/agents/escalation.ts

export async function escalateToHuman(phone: string, context: Message[]): Promise<string> {
  // 1. Notificar al staff (email + Slack)
  await notifyStaff({
    channel: "#clubbot-escalations",
    message: `Nuevo escalamiento desde WhatsApp`,
    metadata: {
      sessionId: generateSessionId(phone),
      messageCount: context.length,
      lastMessages: context.slice(-3).map((m) => m.content),
    },
  });

  // 2. Registrar en la DB para seguimiento
  await db.escalation.create({
    data: {
      phoneHash: hash(phone),
      triggeredAt: new Date(),
      contextLength: context.length,
    },
  });

  // 3. Responder al usuario
  return (
    "Entendido 🤝 Te comunico con uno de nuestros asesores. " +
    "Un miembro del equipo del club te va a responder en breve. " +
    "Nuestro horario de atención es lunes a viernes de 9 a 20 hs."
  );
}
```

**Triggers de escalación:**

```typescript
const ESCALATION_PHRASES = [
  /hablar con (una )?persona/i,
  /quiero un asesor/i,
  /con alguien del club/i,
  /no me estás ayudando/i,
  /esto no funciona/i,
];

// En ConversationAgent
if (
  ESCALATION_PHRASES.some((p) => p.test(message)) ||
  this.consecutiveFailures(context) >= 3
) {
  return escalateToHuman(phone, context);
}
```

---

## Resumen: qué sección de GH-600 aplica en cada fase

| Fase | Concepto GH-600 | Artefactos clave |
|---|---|---|
| **Fase 0: Setup** | §S1 SDLC integration, §S6 guardrails | `copilot-instructions.md`, `CODEOWNERS`, secrets |
| **Fase 1: Núcleo** | §S1 plan→act→evaluate, §S3 memoria | `conversation.agent.ts`, `redis-store.ts`, CI |
| **Fase 2: MCP Servers** | §S2 MCP configuration y herramientas | `mcp.json`, `*-server.ts`, audit logger |
| **Fase 3: Multi-agente** | §S5 orquestación de agentes | `booking.agent.ts` (TDD), `.agents/*.agent.md` |
| **Fase 4: Memoria** | §S3 estado y contexto | Redis sliding window, context reset triggers |
| **Fase 5: CI/CD** | §S1 SDLC, §S2 entornos | `ci.yml`, `deploy.yml`, GitHub Environments |
| **Fase 6: Evaluación** | §S4 evaluación y ajuste continuo | Application Insights, KQL queries, loop semanal |
| **Fase 7: Guardrails** | §S6 protección y responsabilidad | Content Safety, mínimo privilegio, audit trail |

---

## Comandos de referencia rápida

```bash
# Desarrollo local
npm run dev                          # Levanta el bot localmente
ngrok http 3000                      # Expone el webhook para Twilio sandbox

# Tests
npm test                             # Unit tests
npm run test:integration             # Integration tests (requiere DB)
npm run test:coverage                # Con reporte de coverage

# Deploy manual (emergencia)
az containerapp update \
  --name clubbot-production \
  --resource-group rg-clubbot \
  --image clubbot.azurecr.io/clubbot-whatsapp:<sha>

# Ver logs en producción
az containerapp logs show \
  --name clubbot-production \
  --resource-group rg-clubbot \
  --follow
```

---

*Guía creada para el estudio del examen GH-600 — sección de agentes de IA en el ciclo de vida del software.*
