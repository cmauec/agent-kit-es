# Descripción general del Agent SDK

Construye agentes de IA en producción con Claude Code como biblioteca

---

> **Nota:** El Claude Code SDK ha sido renombrado a Claude Agent SDK. Si estás migrando desde el SDK anterior, consulta la [Guía de migración](/docs/en/agent-sdk/migration-guide).

Construye agentes de IA que de forma autónoma lean archivos, ejecuten comandos, busquen en la web, editen código y más. El Agent SDK te brinda las mismas herramientas, el mismo bucle de agente y la misma gestión de contexto que impulsan Claude Code, programables en Python y TypeScript.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def main():
    async for message in query(
        prompt="Find and fix the bug in auth.py",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Bash"]),
    ):
        print(message)  # Claude reads the file, finds the bug, edits it


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and fix the bug in auth.py",
  options: { allowedTools: ["Read", "Edit", "Bash"] }
})) {
  console.log(message); // Claude reads the file, finds the bug, edits it
}
```

El Agent SDK incluye herramientas integradas para leer archivos, ejecutar comandos y editar código, por lo que tu agente puede comenzar a trabajar de inmediato sin que necesites implementar la ejecución de herramientas. Sumérgete en el inicio rápido o explora agentes reales construidos con el SDK:

- [Inicio rápido](./Quickstart.md) — Construye un agente que corrige errores en minutos
- [Agentes de ejemplo](https://github.com/anthropics/claude-agent-sdk-demos) — Asistente de correo electrónico, agente de investigación y más

## Primeros pasos

### 1. Instala el SDK

**TypeScript**
```bash
npm install @anthropic-ai/claude-agent-sdk
```

**Python**
```bash
pip install claude-agent-sdk
```

### 2. Configura tu API key

Obtén una API key desde la [Consola](https://platform.claude.com/), y luego defínela como variable de entorno:

```bash
export ANTHROPIC_API_KEY=your-api-key
```

El SDK también admite autenticación a través de proveedores de API de terceros:

- **Amazon Bedrock**: define la variable de entorno `CLAUDE_CODE_USE_BEDROCK=1` y configura las credenciales de AWS
- **Google Vertex AI**: define la variable de entorno `CLAUDE_CODE_USE_VERTEX=1` y configura las credenciales de Google Cloud
- **Microsoft Azure**: define la variable de entorno `CLAUDE_CODE_USE_FOUNDRY=1` y configura las credenciales de Azure

Consulta las guías de configuración para [Bedrock](https://code.claude.com/docs/en/amazon-bedrock), [Vertex AI](https://code.claude.com/docs/en/google-vertex-ai) o [Azure AI Foundry](https://code.claude.com/docs/en/azure-ai-foundry) para más detalles.

> **Nota:** Salvo aprobación previa, Anthropic no permite que desarrolladores de terceros ofrezcan el inicio de sesión de claude.ai ni sus límites de uso para sus productos, incluidos los agentes construidos sobre el Claude Agent SDK. Utiliza en cambio los métodos de autenticación mediante API key descritos en este documento.

### 3. Ejecuta tu primer agente

Este ejemplo crea un agente que lista los archivos en tu directorio actual usando herramientas integradas.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def main():
    async for message in query(
        prompt="What files are in this directory?",
        options=ClaudeAgentOptions(allowed_tools=["Bash", "Glob"]),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "What files are in this directory?",
  options: { allowedTools: ["Bash", "Glob"] }
})) {
  if ("result" in message) console.log(message.result);
}
```

**¿Listo para construir?** Sigue el [Inicio rápido](./Quickstart.md) para crear un agente que encuentre y corrija errores en minutos.

## Capacidades

Todo lo que hace poderoso a Claude Code está disponible en el SDK:

### Herramientas integradas

Tu agente puede leer archivos, ejecutar comandos y buscar en bases de código de forma inmediata. Las herramientas principales incluyen:

| Herramienta | Qué hace |
|-------------|----------|
| **Read** | Lee cualquier archivo en el directorio de trabajo |
| **Write** | Crea nuevos archivos |
| **Edit** | Realiza ediciones precisas en archivos existentes |
| **Bash** | Ejecuta comandos de terminal, scripts, operaciones de git |
| **Glob** | Busca archivos por patrón (`**/*.ts`, `src/**/*.py`) |
| **Grep** | Busca contenido en archivos con regex |
| **WebSearch** | Busca en la web información actualizada |
| **WebFetch** | Obtiene y analiza el contenido de páginas web |
| **[AskUserQuestion](./Guides/Handle%20approvals%20and%20user%20input.md#handle-clarifying-questions)** | Formula preguntas de aclaración al usuario con opciones de respuesta múltiple |

Este ejemplo crea un agente que busca comentarios TODO en tu base de código:

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def main():
    async for message in query(
        prompt="Find all TODO comments and create a summary",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Glob", "Grep"]),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find all TODO comments and create a summary",
  options: { allowedTools: ["Read", "Glob", "Grep"] }
})) {
  if ("result" in message) console.log(message.result);
}
```

### Hooks

Ejecuta código personalizado en puntos clave del ciclo de vida del agente. Los hooks del SDK utilizan funciones de callback para validar, registrar, bloquear o transformar el comportamiento del agente.

**Hooks disponibles:** `PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, `SessionEnd`, `UserPromptSubmit` y más.

Este ejemplo registra todos los cambios de archivos en un archivo de auditoría:

**Python**
```python
import asyncio
from datetime import datetime
from claude_agent_sdk import query, ClaudeAgentOptions, HookMatcher


async def log_file_change(input_data, tool_use_id, context):
    file_path = input_data.get("tool_input", {}).get("file_path", "unknown")
    with open("./audit.log", "a") as f:
        f.write(f"{datetime.now()}: modified {file_path}\n")
    return {}


async def main():
    async for message in query(
        prompt="Refactor utils.py to improve readability",
        options=ClaudeAgentOptions(
            permission_mode="acceptEdits",
            hooks={
                "PostToolUse": [
                    HookMatcher(matcher="Edit|Write", hooks=[log_file_change])
                ]
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query, HookCallback } from "@anthropic-ai/claude-agent-sdk";
import { appendFile } from "fs/promises";

const logFileChange: HookCallback = async (input) => {
  const filePath = (input as any).tool_input?.file_path ?? "unknown";
  await appendFile("./audit.log", `${new Date().toISOString()}: modified ${filePath}\n`);
  return {};
};

for await (const message of query({
  prompt: "Refactor utils.py to improve readability",
  options: {
    permissionMode: "acceptEdits",
    hooks: {
      PostToolUse: [{ matcher: "Edit|Write", hooks: [logFileChange] }]
    }
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

[Más información sobre hooks →](./Guides/Intercept%20and%20control%20agent%20behavior%20with%20hooks.md)

### Subagentes

Crea agentes especializados para manejar subtareas concretas. Tu agente principal delega el trabajo y los subagentes devuelven los resultados.

Define agentes personalizados con instrucciones especializadas. Incluye `Agent` en `allowedTools` ya que los subagentes se invocan a través de la herramienta Agent:

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition


async def main():
    async for message in query(
        prompt="Use the code-reviewer agent to review this codebase",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep", "Agent"],
            agents={
                "code-reviewer": AgentDefinition(
                    description="Expert code reviewer for quality and security reviews.",
                    prompt="Analyze code quality and suggest improvements.",
                    tools=["Read", "Glob", "Grep"],
                )
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Use the code-reviewer agent to review this codebase",
  options: {
    allowedTools: ["Read", "Glob", "Grep", "Agent"],
    agents: {
      "code-reviewer": {
        description: "Expert code reviewer for quality and security reviews.",
        prompt: "Analyze code quality and suggest improvements.",
        tools: ["Read", "Glob", "Grep"]
      }
    }
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

Los mensajes provenientes del contexto de un subagente incluyen un campo `parent_tool_use_id`, lo que te permite rastrear a qué ejecución de subagente pertenece cada mensaje.

[Más información sobre subagentes →](./Guides/Subagents%20in%20the%20SDK.md)

### MCP

Conéctate a sistemas externos mediante el Model Context Protocol: bases de datos, navegadores, APIs y [cientos más](https://github.com/modelcontextprotocol/servers).

Este ejemplo conecta el [servidor MCP de Playwright](https://github.com/microsoft/playwright-mcp) para dotar a tu agente de capacidades de automatización del navegador:

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def main():
    async for message in query(
        prompt="Open example.com and describe what you see",
        options=ClaudeAgentOptions(
            mcp_servers={
                "playwright": {"command": "npx", "args": ["@playwright/mcp@latest"]}
            }
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Open example.com and describe what you see",
  options: {
    mcpServers: {
      playwright: { command: "npx", args: ["@playwright/mcp@latest"] }
    }
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

[Más información sobre MCP →](./Guides/Connect%20to%20external%20tools%20with%20MCP.md)

### Permisos

Controla exactamente qué herramientas puede usar tu agente. Permite operaciones seguras, bloquea las peligrosas o requiere aprobación para acciones sensibles.

> **Nota:** Para los prompts de aprobación interactiva y la herramienta `AskUserQuestion`, consulta [Gestionar aprobaciones y entrada del usuario](./Guides/Handle%20approvals%20and%20user%20input.md).

Este ejemplo crea un agente de solo lectura que puede analizar pero no modificar el código. `allowed_tools` preaprueba `Read`, `Glob` y `Grep`.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def main():
    async for message in query(
        prompt="Review this code for best practices",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep"],
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Review this code for best practices",
  options: {
    allowedTools: ["Read", "Glob", "Grep"]
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

[Más información sobre permisos →](./Guides/Configure%20permissions.md)

### Sesiones

Mantén el contexto a través de múltiples intercambios. Claude recuerda los archivos leídos, el análisis realizado y el historial de conversación. Reanuda sesiones más tarde o bifúrcalas para explorar diferentes enfoques.

Este ejemplo captura el ID de sesión de la primera consulta y luego lo reanuda para continuar con el contexto completo:

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def main():
    session_id = None

    # First query: capture the session ID
    async for message in query(
        prompt="Read the authentication module",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Glob"]),
    ):
        if hasattr(message, "subtype") and message.subtype == "init":
            session_id = message.session_id

    # Resume with full context from the first query
    async for message in query(
        prompt="Now find all places that call it",  # "it" = auth module
        options=ClaudeAgentOptions(resume=session_id),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

let sessionId: string | undefined;

// First query: capture the session ID
for await (const message of query({
  prompt: "Read the authentication module",
  options: { allowedTools: ["Read", "Glob"] }
})) {
  if (message.type === "system" && message.subtype === "init") {
    sessionId = message.session_id;
  }
}

// Resume with full context from the first query
for await (const message of query({
  prompt: "Now find all places that call it", // "it" = auth module
  options: { resume: sessionId }
})) {
  if ("result" in message) console.log(message.result);
}
```

[Más información sobre sesiones →](./Core%20concepts/Work%20with%20sessions.md)

### Funcionalidades de Claude Code

El SDK también admite la configuración basada en el sistema de archivos de Claude Code. Para usar estas funcionalidades, define `setting_sources=["project"]` (Python) o `settingSources: ['project']` (TypeScript) en tus opciones.

| Funcionalidad | Descripción | Ubicación |
|---------------|-------------|-----------|
| [Skills](./Guides/Agent%20Skills%20in%20the%20SDK.md) | Capacidades especializadas definidas en Markdown | `.claude/skills/SKILL.md` |
| [Slash commands](./Guides/Slash%20Commands%20in%20the%20SDK.md) | Comandos personalizados para tareas comunes | `.claude/commands/*.md` |
| [Memory](./Guides/Modifying%20system%20prompts.md) | Contexto e instrucciones del proyecto | `CLAUDE.md` o `.claude/CLAUDE.md` |
| [Plugins](./Guides/Plugins%20in%20the%20SDK.md) | Extiende con comandos, agentes y servidores MCP personalizados | Programático mediante la opción `plugins` |

## Comparación del Agent SDK con otras herramientas de Claude

La plataforma de Claude ofrece múltiples formas de construir con Claude. Así es como encaja el Agent SDK:

### Agent SDK vs Client SDK

El [Anthropic Client SDK](/docs/en/api/client-sdks) te da acceso directo a la API: tú envías prompts e implementas la ejecución de herramientas por tu cuenta. El **Agent SDK** te ofrece Claude con ejecución de herramientas integrada.

Con el Client SDK, tú implementas el bucle de herramientas. Con el Agent SDK, Claude lo gestiona:

**Python**
```python
# Client SDK: You implement the tool loop
response = client.messages.create(...)
while response.stop_reason == "tool_use":
    result = your_tool_executor(response.tool_use)
    response = client.messages.create(tool_result=result, **params)

# Agent SDK: Claude handles tools autonomously
async for message in query(prompt="Fix the bug in auth.py"):
    print(message)
```

**TypeScript**
```typescript
// Client SDK: You implement the tool loop
let response = await client.messages.create({ ...params });
while (response.stop_reason === "tool_use") {
  const result = yourToolExecutor(response.tool_use);
  response = await client.messages.create({ tool_result: result, ...params });
}

// Agent SDK: Claude handles tools autonomously
for await (const message of query({ prompt: "Fix the bug in auth.py" })) {
  console.log(message);
}
```

### Agent SDK vs Claude Code CLI

Las mismas capacidades, diferente interfaz:

| Caso de uso | Mejor opción |
|-------------|-------------|
| Desarrollo interactivo | CLI |
| Pipelines de CI/CD | SDK |
| Aplicaciones personalizadas | SDK |
| Tareas puntuales | CLI |
| Automatización en producción | SDK |

Muchos equipos usan ambos: CLI para el desarrollo diario, SDK para producción. Los flujos de trabajo se trasladan directamente entre ellos.

## Changelog

Consulta el changelog completo para actualizaciones del SDK, correcciones de errores y nuevas funcionalidades:

- **TypeScript SDK**: [ver CHANGELOG.md](https://github.com/anthropics/claude-agent-sdk-typescript/blob/main/CHANGELOG.md)
- **Python SDK**: [ver CHANGELOG.md](https://github.com/anthropics/claude-agent-sdk-python/blob/main/CHANGELOG.md)

## Reportar errores

Si encuentras bugs o problemas con el Agent SDK:

- **TypeScript SDK**: [reportar problemas en GitHub](https://github.com/anthropics/claude-agent-sdk-typescript/issues)
- **Python SDK**: [reportar problemas en GitHub](https://github.com/anthropics/claude-agent-sdk-python/issues)

## Directrices de marca

Para los socios que integran el Claude Agent SDK, el uso de la marca Claude es opcional. Cuando hagas referencia a Claude en tu producto:

**Permitido:**
- "Claude Agent" (preferido para menús desplegables)
- "Claude" (cuando está dentro de un menú ya etiquetado como "Agents")
- "{NombreDeAgente} Powered by Claude" (si ya tienes un nombre de agente existente)

**No permitido:**
- "Claude Code" o "Claude Code Agent"
- Arte ASCII con la marca de Claude Code o elementos visuales que imiten a Claude Code

Tu producto debe mantener su propia identidad de marca y no debe aparentar ser Claude Code ni ningún producto de Anthropic. Para preguntas sobre cumplimiento de marca, contacta al [equipo de ventas](https://www.anthropic.com/contact-sales) de Anthropic.

## Licencia y términos

El uso del Claude Agent SDK está regido por los [Términos de Servicio Comerciales de Anthropic](https://www.anthropic.com/legal/commercial-terms), incluso cuando lo uses para impulsar productos y servicios que pones a disposición de tus propios clientes y usuarios finales, excepto en la medida en que un componente o dependencia específica esté cubierta por una licencia diferente según se indique en el archivo LICENSE de ese componente.

## Próximos pasos

- [Inicio rápido](./Quickstart.md) — Construye un agente que encuentre y corrija errores en minutos
- [Agentes de ejemplo](https://github.com/anthropics/claude-agent-sdk-demos) — Asistente de correo electrónico, agente de investigación y más
- [TypeScript SDK](/docs/en/agent-sdk/typescript) — Referencia completa de la API de TypeScript y ejemplos
- [Python SDK](/docs/en/agent-sdk/python) — Referencia completa de la API de Python y ejemplos
