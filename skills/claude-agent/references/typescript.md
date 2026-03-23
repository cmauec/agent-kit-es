# Referencia TypeScript — Claude Agent SDK

## Índice
1. [Setup del proyecto](#setup)
2. [Agente básico](#agente-basico)
3. [Opciones de configuración](#opciones)
4. [Manejo de mensajes](#mensajes)
5. [Sesiones multi-turno](#sesiones)
6. [Subagentes](#subagentes)
7. [Herramientas custom (MCP)](#herramientas-custom)
8. [System prompts](#system-prompts)
9. [Ejemplos por caso de uso](#ejemplos)

---

## Setup del proyecto {#setup}

### Requisitos
- Node.js 18+
- Una API key de Anthropic (obtener en https://platform.claude.com/)

### Instalación
```bash
mkdir mi-agente && cd mi-agente
npm init -y
npm install @anthropic-ai/claude-agent-sdk
npm install -D tsx typescript @types/node
```

### Archivo .env
```bash
ANTHROPIC_API_KEY=tu-api-key-aqui
```

### tsconfig.json (opcional pero recomendado)
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true
  }
}
```

---

## Agente básico {#agente-basico}

El patrón mínimo: importar `query`, definir opciones, iterar mensajes con `for await`.

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// El loop agentico: Claude trabaja hasta completar la tarea
for await (const message of query({
  prompt: "Revisa los archivos .ts de este directorio y dime qué hace cada uno.",
  options: {
    allowedTools: ["Read", "Glob", "Grep"],
    permissionMode: "default",
  },
})) {
  // Respuesta y razonamiento de Claude
  if (message.type === "assistant" && message.message?.content) {
    for (const block of message.message.content) {
      if ("text" in block) {
        process.stdout.write(block.text);
      } else if ("name" in block) {
        console.log(`\nUsando herramienta: ${block.name}`);
      }
    }
  }

  // Resultado final
  if (message.type === "result") {
    console.log(`\nEstado: ${message.subtype}`);
    console.log(`Resultado: ${message.result}`);
  }
}
```

### Ejecutar
```bash
npx tsx agent.ts
```

---

## Opciones de configuración {#opciones}

El objeto `options` acepta estos parámetros clave:

```typescript
const options = {
  // Herramientas que Claude puede usar
  allowedTools: ["Read", "Edit", "Glob", "Grep", "Bash", "WebSearch", "WebFetch"],

  // Herramientas explícitamente bloqueadas (opcional)
  disallowedTools: ["Bash"],

  // Modo de permisos:
  // "default"           → pide confirmación para acciones sensibles
  // "acceptEdits"       → aprueba ediciones de archivo automáticamente
  // "bypassPermissions" → ejecuta todo sin preguntar (solo en entornos seguros)
  // "dontAsk"           → deniega todo lo que no esté en allowedTools
  permissionMode: "acceptEdits" as const,

  // Prompt de sistema que define el rol del agente (opcional)
  systemPrompt: "Eres un experto en TypeScript. Siempre usa tipos explícitos.",

  // Modelo a usar
  model: "claude-sonnet-4-6",

  // Número máximo de turnos del loop (opcional)
  maxTurns: 10,

  // Presupuesto máximo en USD (opcional)
  maxBudgetUsd: 0.50,
};
```

### Herramientas disponibles

| Herramienta | Qué hace |
|-------------|----------|
| `Read` | Lee archivos |
| `Write` | Crea archivos nuevos |
| `Edit` | Modifica archivos existentes |
| `Glob` | Busca archivos por patrón |
| `Grep` | Busca contenido dentro de archivos |
| `Bash` | Ejecuta comandos en la terminal |
| `WebSearch` | Busca en internet |
| `WebFetch` | Descarga el contenido de una URL |
| `AskUserQuestion` | Hace preguntas al usuario durante la ejecución |
| `Agent` | Invoca subagentes (requerido para usar `agents:`) |

---

## Manejo de mensajes {#mensajes}

El loop emite distintos tipos de mensajes. Filtra por `message.type`:

```typescript
import { query, type SDKMessage } from "@anthropic-ai/claude-agent-sdk";

let sessionId: string | undefined;

for await (const message of query({ prompt: "...", options })) {
  switch (message.type) {
    case "assistant":
      // Respuesta y razonamiento de Claude
      for (const block of message.message?.content ?? []) {
        if ("text" in block) {
          process.stdout.write(block.text);
        } else if ("name" in block) {
          console.log(`\nUsando herramienta: ${block.name}`);
        }
      }
      break;

    case "result":
      // Resultado final con métricas
      console.log(`Estado: ${message.subtype}`);    // "success" o "error_*"
      console.log(`Resultado: ${message.result}`);
      sessionId = message.session_id;               // Guardar para continuar después
      break;

    case "system":
      // Mensajes de sistema (inicialización, etc.)
      break;
  }
}
```

---

## Sesiones multi-turno {#sesiones}

Para agentes conversacionales que mantienen contexto entre prompts.

### Con continue: true (automático)
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Primera consulta — crea la sesión
for await (const message of query({
  prompt: "Analiza el módulo de autenticación",
  options: { allowedTools: ["Read", "Glob", "Grep"] },
})) {
  if (message.type === "result") console.log(message.result);
}

// Segunda consulta — continúa la sesión más reciente automáticamente
for await (const message of query({
  prompt: "Ahora refactorízalo para usar JWT",
  options: {
    continue: true,  // busca y retoma la sesión más reciente
    allowedTools: ["Read", "Edit", "Write", "Glob", "Grep"],
  },
})) {
  if (message.type === "result") console.log(message.result);
}
```

### Retomar sesión por ID
```typescript
// Primera ejecución — captura el session_id
let sessionId: string | undefined;

for await (const message of query({ prompt: "Analiza el código", options })) {
  if (message.type === "result") {
    sessionId = message.session_id;
  }
}

// Segunda ejecución — retoma donde quedó
if (sessionId) {
  for await (const message of query({
    prompt: "Ahora implementa las mejoras que sugeriste",
    options: {
      resume: sessionId,
      allowedTools: ["Read", "Edit", "Write"],
    },
  })) {
    if (message.type === "result") console.log(message.result);
  }
}
```

### Hacer fork de una sesión
```typescript
// Explorar una dirección alternativa sin perder la original
for await (const message of query({
  prompt: "En lugar de JWT, implementa OAuth2",
  options: {
    resume: sessionId,
    forkSession: true,  // crea una rama nueva, la original no cambia
  },
})) {
  if (message.type === "result") {
    const forkedId = message.session_id;  // ID de la sesión forkeada
  }
}
```

---

## Subagentes {#subagentes}

Los subagentes son instancias separadas que el agente principal puede lanzar para tareas especializadas.

```typescript
import { query, type AgentDefinition } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Revisa el módulo de autenticación: busca problemas de seguridad y también corre los tests",
  options: {
    // "Agent" es obligatorio para poder invocar subagentes
    allowedTools: ["Read", "Grep", "Glob", "Agent"],
    agents: {
      "revisor-seguridad": {
        description: "Especialista en seguridad. Úsalo para revisar código en busca de vulnerabilidades.",
        prompt: `Eres un experto en seguridad de aplicaciones.
Analiza el código buscando:
- Vulnerabilidades de inyección (SQL, comandos, etc.)
- Manejo inseguro de credenciales o tokens
- Problemas de autenticación y autorización
Sé específico en tus hallazgos.`,
        tools: ["Read", "Grep", "Glob"],  // solo lectura
        model: "sonnet",
      } satisfies AgentDefinition,

      "ejecutor-tests": {
        description: "Ejecutor de tests. Úsalo para correr la suite de pruebas y analizar resultados.",
        prompt: "Ejecuta los tests del proyecto y reporta cuáles fallan y por qué.",
        tools: ["Bash", "Read", "Grep"],  // puede ejecutar comandos
      } satisfies AgentDefinition,
    },
  },
})) {
  if ("result" in message) console.log(message.result);
}
```

> **Nota:** Los subagentes no pueden lanzar sus propios subagentes. No incluyas `Agent` en el `tools` de un subagente.

---

## Herramientas custom (MCP) {#herramientas-custom}

Extiende al agente con tus propias herramientas usando el protocolo MCP integrado.

```typescript
import { query, tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

// Define una herramienta con validación de tipos via Zod
const servidorCustom = createSdkMcpServer({
  name: "mis-herramientas",
  version: "1.0.0",
  tools: [
    tool(
      "obtener_clima",
      "Obtiene la temperatura actual para una ubicación",
      {
        ciudad: z.string().describe("Nombre de la ciudad"),
        pais: z.string().describe("Código de país (ej: ES, MX, AR)"),
      },
      async (args) => {
        // Tu lógica aquí — llama APIs, bases de datos, lo que necesites
        return {
          content: [
            { type: "text", text: `Clima en ${args.ciudad}: 22°C, soleado` },
          ],
        };
      }
    ),
  ],
});

// Las herramientas custom requieren streaming input (generador async)
async function* generarMensajes() {
  yield {
    type: "user" as const,
    message: {
      role: "user" as const,
      content: "¿Qué clima hace en Madrid, España?",
    },
  };
}

for await (const message of query({
  prompt: generarMensajes(),  // generador, no string
  options: {
    mcpServers: { "mis-herramientas": servidorCustom },
    // El nombre de la herramienta sigue el formato: mcp__{servidor}__{herramienta}
    allowedTools: ["mcp__mis-herramientas__obtener_clima"],
    maxTurns: 3,
  },
})) {
  if (message.type === "result") console.log(message.result);
}
```

---

## System prompts {#system-prompts}

El system prompt define la personalidad y restricciones del agente:

```typescript
// Agente de análisis de código
const systemPrompt = `Eres un senior developer con 10 años de experiencia.
Cuando analices código:
- Identifica problemas de rendimiento y seguridad
- Sugiere mejoras concretas con ejemplos de código
- Explica el "por qué" de cada sugerencia`;

// Agente de documentación
const systemPrompt = `Eres un technical writer experto.
Escribe documentación clara y concisa:
- Usa ejemplos de código para cada concepto
- Estructura con headers jerárquicos (H2, H3)
- Escribe en español salvo que se indique lo contrario`;

// Agente de automatización
const systemPrompt = `Eres un asistente de automatización.
- Antes de modificar archivos, muestra un resumen de los cambios planeados
- Crea backups antes de editar archivos críticos
- Reporta cada acción completada con ✓`;
```

---

## Ejemplos por caso de uso {#ejemplos}

### Agente de análisis de código
```typescript
for await (const message of query({
  prompt: "Revisa todos los archivos TypeScript y lista los problemas que encuentres.",
  options: {
    allowedTools: ["Read", "Glob", "Grep"],
    permissionMode: "default",
    systemPrompt: "Eres un revisor de código experto. Analiza sin modificar.",
  },
})) {
  if (message.type === "result") console.log(message.result);
}
```

### Agente de corrección automática
```typescript
for await (const message of query({
  prompt: "Corrige todos los bugs en utils.ts y añade manejo de errores.",
  options: {
    allowedTools: ["Read", "Edit", "Glob", "Bash"],
    permissionMode: "acceptEdits",
    systemPrompt: "Eres un developer TypeScript. Siempre usa tipos explícitos.",
  },
})) {
  if (message.type === "result") console.log(message.result);
}
```

### Agente con búsqueda web
```typescript
for await (const message of query({
  prompt: "Investiga las mejores prácticas de autenticación JWT en 2024 y crea un resumen en SECURITY.md",
  options: {
    allowedTools: ["Read", "WebSearch", "WebFetch", "Write"],
    permissionMode: "acceptEdits",
  },
})) {
  if (message.type === "result") console.log(message.result);
}
```

### Agente de automatización de terminal
```typescript
for await (const message of query({
  prompt: "Instala las dependencias, corre los tests y genera el reporte de cobertura.",
  options: {
    allowedTools: ["Read", "Edit", "Bash", "Glob"],
    permissionMode: "bypassPermissions",  // solo en entornos controlados
    systemPrompt: "Eres un DevOps engineer. Automatiza el setup del proyecto.",
  },
})) {
  if (message.type === "result") console.log(message.result);
}
```
