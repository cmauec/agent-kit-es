# Intercept and control agent behavior with hooks

Intercepta y personaliza el comportamiento del agente en puntos clave de ejecución mediante hooks

---

Los hooks son funciones de callback que ejecutan tu código en respuesta a eventos del agente, como la llamada a una herramienta, el inicio de una sesión o la detención de la ejecución. Con los hooks puedes:

- **Bloquear operaciones peligrosas** antes de que se ejecuten, como comandos de shell destructivos o accesos no autorizados a archivos
- **Registrar y auditar** cada llamada a herramienta para cumplimiento normativo, depuración o analítica
- **Transformar entradas y salidas** para sanear datos, inyectar credenciales o redirigir rutas de archivos
- **Requerir aprobación humana** para acciones sensibles como escrituras en bases de datos o llamadas a APIs
- **Rastrear el ciclo de vida de la sesión** para gestionar estado, liberar recursos o enviar notificaciones

Esta guía explica cómo funcionan los hooks, cómo configurarlos y ofrece ejemplos de patrones comunes como bloquear herramientas, modificar entradas y reenviar notificaciones.

## Cómo funcionan los hooks

### 1. Se dispara un evento

Algo ocurre durante la ejecución del agente y el SDK dispara un evento: una herramienta está a punto de ser llamada (`PreToolUse`), una herramienta devolvió un resultado (`PostToolUse`), un subagente inició o se detuvo, el agente está inactivo o la ejecución finalizó. Consulta la [lista completa de eventos](#available-hooks).

### 2. El SDK recopila los hooks registrados

El SDK verifica si hay hooks registrados para ese tipo de evento. Esto incluye los hooks de callback que pasas en `options.hooks` y los hooks de comandos de shell definidos en archivos de configuración, pero solo si los cargas explícitamente con [`settingSources`](/docs/en/agent-sdk/typescript#setting-source) o [`setting_sources`](/docs/en/agent-sdk/python#setting-source).

### 3. Los matchers filtran qué hooks se ejecutan

Si un hook tiene un patrón [`matcher`](#matchers) (como `"Write|Edit"`), el SDK lo evalúa contra el objetivo del evento (por ejemplo, el nombre de la herramienta). Los hooks sin matcher se ejecutan para cada evento de ese tipo.

### 4. Se ejecutan las funciones de callback

La [función de callback](#callback-functions) de cada hook que coincide recibe información sobre lo que está ocurriendo: el nombre de la herramienta, sus argumentos, el ID de sesión y otros detalles específicos del evento.

### 5. Tu callback devuelve una decisión

Tras realizar cualquier operación (registro, llamadas a APIs, validación), tu callback devuelve un [objeto de salida](#outputs) que le indica al agente qué hacer: permitir la operación, bloquearla, modificar la entrada o inyectar contexto en la conversación.

El siguiente ejemplo reúne estos pasos. Registra un hook `PreToolUse` (paso 1) con un matcher `"Write|Edit"` (paso 3) para que el callback solo se dispare con herramientas de escritura de archivos. Cuando se activa, el callback recibe la entrada de la herramienta (paso 4), comprueba si la ruta del archivo apunta a un archivo `.env` y devuelve `permissionDecision: "deny"` para bloquear la operación (paso 5):

**Python**
```python
import asyncio
from claude_agent_sdk import (
    AssistantMessage,
    ClaudeSDKClient,
    ClaudeAgentOptions,
    HookMatcher,
    ResultMessage,
)


# Define a hook callback that receives tool call details
async def protect_env_files(input_data, tool_use_id, context):
    # Extract the file path from the tool's input arguments
    file_path = input_data["tool_input"].get("file_path", "")
    file_name = file_path.split("/")[-1]

    # Block the operation if targeting a .env file
    if file_name == ".env":
        return {
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "deny",
                "permissionDecisionReason": "Cannot modify .env files",
            }
        }

    # Return empty object to allow the operation
    return {}


async def main():
    options = ClaudeAgentOptions(
        hooks={
            # Register the hook for PreToolUse events
            # The matcher filters to only Write and Edit tool calls
            "PreToolUse": [HookMatcher(matcher="Write|Edit", hooks=[protect_env_files])]
        }
    )

    async with ClaudeSDKClient(options=options) as client:
        await client.query("Update the database configuration")
        async for message in client.receive_response():
            # Filter for assistant and result messages
            if isinstance(message, (AssistantMessage, ResultMessage)):
                print(message)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query, HookCallback, PreToolUseHookInput } from "@anthropic-ai/claude-agent-sdk";

// Define a hook callback with the HookCallback type
const protectEnvFiles: HookCallback = async (input, toolUseID, { signal }) => {
  // Cast input to the specific hook type for type safety
  const preInput = input as PreToolUseHookInput;

  // Cast tool_input to access its properties (typed as unknown in the SDK)
  const toolInput = preInput.tool_input as Record<string, unknown>;
  const filePath = toolInput?.file_path as string;
  const fileName = filePath?.split("/").pop();

  // Block the operation if targeting a .env file
  if (fileName === ".env") {
    return {
      hookSpecificOutput: {
        hookEventName: preInput.hook_event_name,
        permissionDecision: "deny",
        permissionDecisionReason: "Cannot modify .env files"
      }
    };
  }

  // Return empty object to allow the operation
  return {};
};

for await (const message of query({
  prompt: "Update the database configuration",
  options: {
    hooks: {
      // Register the hook for PreToolUse events
      // The matcher filters to only Write and Edit tool calls
      PreToolUse: [{ matcher: "Write|Edit", hooks: [protectEnvFiles] }]
    }
  }
})) {
  // Filter for assistant and result messages
  if (message.type === "assistant" || message.type === "result") {
    console.log(message);
  }
}
```

## Hooks disponibles

El SDK proporciona hooks para distintas etapas de la ejecución del agente. Algunos hooks están disponibles en ambos SDKs, mientras que otros son exclusivos de TypeScript.

| Evento del hook | Python SDK | TypeScript SDK | Qué lo activa | Caso de uso de ejemplo |
|----------------|------------|----------------|---------------|------------------------|
| `PreToolUse` | Sí | Sí | Solicitud de llamada a herramienta (puede bloquear o modificar) | Bloquear comandos de shell peligrosos |
| `PostToolUse` | Sí | Sí | Resultado de ejecución de herramienta | Registrar todos los cambios de archivos en un registro de auditoría |
| `PostToolUseFailure` | Sí | Sí | Fallo en la ejecución de herramienta | Gestionar o registrar errores de herramientas |
| `UserPromptSubmit` | Sí | Sí | Envío de prompt del usuario | Inyectar contexto adicional en los prompts |
| `Stop` | Sí | Sí | Detención de la ejecución del agente | Guardar el estado de la sesión antes de salir |
| `SubagentStart` | Sí | Sí | Inicialización de subagente | Rastrear el inicio de tareas paralelas |
| `SubagentStop` | Sí | Sí | Finalización de subagente | Agregar resultados de tareas paralelas |
| `PreCompact` | Sí | Sí | Solicitud de compactación de conversación | Archivar la transcripción completa antes de resumir |
| `PermissionRequest` | Sí | Sí | Se mostraría un diálogo de permisos | Gestión personalizada de permisos |
| `SessionStart` | No | Sí | Inicialización de sesión | Inicializar registro y telemetría |
| `SessionEnd` | No | Sí | Terminación de sesión | Limpiar recursos temporales |
| `Notification` | Sí | Sí | Mensajes de estado del agente | Enviar actualizaciones de estado del agente a Slack o PagerDuty |
| `Setup` | No | Sí | Configuración/mantenimiento de sesión | Ejecutar tareas de inicialización |
| `TeammateIdle` | No | Sí | Un compañero queda inactivo | Reasignar trabajo o notificar |
| `TaskCompleted` | No | Sí | Tarea en segundo plano completada | Agregar resultados de tareas paralelas |
| `ConfigChange` | No | Sí | Cambios en el archivo de configuración | Recargar configuración dinámicamente |
| `WorktreeCreate` | No | Sí | Git worktree creado | Rastrear espacios de trabajo aislados |
| `WorktreeRemove` | No | Sí | Git worktree eliminado | Limpiar recursos del espacio de trabajo |

## Configurar hooks

Para configurar un hook, pásalo en el campo `hooks` de las opciones de tu agente (`ClaudeAgentOptions` en Python, el objeto `options` en TypeScript):

**Python**
```python
options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher="Bash", hooks=[my_callback])]}
)

async with ClaudeSDKClient(options=options) as client:
    await client.query("Your prompt")
    async for message in client.receive_response():
        print(message)
```

**TypeScript**
```typescript
for await (const message of query({
  prompt: "Your prompt",
  options: {
    hooks: {
      PreToolUse: [{ matcher: "Bash", hooks: [myCallback] }]
    }
  }
})) {
  console.log(message);
}
```

La opción `hooks` es un diccionario (Python) u objeto (TypeScript) donde:
- Las **claves** son [nombres de eventos de hook](#available-hooks) (p. ej., `'PreToolUse'`, `'PostToolUse'`, `'Stop'`)
- Los **valores** son arrays de [matchers](#matchers), cada uno con un patrón de filtro opcional y tus [funciones de callback](#callback-functions)

### Matchers

Usa matchers para filtrar cuándo se disparan tus callbacks. El campo `matcher` es una cadena de regex que se evalúa contra un valor distinto según el tipo de evento de hook. Por ejemplo, los hooks basados en herramientas coinciden con el nombre de la herramienta, mientras que los hooks `Notification` coinciden con el tipo de notificación. Consulta la [referencia de hooks de Claude Code](https://code.claude.com/docs/en/hooks#matcher-patterns) para ver la lista completa de valores de matcher por tipo de evento.

| Opción | Tipo | Valor por defecto | Descripción |
|--------|------|-------------------|-------------|
| `matcher` | `string` | `undefined` | Patrón regex evaluado contra el campo de filtro del evento. Para hooks de herramientas, este es el nombre de la herramienta. Las herramientas integradas incluyen `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `WebFetch`, `Agent` y otras (consulta [Tool Input Types](/docs/en/agent-sdk/typescript#tool-input-types) para la lista completa). Las herramientas MCP usan el patrón `mcp__<server>__<action>`. |
| `hooks` | `HookCallback[]` | - | Obligatorio. Array de funciones de callback a ejecutar cuando el patrón coincide |
| `timeout` | `number` | `60` | Tiempo de espera en segundos |

Usa el patrón `matcher` para apuntar a herramientas específicas siempre que sea posible. Un matcher con `'Bash'` solo se ejecuta para comandos Bash, mientras que omitir el patrón ejecuta tus callbacks para cada ocurrencia del evento. Ten en cuenta que para los hooks basados en herramientas, los matchers solo filtran por **nombre de herramienta**, no por rutas de archivo u otros argumentos. Para filtrar por ruta de archivo, comprueba `tool_input.file_path` dentro de tu callback.

> **Consejo:** **Descubrir nombres de herramientas:** Consulta [Tool Input Types](/docs/en/agent-sdk/typescript#tool-input-types) para la lista completa de nombres de herramientas integradas, o añade un hook sin matcher para registrar todas las llamadas a herramientas que realiza tu sesión.
>
> **Nomenclatura de herramientas MCP:** Las herramientas MCP siempre comienzan con `mcp__` seguido del nombre del servidor y la acción: `mcp__<server>__<action>`. Por ejemplo, si configuras un servidor llamado `playwright`, sus herramientas se llamarán `mcp__playwright__browser_screenshot`, `mcp__playwright__browser_click`, etc. El nombre del servidor proviene de la clave que usas en la configuración de `mcpServers`.

### Funciones de callback

#### Entradas

Cada callback de hook recibe tres argumentos:

- **Datos de entrada:** un objeto tipado con los detalles del evento. Cada tipo de hook tiene su propia forma de entrada (por ejemplo, `PreToolUseHookInput` incluye `tool_name` y `tool_input`, mientras que `NotificationHookInput` incluye `message`). Consulta las definiciones de tipo completas en las referencias del SDK de [TypeScript](/docs/en/agent-sdk/typescript#hook-input) y [Python](/docs/en/agent-sdk/python#hook-input).
  - Todas las entradas de hook comparten `session_id`, `cwd` y `hook_event_name`.
  - `agent_id` y `agent_type` se rellenan cuando el hook se dispara dentro de un subagente. En TypeScript, estos están en la entrada base del hook y están disponibles para todos los tipos de hook. En Python, solo están en `PreToolUse`, `PostToolUse` y `PostToolUseFailure`.
- **Tool use ID** (`str | None` / `string | undefined`): correlaciona los eventos `PreToolUse` y `PostToolUse` para la misma llamada a herramienta.
- **Context:** en TypeScript, contiene una propiedad `signal` (`AbortSignal`) para la cancelación. En Python, este argumento está reservado para uso futuro.

#### Salidas

Tu callback devuelve un objeto con dos categorías de campos:

- Los **campos de nivel superior** controlan la conversación: `systemMessage` inyecta un mensaje en la conversación visible para el modelo, y `continue` (`continue_` en Python) determina si el agente sigue ejecutándose después de este hook.
- **`hookSpecificOutput`** controla la operación actual. Los campos internos dependen del tipo de evento del hook. Para los hooks `PreToolUse`, aquí es donde estableces `permissionDecision` (`"allow"`, `"deny"` o `"ask"`), `permissionDecisionReason` y `updatedInput`. Para los hooks `PostToolUse`, puedes establecer `additionalContext` para añadir información al resultado de la herramienta.

Devuelve `{}` para permitir la operación sin cambios. Los hooks de callback del SDK usan el mismo formato de salida JSON que los [hooks de comandos de shell de Claude Code](https://code.claude.com/docs/en/hooks#json-output), que documenta cada campo y opción específica de evento. Para las definiciones de tipo del SDK, consulta las referencias de [TypeScript](/docs/en/agent-sdk/typescript#sync-hook-json-output) y [Python](/docs/en/agent-sdk/python#sync-hook-json-output).

> **Nota:** Cuando se aplican múltiples hooks o reglas de permisos, **deny** tiene prioridad sobre **ask**, que tiene prioridad sobre **allow**. Si algún hook devuelve `deny`, la operación se bloquea independientemente de los demás hooks.

#### Salida asíncrona

Por defecto, el agente espera a que tu hook devuelva un resultado antes de continuar. Si tu hook realiza un efecto secundario (registro, envío de un webhook) y no necesita influir en el comportamiento del agente, puedes devolver una salida asíncrona en su lugar. Esto le indica al agente que continúe inmediatamente sin esperar a que el hook finalice:

**Python**
```python
async def async_hook(input_data, tool_use_id, context):
    # Start a background task, then return immediately
    asyncio.create_task(send_to_logging_service(input_data))
    return {"async_": True, "asyncTimeout": 30000}
```

**TypeScript**
```typescript
const asyncHook: HookCallback = async (input, toolUseID, { signal }) => {
  // Start a background task, then return immediately
  sendToLoggingService(input).catch(console.error);
  return { async: true, asyncTimeout: 30000 };
};
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `async` | `true` | Indica modo asíncrono. El agente continúa sin esperar. En Python, usa `async_` para evitar la palabra reservada. |
| `asyncTimeout` | `number` | Tiempo de espera opcional en milisegundos para la operación en segundo plano |

> **Nota:** Las salidas asíncronas no pueden bloquear, modificar ni inyectar contexto en la operación, ya que el agente ya ha continuado. Úsalas únicamente para efectos secundarios como registro, métricas o notificaciones.

## Ejemplos

### Modificar la entrada de una herramienta

Este ejemplo intercepta las llamadas a la herramienta Write y reescribe el argumento `file_path` para anteponer `/sandbox`, redirigiendo todas las escrituras de archivos a un directorio en sandbox. El callback devuelve `updatedInput` con la ruta modificada y `permissionDecision: 'allow'` para aprobar automáticamente la operación reescrita:

**Python**
```python
async def redirect_to_sandbox(input_data, tool_use_id, context):
    if input_data["hook_event_name"] != "PreToolUse":
        return {}

    if input_data["tool_name"] == "Write":
        original_path = input_data["tool_input"].get("file_path", "")
        return {
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "allow",
                "updatedInput": {
                    **input_data["tool_input"],
                    "file_path": f"/sandbox{original_path}",
                },
            }
        }
    return {}
```

**TypeScript**
```typescript
const redirectToSandbox: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name !== "PreToolUse") return {};

  const preInput = input as PreToolUseHookInput;
  const toolInput = preInput.tool_input as Record<string, unknown>;
  if (preInput.tool_name === "Write") {
    const originalPath = toolInput.file_path as string;
    return {
      hookSpecificOutput: {
        hookEventName: preInput.hook_event_name,
        permissionDecision: "allow",
        updatedInput: {
          ...toolInput,
          file_path: `/sandbox${originalPath}`
        }
      }
    };
  }
  return {};
};
```

> **Nota:** Al usar `updatedInput`, también debes incluir `permissionDecision: 'allow'`. Devuelve siempre un nuevo objeto en lugar de mutar el `tool_input` original.

### Añadir contexto y bloquear una herramienta

Este ejemplo bloquea cualquier intento de escribir en el directorio `/etc` y usa dos campos de salida juntos: `permissionDecision: 'deny'` detiene la llamada a la herramienta, mientras que `systemMessage` inyecta un recordatorio en la conversación para que el agente reciba contexto sobre por qué se bloqueó la operación y evite reintentarla:

**Python**
```python
async def block_etc_writes(input_data, tool_use_id, context):
    file_path = input_data["tool_input"].get("file_path", "")

    if file_path.startswith("/etc"):
        return {
            # Top-level field: inject guidance into the conversation
            "systemMessage": "Remember: system directories like /etc are protected.",
            # hookSpecificOutput: block the operation
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "deny",
                "permissionDecisionReason": "Writing to /etc is not allowed",
            },
        }
    return {}
```

**TypeScript**
```typescript
const blockEtcWrites: HookCallback = async (input, toolUseID, { signal }) => {
  const preInput = input as PreToolUseHookInput;
  const toolInput = preInput.tool_input as Record<string, unknown>;
  const filePath = toolInput?.file_path as string;

  if (filePath?.startsWith("/etc")) {
    return {
      // Top-level field: inject guidance into the conversation
      systemMessage: "Remember: system directories like /etc are protected.",
      // hookSpecificOutput: block the operation
      hookSpecificOutput: {
        hookEventName: preInput.hook_event_name,
        permissionDecision: "deny",
        permissionDecisionReason: "Writing to /etc is not allowed"
      }
    };
  }
  return {};
};
```

### Aprobar automáticamente herramientas específicas

Por defecto, el agente puede solicitar permiso antes de usar ciertas herramientas. Este ejemplo aprueba automáticamente las herramientas de solo lectura del sistema de archivos (Read, Glob, Grep) devolviendo `permissionDecision: 'allow'`, permitiéndoles ejecutarse sin confirmación del usuario mientras deja todas las demás herramientas sujetas a las comprobaciones de permisos normales:

**Python**
```python
async def auto_approve_read_only(input_data, tool_use_id, context):
    if input_data["hook_event_name"] != "PreToolUse":
        return {}

    read_only_tools = ["Read", "Glob", "Grep"]
    if input_data["tool_name"] in read_only_tools:
        return {
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "allow",
                "permissionDecisionReason": "Read-only tool auto-approved",
            }
        }
    return {}
```

**TypeScript**
```typescript
const autoApproveReadOnly: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name !== "PreToolUse") return {};

  const preInput = input as PreToolUseHookInput;
  const readOnlyTools = ["Read", "Glob", "Grep"];
  if (readOnlyTools.includes(preInput.tool_name)) {
    return {
      hookSpecificOutput: {
        hookEventName: preInput.hook_event_name,
        permissionDecision: "allow",
        permissionDecisionReason: "Read-only tool auto-approved"
      }
    };
  }
  return {};
};
```

### Encadenar múltiples hooks

Los hooks se ejecutan en el orden en que aparecen en el array. Mantén cada hook centrado en una única responsabilidad y encadena múltiples hooks para lógica compleja:

**Python**
```python
options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(hooks=[rate_limiter]),  # First: check rate limits
            HookMatcher(hooks=[authorization_check]),  # Second: verify permissions
            HookMatcher(hooks=[input_sanitizer]),  # Third: sanitize inputs
            HookMatcher(hooks=[audit_logger]),  # Last: log the action
        ]
    }
)
```

**TypeScript**
```typescript
const options = {
  hooks: {
    PreToolUse: [
      { hooks: [rateLimiter] }, // First: check rate limits
      { hooks: [authorizationCheck] }, // Second: verify permissions
      { hooks: [inputSanitizer] }, // Third: sanitize inputs
      { hooks: [auditLogger] } // Last: log the action
    ]
  }
};
```

### Filtrar con matchers de regex

Usa patrones de regex para coincidir con múltiples herramientas. Este ejemplo registra tres matchers con diferentes alcances: el primero activa `file_security_hook` solo para herramientas de modificación de archivos, el segundo activa `mcp_audit_hook` para cualquier herramienta MCP (herramientas cuyos nombres comienzan con `mcp__`), y el tercero activa `global_logger` para cada llamada a herramienta independientemente del nombre:

**Python**
```python
options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            # Match file modification tools
            HookMatcher(matcher="Write|Edit|Delete", hooks=[file_security_hook]),
            # Match all MCP tools
            HookMatcher(matcher="^mcp__", hooks=[mcp_audit_hook]),
            # Match everything (no matcher)
            HookMatcher(hooks=[global_logger]),
        ]
    }
)
```

**TypeScript**
```typescript
const options = {
  hooks: {
    PreToolUse: [
      // Match file modification tools
      { matcher: "Write|Edit|Delete", hooks: [fileSecurityHook] },

      // Match all MCP tools
      { matcher: "^mcp__", hooks: [mcpAuditHook] },

      // Match everything (no matcher)
      { hooks: [globalLogger] }
    ]
  }
};
```

### Rastrear actividad de subagentes

Usa hooks `SubagentStop` para monitorizar cuándo los subagentes finalizan su trabajo. Consulta el tipo de entrada completo en las referencias del SDK de [TypeScript](/docs/en/agent-sdk/typescript#hook-input) y [Python](/docs/en/agent-sdk/python#hook-input). Este ejemplo registra un resumen cada vez que un subagente completa su ejecución:

**Python**
```python
async def subagent_tracker(input_data, tool_use_id, context):
    # Log subagent details when it finishes
    print(f"[SUBAGENT] Completed: {input_data['agent_id']}")
    print(f"  Transcript: {input_data['agent_transcript_path']}")
    print(f"  Tool use ID: {tool_use_id}")
    print(f"  Stop hook active: {input_data.get('stop_hook_active')}")
    return {}


options = ClaudeAgentOptions(
    hooks={"SubagentStop": [HookMatcher(hooks=[subagent_tracker])]}
)
```

**TypeScript**
```typescript
import { HookCallback, SubagentStopHookInput } from "@anthropic-ai/claude-agent-sdk";

const subagentTracker: HookCallback = async (input, toolUseID, { signal }) => {
  // Cast to SubagentStopHookInput to access subagent-specific fields
  const subInput = input as SubagentStopHookInput;

  // Log subagent details when it finishes
  console.log(`[SUBAGENT] Completed: ${subInput.agent_id}`);
  console.log(`  Transcript: ${subInput.agent_transcript_path}`);
  console.log(`  Tool use ID: ${toolUseID}`);
  console.log(`  Stop hook active: ${subInput.stop_hook_active}`);
  return {};
};

const options = {
  hooks: {
    SubagentStop: [{ hooks: [subagentTracker] }]
  }
};
```

### Realizar solicitudes HTTP desde hooks

Los hooks pueden realizar operaciones asíncronas como solicitudes HTTP. Captura los errores dentro de tu hook en lugar de dejar que se propaguen, ya que una excepción no controlada puede interrumpir al agente.

Este ejemplo envía un webhook después de que cada herramienta completa su ejecución, registrando qué herramienta se ejecutó y cuándo. El hook captura los errores para que un webhook fallido no interrumpa al agente:

**Python**
```python
import asyncio
import json
import urllib.request
from datetime import datetime


def _send_webhook(tool_name):
    """Synchronous helper that POSTs tool usage data to an external webhook."""
    data = json.dumps(
        {
            "tool": tool_name,
            "timestamp": datetime.now().isoformat(),
        }
    ).encode()
    req = urllib.request.Request(
        "https://api.example.com/webhook",
        data=data,
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    urllib.request.urlopen(req)


async def webhook_notifier(input_data, tool_use_id, context):
    # Only fire after a tool completes (PostToolUse), not before
    if input_data["hook_event_name"] != "PostToolUse":
        return {}

    try:
        # Run the blocking HTTP call in a thread to avoid blocking the event loop
        await asyncio.to_thread(_send_webhook, input_data["tool_name"])
    except Exception as e:
        # Log the error but don't raise. A failed webhook shouldn't stop the agent
        print(f"Webhook request failed: {e}")

    return {}
```

**TypeScript**
```typescript
import { query, HookCallback, PostToolUseHookInput } from "@anthropic-ai/claude-agent-sdk";

const webhookNotifier: HookCallback = async (input, toolUseID, { signal }) => {
  // Only fire after a tool completes (PostToolUse), not before
  if (input.hook_event_name !== "PostToolUse") return {};

  try {
    await fetch("https://api.example.com/webhook", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        tool: (input as PostToolUseHookInput).tool_name,
        timestamp: new Date().toISOString()
      }),
      // Pass signal so the request cancels if the hook times out
      signal
    });
  } catch (error) {
    // Handle cancellation separately from other errors
    if (error instanceof Error && error.name === "AbortError") {
      console.log("Webhook request cancelled");
    }
    // Don't re-throw. A failed webhook shouldn't stop the agent
  }

  return {};
};

// Register as a PostToolUse hook
for await (const message of query({
  prompt: "Refactor the auth module",
  options: {
    hooks: {
      PostToolUse: [{ hooks: [webhookNotifier] }]
    }
  }
})) {
  console.log(message);
}
```

### Reenviar notificaciones a Slack

Usa hooks `Notification` para recibir notificaciones del sistema del agente y reenviarlas a servicios externos. Las notificaciones se disparan para tipos de evento específicos: `permission_prompt` (Claude necesita permiso), `idle_prompt` (Claude espera entrada), `auth_success` (autenticación completada) y `elicitation_dialog` (Claude está solicitando al usuario). Cada notificación incluye un campo `message` con una descripción legible por humanos y opcionalmente un `title`.

Este ejemplo reenvía cada notificación a un canal de Slack. Requiere una [URL de webhook entrante de Slack](https://api.slack.com/messaging/webhooks), que se crea añadiendo una aplicación a tu espacio de trabajo de Slack y habilitando los webhooks entrantes:

**Python**
```python
import asyncio
import json
import urllib.request

from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, HookMatcher


def _send_slack_notification(message):
    """Synchronous helper that sends a message to Slack via incoming webhook."""
    data = json.dumps({"text": f"Agent status: {message}"}).encode()
    req = urllib.request.Request(
        "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
        data=data,
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    urllib.request.urlopen(req)


async def notification_handler(input_data, tool_use_id, context):
    try:
        # Run the blocking HTTP call in a thread to avoid blocking the event loop
        await asyncio.to_thread(_send_slack_notification, input_data.get("message", ""))
    except Exception as e:
        print(f"Failed to send notification: {e}")

    # Return empty object. Notification hooks don't modify agent behavior
    return {}


async def main():
    options = ClaudeAgentOptions(
        hooks={
            # Register the hook for Notification events (no matcher needed)
            "Notification": [HookMatcher(hooks=[notification_handler])],
        },
    )

    async with ClaudeSDKClient(options=options) as client:
        await client.query("Analyze this codebase")
        async for message in client.receive_response():
            print(message)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query, HookCallback, NotificationHookInput } from "@anthropic-ai/claude-agent-sdk";

// Define a hook callback that sends notifications to Slack
const notificationHandler: HookCallback = async (input, toolUseID, { signal }) => {
  // Cast to NotificationHookInput to access the message field
  const notification = input as NotificationHookInput;

  try {
    // POST the notification message to a Slack incoming webhook
    await fetch("https://hooks.slack.com/services/YOUR/WEBHOOK/URL", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        text: `Agent status: ${notification.message}`
      }),
      // Pass signal so the request cancels if the hook times out
      signal
    });
  } catch (error) {
    if (error instanceof Error && error.name === "AbortError") {
      console.log("Notification cancelled");
    } else {
      console.error("Failed to send notification:", error);
    }
  }

  // Return empty object. Notification hooks don't modify agent behavior
  return {};
};

// Register the hook for Notification events (no matcher needed)
for await (const message of query({
  prompt: "Analyze this codebase",
  options: {
    hooks: {
      Notification: [{ hooks: [notificationHandler] }]
    }
  }
})) {
  console.log(message);
}
```

## Solucionar problemas comunes

### El hook no se dispara

- Verifica que el nombre del evento del hook sea correcto y distingue mayúsculas de minúsculas (`PreToolUse`, no `preToolUse`)
- Comprueba que tu patrón matcher coincida exactamente con el nombre de la herramienta
- Asegúrate de que el hook esté bajo el tipo de evento correcto en `options.hooks`
- Para hooks que no son de herramientas como `Stop` y `SubagentStop`, los matchers coinciden con campos diferentes (consulta [matcher patterns](https://code.claude.com/docs/en/hooks#matcher-patterns))
- Los hooks pueden no dispararse cuando el agente alcanza el límite [`max_turns`](/docs/en/agent-sdk/python#claude-agent-options) porque la sesión termina antes de que los hooks puedan ejecutarse

### El matcher no filtra como se esperaba

Los matchers solo coinciden con **nombres de herramientas**, no con rutas de archivos u otros argumentos. Para filtrar por ruta de archivo, comprueba `tool_input.file_path` dentro de tu hook:

```typescript
const myHook: HookCallback = async (input, toolUseID, { signal }) => {
  const preInput = input as PreToolUseHookInput;
  const toolInput = preInput.tool_input as Record<string, unknown>;
  const filePath = toolInput?.file_path as string;
  if (!filePath?.endsWith(".md")) return {}; // Skip non-markdown files
  // Process markdown files...
  return {};
};
```

### Tiempo de espera del hook agotado

- Aumenta el valor de `timeout` en la configuración de `HookMatcher`
- Usa el `AbortSignal` del tercer argumento del callback para gestionar la cancelación de forma elegante en TypeScript

### Herramienta bloqueada inesperadamente

- Comprueba todos los hooks `PreToolUse` en busca de devoluciones con `permissionDecision: 'deny'`
- Añade registro a tus hooks para ver qué `permissionDecisionReason` están devolviendo
- Verifica que los patrones matcher no sean demasiado amplios (un matcher vacío coincide con todas las herramientas)

### La entrada modificada no se aplica

- Asegúrate de que `updatedInput` esté dentro de `hookSpecificOutput`, no en el nivel superior:

  ```typescript
  return {
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "allow",
      updatedInput: { command: "new command" }
    }
  };
  ```

- También debes devolver `permissionDecision: 'allow'` para que la modificación de la entrada surta efecto
- Incluye `hookEventName` en `hookSpecificOutput` para identificar para qué tipo de hook es la salida

### Hooks de sesión no disponibles en Python

`SessionStart` y `SessionEnd` pueden registrarse como hooks de callback del SDK en TypeScript, pero no están disponibles en el SDK de Python (`HookEvent` los omite). En Python, solo están disponibles como [hooks de comandos de shell](https://code.claude.com/docs/en/hooks#hook-events) definidos en archivos de configuración (por ejemplo, `.claude/settings.json`). Para cargar hooks de comandos de shell desde tu aplicación SDK, incluye la fuente de configuración apropiada con [`setting_sources`](/docs/en/agent-sdk/python#setting-source) o [`settingSources`](/docs/en/agent-sdk/typescript#setting-source):

**Python**
```python
options = ClaudeAgentOptions(
    setting_sources=["project"],  # Loads .claude/settings.json including hooks
)
```

**TypeScript**
```typescript
const options = {
  settingSources: ["project"] // Loads .claude/settings.json including hooks
};
```

Para ejecutar lógica de inicialización como callback del SDK de Python en su lugar, usa el primer mensaje de `client.receive_response()` como disparador.

### Multiplicación de solicitudes de permiso en subagentes

Al generar múltiples subagentes, cada uno puede solicitar permisos por separado. Los subagentes no heredan automáticamente los permisos del agente padre. Para evitar solicitudes repetidas, usa hooks `PreToolUse` para aprobar automáticamente herramientas específicas, o configura reglas de permisos que se apliquen a las sesiones de subagentes.

### Bucles de hooks recursivos con subagentes

Un hook `UserPromptSubmit` que genera subagentes puede crear bucles infinitos si esos subagentes disparan el mismo hook. Para evitarlo:

- Comprueba si hay un indicador de subagente en la entrada del hook antes de generar uno
- Usa una variable compartida o el estado de sesión para rastrear si ya estás dentro de un subagente
- Limita el alcance de los hooks para que solo se ejecuten en la sesión del agente de nivel superior

### systemMessage no aparece en la salida

El campo `systemMessage` añade contexto a la conversación que el modelo puede ver, pero puede que no aparezca en todos los modos de salida del SDK. Si necesitas exponer las decisiones del hook a tu aplicación, regístralas por separado o usa un canal de salida dedicado.

## Recursos relacionados

- [Referencia de hooks de Claude Code](https://code.claude.com/docs/en/hooks): esquemas JSON completos de entrada/salida, documentación de eventos y patrones de matchers
- [Guía de hooks de Claude Code](https://code.claude.com/docs/en/hooks-guide): ejemplos y tutoriales de hooks de comandos de shell
- [Referencia del SDK de TypeScript](/docs/en/agent-sdk/typescript): tipos de hook, definiciones de entrada/salida y opciones de configuración
- [Referencia del SDK de Python](/docs/en/agent-sdk/python): tipos de hook, definiciones de entrada/salida y opciones de configuración
- [Permissions](/docs/en/agent-sdk/permissions): controla lo que puede hacer tu agente
- [Custom tools](/docs/en/agent-sdk/custom-tools): crea herramientas para ampliar las capacidades del agente
