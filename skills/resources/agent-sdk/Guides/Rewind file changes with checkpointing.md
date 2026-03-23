# Rewind file changes with checkpointing

Rastrea los cambios de archivos durante las sesiones del agente y restaura archivos a cualquier estado anterior

---

El sistema de checkpointing de archivos rastrea las modificaciones realizadas a través de las herramientas Write, Edit y NotebookEdit durante una sesión del agente, permitiéndote volver los archivos a cualquier estado anterior. ¿Quieres probarlo? Salta al [ejemplo interactivo](#try-it-out).

Con el checkpointing puedes:

- **Deshacer cambios no deseados** restaurando archivos a un estado conocido y funcional
- **Explorar alternativas** restaurando a un checkpoint e intentando un enfoque diferente
- **Recuperarte de errores** cuando el agente realiza modificaciones incorrectas

> **Advertencia:** Solo se rastrean los cambios realizados a través de las herramientas Write, Edit y NotebookEdit. Los cambios realizados mediante comandos Bash (como `echo > file.txt` o `sed -i`) no son capturados por el sistema de checkpointing.

## Cómo funciona el checkpointing

Cuando habilitas el checkpointing de archivos, el SDK crea copias de respaldo de los archivos antes de modificarlos mediante las herramientas Write, Edit o NotebookEdit. Los mensajes de usuario en el flujo de respuesta incluyen un UUID de checkpoint que puedes usar como punto de restauración.

El checkpointing funciona con estas herramientas integradas que el agente usa para modificar archivos:

| Herramienta | Descripción |
|-------------|-------------|
| Write | Crea un nuevo archivo o sobrescribe uno existente con nuevo contenido |
| Edit | Realiza ediciones específicas en partes concretas de un archivo existente |
| NotebookEdit | Modifica celdas en notebooks de Jupyter (archivos `.ipynb`) |

> **Nota:** Revertir archivos restaura los archivos en disco a un estado anterior. No revierte la conversación en sí. El historial y el contexto de la conversación permanecen intactos después de llamar a `rewindFiles()` (TypeScript) o `rewind_files()` (Python).

El sistema de checkpointing rastrea:

- Archivos creados durante la sesión
- Archivos modificados durante la sesión
- El contenido original de los archivos modificados

Cuando vuelves a un checkpoint, los archivos creados se eliminan y los archivos modificados se restauran a su contenido en ese momento.

## Implementar el checkpointing

Para usar el checkpointing de archivos, actívalo en tus opciones, captura los UUIDs de checkpoint del flujo de respuesta y luego llama a `rewindFiles()` (TypeScript) o `rewind_files()` (Python) cuando necesites restaurar.

El siguiente ejemplo muestra el flujo completo: habilitar el checkpointing, capturar el UUID del checkpoint y el ID de sesión del flujo de respuesta, y luego reanudar la sesión más tarde para revertir los archivos. Cada paso se explica en detalle a continuación.

**Python**
```python
import asyncio
from claude_agent_sdk import (
    ClaudeSDKClient,
    ClaudeAgentOptions,
    UserMessage,
    ResultMessage,
)


async def main():
    # Step 1: Enable checkpointing
    options = ClaudeAgentOptions(
        enable_file_checkpointing=True,
        permission_mode="acceptEdits",  # Auto-accept file edits without prompting
        extra_args={
            "replay-user-messages": None
        },  # Required to receive checkpoint UUIDs in the response stream
    )

    checkpoint_id = None
    session_id = None

    # Run the query and capture checkpoint UUID and session ID
    async with ClaudeSDKClient(options) as client:
        await client.query("Refactor the authentication module")

        # Step 2: Capture checkpoint UUID from the first user message
        async for message in client.receive_response():
            if isinstance(message, UserMessage) and message.uuid and not checkpoint_id:
                checkpoint_id = message.uuid
            if isinstance(message, ResultMessage) and not session_id:
                session_id = message.session_id

    # Step 3: Later, rewind by resuming the session with an empty prompt
    if checkpoint_id and session_id:
        async with ClaudeSDKClient(
            ClaudeAgentOptions(enable_file_checkpointing=True, resume=session_id)
        ) as client:
            await client.query("")  # Empty prompt to open the connection
            async for message in client.receive_response():
                await client.rewind_files(checkpoint_id)
                break
        print(f"Rewound to checkpoint: {checkpoint_id}")


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function main() {
  // Step 1: Enable checkpointing
  const opts = {
    enableFileCheckpointing: true,
    permissionMode: "acceptEdits" as const, // Auto-accept file edits without prompting
    extraArgs: { "replay-user-messages": null } // Required to receive checkpoint UUIDs in the response stream
  };

  const response = query({
    prompt: "Refactor the authentication module",
    options: opts
  });

  let checkpointId: string | undefined;
  let sessionId: string | undefined;

  // Step 2: Capture checkpoint UUID from the first user message
  for await (const message of response) {
    if (message.type === "user" && message.uuid && !checkpointId) {
      checkpointId = message.uuid;
    }
    if ("session_id" in message && !sessionId) {
      sessionId = message.session_id;
    }
  }

  // Step 3: Later, rewind by resuming the session with an empty prompt
  if (checkpointId && sessionId) {
    const rewindQuery = query({
      prompt: "", // Empty prompt to open the connection
      options: { ...opts, resume: sessionId }
    });

    for await (const msg of rewindQuery) {
      await rewindQuery.rewindFiles(checkpointId);
      break;
    }
    console.log(`Rewound to checkpoint: ${checkpointId}`);
  }
}

main();
```

### 1. Habilitar el checkpointing

Configura las opciones del SDK para habilitar el checkpointing y recibir los UUIDs de checkpoint:

| Opción | Python | TypeScript | Descripción |
|--------|--------|------------|-------------|
| Habilitar checkpointing | `enable_file_checkpointing=True` | `enableFileCheckpointing: true` | Rastrea los cambios de archivos para poder revertirlos |
| Recibir UUIDs de checkpoint | `extra_args={"replay-user-messages": None}` | `extraArgs: { 'replay-user-messages': null }` | Necesario para obtener los UUIDs de mensajes de usuario en el flujo |

**Python**
```python
options = ClaudeAgentOptions(
    enable_file_checkpointing=True,
    permission_mode="acceptEdits",
    extra_args={"replay-user-messages": None},
)

async with ClaudeSDKClient(options) as client:
    await client.query("Refactor the authentication module")
```

**TypeScript**
```typescript
const response = query({
  prompt: "Refactor the authentication module",
  options: {
    enableFileCheckpointing: true,
    permissionMode: "acceptEdits" as const,
    extraArgs: { "replay-user-messages": null }
  }
});
```

### 2. Capturar el UUID del checkpoint y el ID de sesión

Con la opción `replay-user-messages` activada (mostrada arriba), cada mensaje de usuario en el flujo de respuesta tiene un UUID que sirve como checkpoint.

Para la mayoría de los casos de uso, captura el UUID del primer mensaje de usuario (`message.uuid`); revertir a él restaura todos los archivos a su estado original. Para almacenar múltiples checkpoints y revertir a estados intermedios, consulta [Múltiples puntos de restauración](#multiple-restore-points).

Capturar el ID de sesión (`message.session_id`) es opcional; solo lo necesitas si quieres revertir más tarde, después de que el flujo se complete. Si llamas a `rewindFiles()` inmediatamente mientras aún procesas mensajes (como hace el ejemplo en [Checkpoint antes de operaciones arriesgadas](#checkpoint-before-risky-operations)), puedes omitir la captura del ID de sesión.

**Python**
```python
checkpoint_id = None
session_id = None

async for message in client.receive_response():
    # Update checkpoint on each user message (keeps the latest)
    if isinstance(message, UserMessage) and message.uuid:
        checkpoint_id = message.uuid
    # Capture session ID from the result message
    if isinstance(message, ResultMessage):
        session_id = message.session_id
```

**TypeScript**
```typescript
let checkpointId: string | undefined;
let sessionId: string | undefined;

for await (const message of response) {
  // Update checkpoint on each user message (keeps the latest)
  if (message.type === "user" && message.uuid) {
    checkpointId = message.uuid;
  }
  // Capture session ID from any message that has it
  if ("session_id" in message) {
    sessionId = message.session_id;
  }
}
```

### 3. Revertir archivos

Para revertir después de que el flujo se complete, reanuda la sesión con un prompt vacío y llama a `rewind_files()` (Python) o `rewindFiles()` (TypeScript) con tu UUID de checkpoint. También puedes revertir durante el flujo; consulta [Checkpoint antes de operaciones arriesgadas](#checkpoint-before-risky-operations) para ese patrón.

**Python**
```python
async with ClaudeSDKClient(
    ClaudeAgentOptions(enable_file_checkpointing=True, resume=session_id)
) as client:
    await client.query("")  # Empty prompt to open the connection
    async for message in client.receive_response():
        await client.rewind_files(checkpoint_id)
        break
```

**TypeScript**
```typescript
const rewindQuery = query({
  prompt: "", // Empty prompt to open the connection
  options: { ...opts, resume: sessionId }
});

for await (const msg of rewindQuery) {
  await rewindQuery.rewindFiles(checkpointId);
  break;
}
```

Si capturas el ID de sesión y el ID de checkpoint, también puedes revertir desde la CLI:

```bash
claude --resume <session-id> --rewind-files <checkpoint-uuid>
```

## Patrones comunes

Estos patrones muestran distintas formas de capturar y usar los UUIDs de checkpoint según tu caso de uso.

### Checkpoint antes de operaciones arriesgadas

Este patrón conserva únicamente el UUID del checkpoint más reciente, actualizándolo antes de cada turno del agente. Si algo sale mal durante el procesamiento, puedes revertir inmediatamente al último estado seguro y salir del bucle.

**Python**
```python
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, UserMessage


async def main():
    options = ClaudeAgentOptions(
        enable_file_checkpointing=True,
        permission_mode="acceptEdits",
        extra_args={"replay-user-messages": None},
    )

    safe_checkpoint = None

    async with ClaudeSDKClient(options) as client:
        await client.query("Refactor the authentication module")

        async for message in client.receive_response():
            # Update checkpoint before each agent turn starts
            # This overwrites the previous checkpoint. Only keep the latest
            if isinstance(message, UserMessage) and message.uuid:
                safe_checkpoint = message.uuid

            # Decide when to revert based on your own logic
            # For example: error detection, validation failure, or user input
            if your_revert_condition and safe_checkpoint:
                await client.rewind_files(safe_checkpoint)
                # Exit the loop after rewinding, files are restored
                break


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function main() {
  const response = query({
    prompt: "Refactor the authentication module",
    options: {
      enableFileCheckpointing: true,
      permissionMode: "acceptEdits" as const,
      extraArgs: { "replay-user-messages": null }
    }
  });

  let safeCheckpoint: string | undefined;

  for await (const message of response) {
    // Update checkpoint before each agent turn starts
    // This overwrites the previous checkpoint. Only keep the latest
    if (message.type === "user" && message.uuid) {
      safeCheckpoint = message.uuid;
    }

    // Decide when to revert based on your own logic
    // For example: error detection, validation failure, or user input
    if (yourRevertCondition && safeCheckpoint) {
      await response.rewindFiles(safeCheckpoint);
      // Exit the loop after rewinding, files are restored
      break;
    }
  }
}

main();
```

### Múltiples puntos de restauración

Si Claude realiza cambios a lo largo de varios turnos, puede que quieras revertir a un punto específico en lugar de ir todo el camino de vuelta. Por ejemplo, si Claude refactoriza un archivo en el turno uno y agrega pruebas en el turno dos, puede que quieras conservar la refactorización pero deshacer las pruebas.

Este patrón almacena todos los UUIDs de checkpoint en un array con metadatos. Una vez completada la sesión, puedes revertir a cualquier checkpoint anterior:

**Python**
```python
import asyncio
from dataclasses import dataclass
from datetime import datetime
from claude_agent_sdk import (
    ClaudeSDKClient,
    ClaudeAgentOptions,
    UserMessage,
    ResultMessage,
)


# Store checkpoint metadata for better tracking
@dataclass
class Checkpoint:
    id: str
    description: str
    timestamp: datetime


async def main():
    options = ClaudeAgentOptions(
        enable_file_checkpointing=True,
        permission_mode="acceptEdits",
        extra_args={"replay-user-messages": None},
    )

    checkpoints = []
    session_id = None

    async with ClaudeSDKClient(options) as client:
        await client.query("Refactor the authentication module")

        async for message in client.receive_response():
            if isinstance(message, UserMessage) and message.uuid:
                checkpoints.append(
                    Checkpoint(
                        id=message.uuid,
                        description=f"After turn {len(checkpoints) + 1}",
                        timestamp=datetime.now(),
                    )
                )
            if isinstance(message, ResultMessage) and not session_id:
                session_id = message.session_id

    # Later: rewind to any checkpoint by resuming the session
    if checkpoints and session_id:
        target = checkpoints[0]  # Pick any checkpoint
        async with ClaudeSDKClient(
            ClaudeAgentOptions(enable_file_checkpointing=True, resume=session_id)
        ) as client:
            await client.query("")  # Empty prompt to open the connection
            async for message in client.receive_response():
                await client.rewind_files(target.id)
                break
        print(f"Rewound to: {target.description}")


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Store checkpoint metadata for better tracking
interface Checkpoint {
  id: string;
  description: string;
  timestamp: Date;
}

async function main() {
  const opts = {
    enableFileCheckpointing: true,
    permissionMode: "acceptEdits" as const,
    extraArgs: { "replay-user-messages": null }
  };

  const response = query({
    prompt: "Refactor the authentication module",
    options: opts
  });

  const checkpoints: Checkpoint[] = [];
  let sessionId: string | undefined;

  for await (const message of response) {
    if (message.type === "user" && message.uuid) {
      checkpoints.push({
        id: message.uuid,
        description: `After turn ${checkpoints.length + 1}`,
        timestamp: new Date()
      });
    }
    if ("session_id" in message && !sessionId) {
      sessionId = message.session_id;
    }
  }

  // Later: rewind to any checkpoint by resuming the session
  if (checkpoints.length > 0 && sessionId) {
    const target = checkpoints[0]; // Pick any checkpoint
    const rewindQuery = query({
      prompt: "", // Empty prompt to open the connection
      options: { ...opts, resume: sessionId }
    });

    for await (const msg of rewindQuery) {
      await rewindQuery.rewindFiles(target.id);
      break;
    }
    console.log(`Rewound to: ${target.description}`);
  }
}

main();
```

## Try it out

Este ejemplo completo crea un pequeño archivo de utilidades, hace que el agente agregue comentarios de documentación, te muestra los cambios y luego te pregunta si quieres revertir.

Antes de comenzar, asegúrate de tener el [Claude Agent SDK instalado](../Quickstart.md).

### 1. Crear un archivo de prueba

Crea un nuevo archivo llamado `utils.py` (Python) o `utils.ts` (TypeScript) y pega el siguiente código:

**utils.py**
```python
def add(a, b):
    return a + b


def subtract(a, b):
    return a - b


def multiply(a, b):
    return a * b


def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```

**utils.ts**
```typescript
export function add(a: number, b: number): number {
  return a + b;
}

export function subtract(a: number, b: number): number {
  return a - b;
}

export function multiply(a: number, b: number): number {
  return a * b;
}

export function divide(a: number, b: number): number {
  if (b === 0) {
    throw new Error("Cannot divide by zero");
  }
  return a / b;
}
```

### 2. Ejecutar el ejemplo interactivo

Crea un nuevo archivo llamado `try_checkpointing.py` (Python) o `try_checkpointing.ts` (TypeScript) en el mismo directorio que tu archivo de utilidades y pega el siguiente código.

Este script le pide a Claude que agregue comentarios de documentación a tu archivo de utilidades y luego te da la opción de revertir y restaurar el original.

**try_checkpointing.py**
```python
import asyncio
from claude_agent_sdk import (
    ClaudeSDKClient,
    ClaudeAgentOptions,
    UserMessage,
    ResultMessage,
)


async def main():
    # Configure the SDK with checkpointing enabled
    # - enable_file_checkpointing: Track file changes for rewinding
    # - permission_mode: Auto-accept file edits without prompting
    # - extra_args: Required to receive user message UUIDs in the stream
    options = ClaudeAgentOptions(
        enable_file_checkpointing=True,
        permission_mode="acceptEdits",
        extra_args={"replay-user-messages": None},
    )

    checkpoint_id = None  # Store the user message UUID for rewinding
    session_id = None  # Store the session ID for resuming

    print("Running agent to add doc comments to utils.py...\n")

    # Run the agent and capture checkpoint data from the response stream
    async with ClaudeSDKClient(options) as client:
        await client.query("Add doc comments to utils.py")

        async for message in client.receive_response():
            # Capture the first user message UUID - this is our restore point
            if isinstance(message, UserMessage) and message.uuid and not checkpoint_id:
                checkpoint_id = message.uuid
            # Capture the session ID so we can resume later
            if isinstance(message, ResultMessage):
                session_id = message.session_id

    print("Done! Open utils.py to see the added doc comments.\n")

    # Ask the user if they want to rewind the changes
    if checkpoint_id and session_id:
        response = input("Rewind to remove the doc comments? (y/n): ")

        if response.lower() == "y":
            # Resume the session with an empty prompt, then rewind
            async with ClaudeSDKClient(
                ClaudeAgentOptions(enable_file_checkpointing=True, resume=session_id)
            ) as client:
                await client.query("")  # Empty prompt opens the connection
                async for message in client.receive_response():
                    await client.rewind_files(checkpoint_id)  # Restore files
                    break

            print(
                "\n✓ File restored! Open utils.py to verify the doc comments are gone."
            )
        else:
            print("\nKept the modified file.")


asyncio.run(main())
```

**try_checkpointing.ts**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";
import * as readline from "readline";

async function main() {
  // Configure the SDK with checkpointing enabled
  // - enableFileCheckpointing: Track file changes for rewinding
  // - permissionMode: Auto-accept file edits without prompting
  // - extraArgs: Required to receive user message UUIDs in the stream
  const opts = {
    enableFileCheckpointing: true,
    permissionMode: "acceptEdits" as const,
    extraArgs: { "replay-user-messages": null }
  };

  let sessionId: string | undefined; // Store the session ID for resuming
  let checkpointId: string | undefined; // Store the user message UUID for rewinding

  console.log("Running agent to add doc comments to utils.ts...\n");

  // Run the agent and capture checkpoint data from the response stream
  const response = query({
    prompt: "Add doc comments to utils.ts",
    options: opts
  });

  for await (const message of response) {
    // Capture the first user message UUID - this is our restore point
    if (message.type === "user" && message.uuid && !checkpointId) {
      checkpointId = message.uuid;
    }
    // Capture the session ID so we can resume later
    if ("session_id" in message) {
      sessionId = message.session_id;
    }
  }

  console.log("Done! Open utils.ts to see the added doc comments.\n");

  // Ask the user if they want to rewind the changes
  if (checkpointId && sessionId) {
    const rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout
    });

    const answer = await new Promise<string>((resolve) => {
      rl.question("Rewind to remove the doc comments? (y/n): ", resolve);
    });
    rl.close();

    if (answer.toLowerCase() === "y") {
      // Resume the session with an empty prompt, then rewind
      const rewindQuery = query({
        prompt: "", // Empty prompt opens the connection
        options: { ...opts, resume: sessionId }
      });

      for await (const msg of rewindQuery) {
        await rewindQuery.rewindFiles(checkpointId); // Restore files
        break;
      }

      console.log("\n✓ File restored! Open utils.ts to verify the doc comments are gone.");
    } else {
      console.log("\nKept the modified file.");
    }
  }
}

main();
```

Este ejemplo demuestra el flujo completo de checkpointing:

1. **Habilitar el checkpointing**: configura el SDK con `enable_file_checkpointing=True` y `permission_mode="acceptEdits"` para aprobar automáticamente las ediciones de archivos
2. **Capturar los datos del checkpoint**: mientras el agente se ejecuta, almacena el UUID del primer mensaje de usuario (tu punto de restauración) y el ID de sesión
3. **Preguntar por la reversión**: después de que el agente termine, revisa tu archivo de utilidades para ver los comentarios de documentación y decide si quieres deshacer los cambios
4. **Reanudar y revertir**: si es así, reanuda la sesión con un prompt vacío y llama a `rewind_files()` para restaurar el archivo original

### 3. Ejecutar el ejemplo

Ejecuta el script desde el mismo directorio que tu archivo de utilidades.

> **Consejo:** Abre tu archivo de utilidades (`utils.py` o `utils.ts`) en tu IDE o editor antes de ejecutar el script. Verás cómo el archivo se actualiza en tiempo real mientras el agente agrega los comentarios de documentación y cómo vuelve al original cuando eliges revertir.

#### Python

```bash
python try_checkpointing.py
```

#### TypeScript

```bash
npx tsx try_checkpointing.ts
```

Verás cómo el agente agrega los comentarios de documentación y luego aparecerá un prompt preguntando si quieres revertir. Si eliges que sí, el archivo se restaura a su estado original.

## Limitaciones

El checkpointing de archivos tiene las siguientes limitaciones:

| Limitación | Descripción |
|------------|-------------|
| Solo herramientas Write/Edit/NotebookEdit | Los cambios realizados mediante comandos Bash no se rastrean |
| Misma sesión | Los checkpoints están vinculados a la sesión que los creó |
| Solo contenido de archivos | Crear, mover o eliminar directorios no se deshace al revertir |
| Archivos locales | Los archivos remotos o de red no se rastrean |

## Solución de problemas

### Las opciones de checkpointing no se reconocen

Si `enableFileCheckpointing` o `rewindFiles()` no está disponible, puede que estés usando una versión anterior del SDK.

**Solución**: Actualiza a la versión más reciente del SDK:
- **Python**: `pip install --upgrade claude-agent-sdk`
- **TypeScript**: `npm install @anthropic-ai/claude-agent-sdk@latest`

### Los mensajes de usuario no tienen UUIDs

Si `message.uuid` es `undefined` o está ausente, no estás recibiendo los UUIDs de checkpoint.

**Causa**: La opción `replay-user-messages` no está configurada.

**Solución**: Agrega `extra_args={"replay-user-messages": None}` (Python) o `extraArgs: { 'replay-user-messages': null }` (TypeScript) a tus opciones.

### Error "No file checkpoint found for message"

Este error ocurre cuando los datos del checkpoint no existen para el UUID de mensaje de usuario especificado.

**Causas comunes**:
- El checkpointing de archivos no estaba habilitado en la sesión original (`enable_file_checkpointing` o `enableFileCheckpointing` no estaba configurado en `true`)
- La sesión no se completó correctamente antes de intentar reanudarla y revertirla

**Solución**: Asegúrate de que `enable_file_checkpointing=True` (Python) o `enableFileCheckpointing: true` (TypeScript) estaba configurado en la sesión original, luego usa el patrón mostrado en los ejemplos: captura el UUID del primer mensaje de usuario, completa la sesión completamente, luego reanuda con un prompt vacío y llama a `rewindFiles()` una sola vez.

### Error "ProcessTransport is not ready for writing"

Este error ocurre cuando llamas a `rewindFiles()` o `rewind_files()` después de haber terminado de iterar sobre la respuesta. La conexión al proceso de la CLI se cierra cuando el bucle termina.

**Solución**: Reanuda la sesión con un prompt vacío y luego llama a rewind en la nueva consulta:

**Python**
```python
# Resume session with empty prompt, then rewind
async with ClaudeSDKClient(
    ClaudeAgentOptions(enable_file_checkpointing=True, resume=session_id)
) as client:
    await client.query("")
    async for message in client.receive_response():
        await client.rewind_files(checkpoint_id)
        break
```

**TypeScript**
```typescript
// Resume session with empty prompt, then rewind
const rewindQuery = query({
  prompt: "",
  options: { ...opts, resume: sessionId }
});

for await (const msg of rewindQuery) {
  await rewindQuery.rewindFiles(checkpointId);
  break;
}
```

## Próximos pasos

- **[Sessions](../Core%20concepts/Work%20with%20sessions.md)**: aprende a reanudar sesiones, lo cual es necesario para revertir después de que el flujo se complete. Cubre los IDs de sesión, la reanudación de conversaciones y la bifurcación de sesiones.
- **[Permissions](./Configure%20permissions.md)**: configura qué herramientas puede usar Claude y cómo se aprueban las modificaciones de archivos. Útil si quieres tener más control sobre cuándo ocurren las ediciones.
- **[TypeScript SDK reference](/docs/en/agent-sdk/typescript)**: referencia completa de la API incluyendo todas las opciones de `query()` y el método `rewindFiles()`.
- **[Python SDK reference](/docs/en/agent-sdk/python)**: referencia completa de la API incluyendo todas las opciones de `ClaudeAgentOptions` y el método `rewind_files()`.
