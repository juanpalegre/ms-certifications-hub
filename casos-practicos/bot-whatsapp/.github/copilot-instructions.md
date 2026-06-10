# Copilot Coding Agent — Instrucciones para el Repositorio ClubBot

> Este archivo configura el comportamiento del **Copilot coding agent** cuando trabaja en este repositorio.
> Se aplica automáticamente a todas las sesiones del agente, PRs generados, y sugerencias de código.
> Referencia: [Customizing Copilot for your repository](https://docs.github.com/en/copilot/customizing-copilot)

---

## Descripción del Proyecto

**ClubBot** es un bot de WhatsApp para la gestión de un club deportivo. Permite a los socios:
- Reservar pistas y espacios deportivos mediante lenguaje natural
- Consultar disponibilidad de instalaciones en tiempo real
- Recibir confirmaciones y recordatorios por WhatsApp

### Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| Runtime | Node.js 20 + TypeScript (strict mode) |
| Bot Framework | `@whiskeysockets/baileys` (WhatsApp Web API) |
| IA | Azure OpenAI (GPT-4o) via Azure AI Foundry |
| Herramientas | Model Context Protocol (MCP) servers |
| Caché / Estado | Redis |
| Base de datos | PostgreSQL (via Prisma ORM) |
| Despliegue | Azure Container Apps |
| Observabilidad | Azure Application Insights |

---

## Arquitectura y Principios de Diseño

### Patrón de Agentes

El bot usa una arquitectura de **agentes especializados**. Cada dominio tiene su propio agente:

```
src/
├── agents/
│   ├── booking/        ← BookingAgent: reservas de pistas
│   ├── availability/   ← AvailabilityAgent: consulta de horarios
│   └── notification/   ← NotificationAgent: recordatorios
├── mcp/
│   ├── booking/        ← MCP server con herramientas de escritura (createBooking, cancelBooking)
│   └── calendar/       ← MCP server de solo lectura (getAvailability)
├── whatsapp/           ← Integración con Baileys
└── shared/             ← Tipos, utils, configuración
```

### Principio de Mínimo Privilegio para Agentes

- Los agentes de **solo consulta** no tienen acceso a herramientas de escritura
- Los agentes de **escritura** (BookingAgent) siempre requieren confirmación explícita del usuario
- Ningún agente puede acceder a datos de otros socios sin verificación de identidad

---

## Convenciones de Código

### TypeScript

```typescript
// ✅ CORRECTO: tipos explícitos, sin "any"
async function createBooking(input: BookingInput): Promise<BookingResult> { ... }

// ❌ INCORRECTO: any implícito o explícito
async function createBooking(input: any): Promise<any> { ... }
```

- **Strict mode**: `"strict": true` en `tsconfig.json` — no desactivar ninguna opción
- **No `any`**: usa `unknown` + type guards cuando el tipo es verdaderamente desconocido
- **Imports**: usa paths absolutos con alias `@/` (configurado en tsconfig y jest)
- **Exports**: un export por archivo para facilitar el testing

### Nomenclatura

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Clases / Tipos | PascalCase | `BookingAgent`, `BookingInput` |
| Funciones / Variables | camelCase | `createBooking`, `userId` |
| Constantes globales | SCREAMING_SNAKE | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Archivos | kebab-case | `booking-agent.ts`, `booking-agent.test.ts` |
| Env vars | SCREAMING_SNAKE | `AZURE_OPENAI_ENDPOINT` |

### Testing (Jest + ts-jest)

```typescript
// Estructura de tests: describe → it con nombres descriptivos
describe('BookingAgent', () => {
  describe('when user confirms booking', () => {
    it('should call createBooking MCP tool with correct parameters', async () => {
      // Arrange
      const agent = new BookingAgent({ mcpClient: mockMcpClient });
      
      // Act
      const result = await agent.handleConfirmation(mockSession);
      
      // Assert
      expect(mockMcpClient.callTool).toHaveBeenCalledWith('createBooking', {
        courtId: 'court-1',
        startTime: '2024-03-15T10:00:00Z',
        userId: 'user-123',
      });
    });
  });
});
```

- Un archivo `*.test.ts` por cada archivo de código fuente
- **Cobertura mínima: 80%** (configurada en `jest.config.ts` como `coverageThreshold`)
- Los mocks de Azure OpenAI y MCP deben estar en `src/__mocks__/`
- Usa `jest.useFakeTimers()` para tests que involucren timeouts de WhatsApp

---

## ⚠️ REGLAS CRÍTICAS — El Agente NUNCA Debe Hacer Esto

> Estas reglas son no negociables. Si una tarea requiere violar alguna de ellas, escalar a un humano.

### REGLA 1: Nunca Hardcodear Secrets o API Keys

```typescript
// ❌ ABSOLUTAMENTE PROHIBIDO
const client = new OpenAIClient('https://my-instance.openai.azure.com', 'sk-abc123...');

// ✅ CORRECTO: siempre desde variables de entorno
const client = new OpenAIClient(
  process.env.AZURE_OPENAI_ENDPOINT!,
  new AzureKeyCredential(process.env.AZURE_OPENAI_KEY!)
);
```

Esto incluye: API keys, connection strings, passwords, tokens, webhooks URLs con auth.

### REGLA 2: BookingAgent Siempre Requiere Confirmación Humana

El flujo de reserva tiene DOS pasos obligatorios:

```
Usuario: "Quiero reservar la pista 1 mañana a las 10h"
Bot: "¿Confirmas la reserva de Pista 1 el martes 15/03 a las 10:00? (Responde SÍ para confirmar)"
Usuario: "Sí"
Bot: [SOLO AQUÍ se llama a createBooking] "✅ Reserva confirmada. Referencia: #12345"
```

```typescript
// ❌ NUNCA: crear reserva sin confirmación explícita
async handleBookingRequest(session: Session, intent: BookingIntent) {
  const result = await this.mcpClient.callTool('createBooking', intent.params);
  return `Reserva creada: ${result.bookingId}`;
}

// ✅ CORRECTO: dos fases con estado en Redis
async handleBookingRequest(session: Session, intent: BookingIntent) {
  await this.sessionStore.setPendingBooking(session.userId, intent.params);
  return this.buildConfirmationMessage(intent.params); // devuelve mensaje de confirmación
}

async handleConfirmation(session: Session) {
  const pending = await this.sessionStore.getPendingBooking(session.userId);
  if (!pending) throw new Error('No hay reserva pendiente de confirmación');
  const result = await this.mcpClient.callTool('createBooking', pending);
  await this.sessionStore.clearPendingBooking(session.userId);
  return `✅ Reserva confirmada. Ref: ${result.bookingId}`;
}
```

### REGLA 3: MCP Write Servers Deben Registrar Cada Operación

Cada herramienta de escritura en los MCP servers debe registrar en el audit log:

```typescript
// En cualquier MCP tool de escritura (createBooking, cancelBooking, etc.)
export async function createBooking(input: CreateBookingInput): Promise<BookingResult> {
  // Log ANTES de ejecutar — para detectar intentos aunque fallen
  await auditLog.record({
    action: 'createBooking',
    actor: input.requestedBy,        // userId o 'agent:booking'
    params: sanitize(input),          // sin datos sensibles
    timestamp: new Date().toISOString(),
    source: 'mcp-booking-server',
  });
  
  const result = await db.booking.create({ data: mapToDbModel(input) });
  
  await auditLog.record({
    action: 'createBooking',
    result: 'success',
    bookingId: result.id,
  });
  
  return mapToApiResult(result);
}
```

### REGLA 4: Nunca Saltarse el Confirmation Gate en Tests con APIs Reales

```typescript
// ❌ PROHIBIDO: bypass del gate en tests que usen APIs reales
it('should create booking directly', async () => {
  process.env.SKIP_CONFIRMATION = 'true'; // ← NUNCA
  const result = await agent.createBookingDirectly(params);
});

// ✅ CORRECTO: usar mocks para tests unitarios
it('should call createBooking after confirmation', async () => {
  const mockMcp = { callTool: jest.fn().mockResolvedValue({ bookingId: '123' }) };
  const agent = new BookingAgent({ mcpClient: mockMcp });
  await agent.handleConfirmation(mockSession);
  expect(mockMcp.callTool).toHaveBeenCalledWith('createBooking', expect.any(Object));
});
```

---

## Qué Puede Hacer el Agente Autónomamente

El agente puede crear PRs y realizar cambios sin aprobación previa para:

- ✅ Añadir o modificar tests unitarios y de integración
- ✅ Refactorizar código sin cambiar comportamiento observable (pasar CI primero)
- ✅ Corregir bugs de tipado TypeScript reportados en issues
- ✅ Actualizar dependencias de desarrollo (devDependencies)
- ✅ Añadir nuevas herramientas MCP de **solo lectura**
- ✅ Mejorar mensajes de error y logging
- ✅ Documentar funciones y módulos existentes (JSDoc / TSDoc)

## Qué Debe Escalar a Revisión Humana

El agente debe crear un PR y **esperar aprobación** (no auto-merge) para:

- 🔴 Cambios en el flujo de confirmación de BookingAgent
- 🔴 Nuevas herramientas MCP de escritura (crean/modifican/borran datos)
- 🔴 Cambios en la autenticación o verificación de identidad de socios
- 🔴 Actualizaciones de dependencias de producción (`dependencies`, no `devDependencies`)
- 🔴 Cambios en workflows de CI/CD o permisos de GitHub Actions
- 🔴 Cualquier cambio que toque el audit log o mecanismos de seguridad
- 🔴 Nuevos endpoints HTTP o cambios en la superficie de la API

---

## Cómo Escribir Buenos Mensajes de Commit

El agente debe usar [Conventional Commits](https://www.conventionalcommits.org/):

```
<tipo>(<scope>): <descripción corta en inglés>

<cuerpo opcional: explica el POR QUÉ, no el qué>

<footer: referencias a issues>
```

### Tipos válidos

| Tipo | Cuándo usarlo |
|------|--------------|
| `feat` | Nueva funcionalidad |
| `fix` | Corrección de bug |
| `test` | Añadir o corregir tests |
| `refactor` | Refactorización sin cambio de comportamiento |
| `docs` | Solo documentación |
| `chore` | Tareas de mantenimiento (deps, config) |
| `perf` | Mejora de rendimiento |

### Ejemplos

```
feat(booking): add 24h cancellation window validation

Before this change, users could cancel bookings with less than 1h notice,
causing operational issues for club staff. Now the BookingAgent checks the
cancellation window before calling the MCP cancel tool.

Closes #42
```

```
test(booking-agent): add confirmation gate unit tests

Adds 8 tests covering the two-phase booking flow. Tests verify that
createBooking is never called without prior user confirmation.
Improves coverage from 73% to 87%.
```

---

## Cómo Estructurar las Descripciones de PR

El agente debe usar esta plantilla para todos sus PRs:

```markdown
## ¿Qué hace este PR?
<!-- Descripción concisa del cambio -->

## ¿Por qué este cambio?
<!-- El problema que resuelve o la tarea del issue -->

## Decisiones de implementación
<!-- Alternativas consideradas y por qué se eligió este enfoque -->
- **Opción elegida**: ...porque...
- **Alternativa descartada**: ...porque habría causado...

## Tests añadidos/modificados
<!-- Lista de tests nuevos o modificados -->
- `booking-agent.test.ts`: tests del flujo de confirmación
- `mcp-booking-server.test.ts`: tests del audit log

## Checklist
- [ ] Los tests pasan localmente (`npm run test:unit`)
- [ ] No hay secrets hardcodeados
- [ ] El confirmation gate está intacto (si se tocó BookingAgent)
- [ ] El audit log registra las operaciones de escritura nuevas
- [ ] Los tipos TypeScript son correctos (sin `any`)

## Impacto en producción
<!-- Si hay cambios en comportamiento visible para el usuario -->
```

---

## Configuración del Entorno Local

```bash
# 1. Clonar y dependencias
git clone <repo-url> && cd caso-practico-whatsapp
npm install

# 2. Variables de entorno (NUNCA committear .env)
cp .env.example .env
# Editar .env con tus valores de desarrollo

# 3. Servicios locales (Docker)
docker compose up -d redis postgres

# 4. Base de datos
npx prisma migrate dev

# 5. Ejecutar el bot en modo desarrollo
npm run dev
```

---

## Secrets Requeridos (GitHub Repository Secrets)

| Secret | Entorno | Descripción |
|--------|---------|-------------|
| `AZURE_OPENAI_ENDPOINT_TEST` | CI | Endpoint de Azure OpenAI para tests |
| `AZURE_OPENAI_KEY_TEST` | CI | Key de Azure OpenAI para tests |
| `AZURE_OPENAI_ENDPOINT_STAGING` | Staging | Endpoint del entorno de staging |
| `AZURE_OPENAI_ENDPOINT_PROD` | Production | Endpoint del entorno de producción |
| `AZURE_CREDENTIALS_STAGING` | Staging | Service principal JSON para Azure staging |
| `AZURE_CREDENTIALS_PRODUCTION` | Production | Service principal JSON para Azure prod |
| `REDIS_URL_STAGING` | Staging | Connection string de Redis en staging |
| `REDIS_URL_PROD` | Production | Connection string de Redis en producción |
| `DATABASE_URL_STAGING` | Staging | Connection string de PostgreSQL en staging |
| `DATABASE_URL_PROD` | Production | Connection string de PostgreSQL en prod |

> ⚠️ El agente **nunca** debe añadir secrets a este listado sin aprobación humana.
> Los secrets se gestionan manualmente en **GitHub Settings → Secrets and variables → Actions**.
