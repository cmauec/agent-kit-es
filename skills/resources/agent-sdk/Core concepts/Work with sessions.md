# Work with sessions

Cómo las sesiones persisten el historial de conversación del agente, y cuándo usar continue, resume y fork para volver a una ejecución anterior.

---

Una sesión es el historial de conversación que el SDK acumula mientras tu agente trabaja. Contiene tu prompt, cada llamada a herramienta que hizo el agente, cada resultado de herramienta y cada respuesta. El SDK la escribe en disco automáticamente para que puedas volver a ella más tarde.

Volver a una sesión significa que el agente tiene el contexto completo de antes: archivos que ya leyó, análisis que ya realizó, decisiones que ya tomó. Puedes hacer una pregunta de seguimiento, recuperarte de una interrupción o ramificarte para probar un enfoque diferente.

> **Nota:** Las sesiones persisten la **conversación**, no el sistema de archivos. Para hacer snapshots y revertir los cambios de archivos que hizo el agente, usa [file checkpointing](../Guides/Rewind%20file%20changes%20with%20checkpointing.md).

Esta guía cubre cómo elegir el enfoque correcto para tu aplicación, las interfaces del SDK que rastrean sesiones automáticamente, cómo capturar IDs de sesión y usar `resume` y `fork` manualmente, y qué tener en cuenta al reanudar sesiones entre hosts.

## Elige un enfoque

La cantidad de manejo de sesiones que necesitas depende de la forma de tu aplicación. La gestión de sesiones entra en juego cuando envías múltiples prompts que deben compartir contexto. Dentro de una sola llamada a `query()`, el agente ya toma todos los turnos que necesita, y los prompts de permisos y `AskUserQuestion` se [gestionan dentro del bucle](../Guides/Handle%20approvals%20and%20user%20input.md) (no terminan la llamada).

| Qué estás construyendo | Qué usar |
|:---|:---|
| Tarea de un solo disparo: un prompt, sin seguimiento | Nada extra. Una sola llamada a `query()` lo maneja. |
| Chat multi-turno en un solo proceso | [`ClaudeSDKClient` (Python) o `continue: true` (TypeScript)](#automatic-session-management). El SDK rastrea la sesión por ti sin manejo de IDs. |
| Continuar donde se dejó después de reiniciar el proceso | `continue_conversation=True` (Python) / `continue: true` (TypeScript). Reanuda la sesión más reciente en el directorio, sin necesidad de ID. |
| Reanudar una sesión pasada específica (no la más reciente) | Captura el ID de sesión y pásalo a `resume`. |
| Probar un enfoque alternativo sin perder el original | Haz un fork de la sesión. |
| Tarea sin estado, no quieres que nada se escriba en disco (solo TypeScript) | Establece `persistSession: false`. La sesión existe solo en memoria durante la duración de la llamada. Python siempre persiste en disco. |

### Continue, resume y fork

Continue, resume y fork son campos de opción que se establecen en `query()` (`ClaudeAgentOptions` en Python, `Options` en TypeScript).

**Continue** y **resume** ambos retoman una sesión existente y le añaden contenido. La diferencia está en cómo encuentran esa sesión:

- **Continue** encuentra la sesión más reciente en el directorio actual. No necesitas rastrear nada. Funciona bien cuando tu aplicación ejecuta una conversación a la vez.
- **Resume** toma un ID de sesión específico. Tú rastresas el ID. Es necesario cuando tienes múltiples sesiones (por ejemplo, una por usuario en una aplicación multi-usuario) o quieres volver a una que no es la más reciente.

**Fork** es diferente: crea una nueva sesión que comienza con una copia del historial del original. El original permanece sin cambios. Usa fork para probar una dirección diferente mientras conservas la opción de regresar.

## Gestión automática de sesiones

Ambos SDKs ofrecen una interfaz que rastrea el estado de la sesión por ti entre llamadas, para que no tengas que pasar IDs manualmente. Úsalas para conversaciones multi-turno dentro de un solo proceso.

### Python: `ClaudeSDKClient`

`ClaudeSDKClient` maneja los IDs de sesión internamente. Cada llamada a `client.query()` continúa automáticamente la misma sesión. Llama a `client.receive_response()` para iterar sobre los mensajes de la consulta actual. El cliente debe usarse como un context manager asíncrono.

Este ejemplo ejecuta dos consultas contra el mismo `client`. La primera le pide al agente que analice un módulo; la segunda le pide que refactorice ese módulo. Como ambas llamadas pasan por la misma instancia del cliente, la segunda consulta tiene el contexto completo de la primera sin ningún `resume` ni ID de sesión explícito:

```python
import asyncio
from claude_agent_sdk import (
    ClaudeSDKClient,
    ClaudeAgentOptions,
    AssistantMessage,
    ResultMessage,
    TextBlock,
)


def print_response(message):
    """Print only the human-readable parts of a message."""
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text)
    elif isinstance(message, ResultMessage):
        cost = (
            f"${message.total_cost_usd:.4f}"
            if message.total_cost_usd is not None
            else "N/A"
        )
        print(f"[done: {message.subtype}, cost: {cost}]")


async def main():
    options = ClaudeAgentOptions(
        allowed_tools=["Read", "Edit", "Glob", "Grep"],
    )

    async with ClaudeSDKClient(options=options) as client:
        # First query: client captures the session ID internally
        await client.query("Analyze the auth module")
        async for message in client.receive_response():
            print_response(message)

        # Second query: automatically continues the same session
        await client.query("Now refactor it to use JWT")
        async for message in client.receive_response():
            print_response(message)


asyncio.run(main())
```

Consulta la referencia del SDK de Python para obtener detalles sobre cuándo usar `ClaudeSDKClient` frente a la función `query()` independiente.

### TypeScript: `continue: true`

El SDK estable de TypeScript (la función `query()` usada a lo largo de esta documentación, a veces llamada V1) no tiene un objeto cliente que mantenga la sesión como el `ClaudeSDKClient` de Python. En su lugar, pasa `continue: true` en cada llamada subsiguiente a `query()` y el SDK retoma la sesión más reciente en el directorio actual. No se requiere rastreo de IDs.

Este ejemplo hace dos llamadas separadas a `query()`. La primera crea una sesión nueva; la segunda establece `continue: true`, lo que le indica al SDK que busque y reanude la sesión más reciente en disco. El agente tiene el contexto completo de la primera llamada:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// First query: creates a new session
for await (const message of query({
  prompt: "Analyze the auth module",
  options: { allowedTools: ["Read", "Glob", "Grep"] }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}

// Second query: continue: true resumes the most recent session
for await (const message of query({
  prompt: "Now refactor it to use JWT",
  options: {
    continue: true,
    allowedTools: ["Read", "Edit", "Write", "Glob", "Grep"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

> **Nota:** También existe una [vista previa V2](/docs/en/agent-sdk/typescript-v2-preview) del SDK de TypeScript que proporciona `createSession()` con un patrón `send` / `stream`, más cercano en concepto al `ClaudeSDKClient` de Python. V2 es inestable y sus APIs pueden cambiar; el resto de esta documentación usa la función estable V1 `query()`.

## Usa las opciones de sesión con `query()`

### Captura el ID de sesión

Resume y fork requieren un ID de sesión. Léelo desde el campo `session_id` en el mensaje de resultado ([`ResultMessage`](/docs/en/agent-sdk/python#result-message) en Python, [`SDKResultMessage`](/docs/en/agent-sdk/typescript#sdk-result-message) en TypeScript), que está presente en cada resultado independientemente del éxito o error. En TypeScript el ID también está disponible antes como un campo directo en el `SystemMessage` de inicio; en Python está anidado dentro de `SystemMessage.data`.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage


async def main():
    session_id = None

    async for message in query(
        prompt="Analyze the auth module and suggest improvements",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep"],
        ),
    ):
        if isinstance(message, ResultMessage):
            session_id = message.session_id
            if message.subtype == "success":
                print(message.result)

    print(f"Session ID: {session_id}")
    return session_id


session_id = asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

let sessionId: string | undefined;

for await (const message of query({
  prompt: "Analyze the auth module and suggest improvements",
  options: { allowedTools: ["Read", "Glob", "Grep"] }
})) {
  if (message.type === "result") {
    sessionId = message.session_id;
    if (message.subtype === "success") {
      console.log(message.result);
    }
  }
}

console.log(`Session ID: ${sessionId}`);
```

### Reanudar por ID

Pasa un ID de sesión a `resume` para volver a esa sesión específica. El agente retoma con el contexto completo desde donde se dejó la sesión. Razones comunes para reanudar:

- **Dar seguimiento a una tarea completada.** El agente ya analizó algo; ahora quieres que actúe sobre ese análisis sin volver a leer archivos.
- **Recuperarse de un límite.** La primera ejecución terminó con `error_max_turns` o `error_max_budget_usd` (consulta [Handle the result](../How%20the%20agent%20loop%20works.md#handle-the-result)); reanuda con un límite más alto.
- **Reiniciar tu proceso.** Capturaste el ID antes del apagado y quieres restaurar la conversación.

Este ejemplo reanuda la sesión de [Captura el ID de sesión](#capture-the-session-id) con un prompt de seguimiento. Como estás reanudando, el agente ya tiene el análisis previo en contexto:

**Python**
```python
# Earlier session analyzed the code; now build on that analysis
async for message in query(
    prompt="Now implement the refactoring you suggested",
    options=ClaudeAgentOptions(
        resume=session_id,
        allowed_tools=["Read", "Edit", "Write", "Glob", "Grep"],
    ),
):
    if isinstance(message, ResultMessage) and message.subtype == "success":
        print(message.result)
```

**TypeScript**
```typescript
// Earlier session analyzed the code; now build on that analysis
for await (const message of query({
  prompt: "Now implement the refactoring you suggested",
  options: {
    resume: sessionId,
    allowedTools: ["Read", "Edit", "Write", "Glob", "Grep"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

> **Tip:** Si una llamada a `resume` devuelve una sesión nueva en lugar del historial esperado, la causa más común es un `cwd` que no coincide. Las sesiones se almacenan en `~/.claude/projects/<encoded-cwd>/*.jsonl`, donde `<encoded-cwd>` es el directorio de trabajo absoluto con cada carácter no alfanumérico reemplazado por `-` (así `/Users/me/proj` se convierte en `-Users-me-proj`). Si tu llamada a resume se ejecuta desde un directorio diferente, el SDK busca en el lugar equivocado. El archivo de sesión también debe existir en la máquina actual.

### Fork para explorar alternativas

Hacer un fork crea una nueva sesión que comienza con una copia del historial del original pero diverge desde ese punto. El fork obtiene su propio ID de sesión; el ID y el historial del original permanecen sin cambios. Terminas con dos sesiones independientes que puedes reanudar por separado.

> **Nota:** El fork ramifica el historial de la conversación, no el sistema de archivos. Si un agente con fork edita archivos, esos cambios son reales y visibles para cualquier sesión que trabaje en el mismo directorio. Para ramificar y revertir cambios de archivos, usa [file checkpointing](../Guides/Rewind%20file%20changes%20with%20checkpointing.md).

Este ejemplo se basa en [Captura el ID de sesión](#capture-the-session-id): ya analizaste un módulo de autenticación en `session_id` y quieres explorar OAuth2 sin perder el hilo enfocado en JWT. El primer bloque hace un fork de la sesión y captura el ID del fork (`forked_id`); el segundo bloque reanuda el `session_id` original para continuar por el camino de JWT. Ahora tienes dos IDs de sesión apuntando a dos historiales separados:

**Python**
```python
# Fork: branch from session_id into a new session
forked_id = None
async for message in query(
    prompt="Instead of JWT, implement OAuth2 for the auth module",
    options=ClaudeAgentOptions(
        resume=session_id,
        fork_session=True,
    ),
):
    if isinstance(message, ResultMessage):
        forked_id = message.session_id  # The fork's ID, distinct from session_id
        if message.subtype == "success":
            print(message.result)

print(f"Forked session: {forked_id}")

# Original session is untouched; resuming it continues the JWT thread
async for message in query(
    prompt="Continue with the JWT approach",
    options=ClaudeAgentOptions(resume=session_id),
):
    if isinstance(message, ResultMessage) and message.subtype == "success":
        print(message.result)
```

**TypeScript**
```typescript
// Fork: branch from sessionId into a new session
let forkedId: string | undefined;

for await (const message of query({
  prompt: "Instead of JWT, implement OAuth2 for the auth module",
  options: {
    resume: sessionId,
    forkSession: true
  }
})) {
  if (message.type === "system" && message.subtype === "init") {
    forkedId = message.session_id; // The fork's ID, distinct from sessionId
  }
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}

console.log(`Forked session: ${forkedId}`);

// Original session is untouched; resuming it continues the JWT thread
for await (const message of query({
  prompt: "Continue with the JWT approach",
  options: { resume: sessionId }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

## Reanudar entre hosts

Los archivos de sesión son locales a la máquina que los creó. Para reanudar una sesión en un host diferente (workers de CI, contenedores efímeros, serverless), tienes dos opciones:

- **Mueve el archivo de sesión.** Persiste `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl` desde la primera ejecución y restáuralo en la misma ruta en el nuevo host antes de llamar a `resume`. El `cwd` debe coincidir.
- **No dependas de la reanudación de sesiones.** Captura los resultados que necesitas (salida de análisis, decisiones, diffs de archivos) como estado de la aplicación y pásalos al prompt de una sesión nueva. Esto suele ser más robusto que transportar archivos de transcripción.

Ambos SDKs exponen funciones para enumerar sesiones en disco y leer sus mensajes: [`listSessions()`](/docs/en/agent-sdk/typescript#list-sessions) y [`getSessionMessages()`](/docs/en/agent-sdk/typescript#get-session-messages) en TypeScript, [`list_sessions()`](/docs/en/agent-sdk/python#list-sessions) y [`get_session_messages()`](/docs/en/agent-sdk/python#get-session-messages) en Python. Úsalas para construir selectores de sesiones personalizados, lógica de limpieza o visores de transcripciones.

## Recursos relacionados

- [Cómo funciona el bucle del agente](../How%20the%20agent%20loop%20works.md): Comprende turnos, mensajes y acumulación de contexto dentro de una sesión
- [File checkpointing](../Guides/Rewind%20file%20changes%20with%20checkpointing.md): Rastrea y revierte cambios de archivos entre sesiones
- Python `ClaudeAgentOptions`: Referencia completa de opciones de sesión para Python
- TypeScript `Options`: Referencia completa de opciones de sesión para TypeScript
