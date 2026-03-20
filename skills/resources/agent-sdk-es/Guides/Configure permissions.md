# Configurar permisos

Controla cómo tu agente utiliza las herramientas mediante modos de permisos, hooks y reglas declarativas de allow/deny.

---

El SDK de Agentes de Claude ofrece controles de permisos para gestionar cómo Claude usa las herramientas. Utiliza modos de permisos y reglas para definir qué se aprueba automáticamente, y el callback [`canUseTool`](/docs/en/agent-sdk/user-input) para manejar todo lo demás en tiempo de ejecución.

> **Nota:** Esta página cubre los modos de permisos y las reglas. Para crear flujos de aprobación interactivos donde los usuarios aprueben o denieguen solicitudes de herramientas en tiempo de ejecución, consulta [Gestionar aprobaciones y entrada del usuario](/docs/en/agent-sdk/user-input).

## Cómo se evalúan los permisos

Cuando Claude solicita una herramienta, el SDK verifica los permisos en este orden:

### 1. Hooks

Ejecuta los [hooks](/docs/en/agent-sdk/hooks) primero; estos pueden permitir, denegar o continuar al siguiente paso.

### 2. Reglas de denegación

Verifica las reglas `deny` (de `disallowed_tools` y [settings.json](https://code.claude.com/docs/en/settings#permission-settings)). Si una regla de denegación coincide, la herramienta se bloquea, incluso en el modo `bypassPermissions`.

### 3. Modo de permisos

Aplica el [modo de permisos](#modos-de-permisos) activo. `bypassPermissions` aprueba todo lo que llega a este paso. `acceptEdits` aprueba las operaciones de archivo. Los demás modos continúan hacia el siguiente paso.

### 4. Reglas de permiso

Verifica las reglas `allow` (de `allowed_tools` y settings.json). Si una regla coincide, la herramienta se aprueba.

### 5. Callback canUseTool

Si ninguno de los pasos anteriores resuelve la solicitud, se llama al callback [`canUseTool`](/docs/en/agent-sdk/user-input) para tomar una decisión. En el modo `dontAsk`, este paso se omite y la herramienta se deniega.

![Diagrama de flujo de evaluación de permisos](/docs/images/agent-sdk/permissions-flow.svg)

Esta página se centra en las **reglas de allow y deny** y en los **modos de permisos**. Para los demás pasos:

- **Hooks:** ejecuta código personalizado para permitir, denegar o modificar solicitudes de herramientas. Consulta [Controlar la ejecución con hooks](/docs/en/agent-sdk/hooks).
- **Callback canUseTool:** solicita aprobación a los usuarios en tiempo de ejecución. Consulta [Gestionar aprobaciones y entrada del usuario](/docs/en/agent-sdk/user-input).

## Reglas de allow y deny

`allowed_tools` y `disallowed_tools` (TypeScript: `allowedTools` / `disallowedTools`) añaden entradas a las listas de reglas de allow y deny en el flujo de evaluación descrito arriba. Controlan si se aprueba una llamada a herramienta, no si la herramienta está disponible para Claude.

| Opción | Efecto |
| :--- | :--- |
| `allowed_tools=["Read", "Grep"]` | `Read` y `Grep` se aprueban automáticamente. Las herramientas no listadas siguen existiendo y pasan al modo de permisos y a `canUseTool`. |
| `disallowed_tools=["Bash"]` | `Bash` siempre se deniega. Las reglas de denegación se verifican primero y se mantienen en todos los modos de permisos, incluido `bypassPermissions`. |

Para un agente con permisos restringidos, combina `allowedTools` con `permissionMode: "dontAsk"` (solo TypeScript). Las herramientas listadas se aprueban; cualquier otra se deniega directamente en lugar de solicitar confirmación:

```typescript
const options = {
  allowedTools: ["Read", "Glob", "Grep"],
  permissionMode: "dontAsk"
};
```

En Python, `dontAsk` aún no está disponible como modo de permisos. Sin él, Claude puede intentar llamar a herramientas que no están en `allowed_tools`. La llamada se rechaza en tiempo de ejecución, pero Claude desperdicia un turno descubriéndolo. Para un control más estricto en Python, usa `disallowed_tools` para bloquear explícitamente las herramientas que no quieres que Claude intente usar.

> **Advertencia:** **`allowed_tools` no restringe `bypassPermissions`.** `allowed_tools` solo aprueba previamente las herramientas que lista. Las herramientas no listadas no coinciden con ninguna regla de allow y pasan al modo de permisos, donde `bypassPermissions` las aprueba. Establecer `allowed_tools=["Read"]` junto con `permission_mode="bypassPermissions"` sigue aprobando todas las herramientas, incluyendo `Bash`, `Write` y `Edit`. Si necesitas `bypassPermissions` pero quieres bloquear herramientas específicas, usa `disallowed_tools`.

También puedes configurar reglas de allow, deny y ask de forma declarativa en `.claude/settings.json`. El SDK no carga la configuración del sistema de archivos por defecto, por lo que debes establecer `setting_sources=["project"]` (TypeScript: `settingSources: ["project"]`) en tus opciones para que estas reglas se apliquen. Consulta [Configuración de permisos](https://code.claude.com/docs/en/settings#permission-settings) para conocer la sintaxis de las reglas.

## Modos de permisos

Los modos de permisos ofrecen un control global sobre cómo Claude usa las herramientas. Puedes establecer el modo de permisos al llamar a `query()` o cambiarlo dinámicamente durante sesiones de streaming.

### Modos disponibles

El SDK admite los siguientes modos de permisos:

| Modo | Descripción | Comportamiento de herramientas |
| :--- | :---------- | :------------ |
| `default` | Comportamiento de permisos estándar | Sin aprobaciones automáticas; las herramientas sin coincidencia activan tu callback `canUseTool` |
| `dontAsk` (solo TypeScript) | Denegar en lugar de solicitar | Todo lo que no esté preaprobado por `allowed_tools` o reglas se deniega; `canUseTool` nunca se llama |
| `acceptEdits` | Aceptar ediciones de archivos automáticamente | Las ediciones de archivos y las [operaciones del sistema de archivos](#modo-accept-edits-acceptedits) (`mkdir`, `rm`, `mv`, etc.) se aprueban automáticamente |
| `bypassPermissions` | Omitir todas las verificaciones de permisos | Todas las herramientas se ejecutan sin solicitudes de permiso (usar con precaución) |
| `plan` | Modo de planificación | Sin ejecución de herramientas; Claude planifica sin realizar cambios |

> **Advertencia:** **Herencia en subagentes:** Al usar `bypassPermissions`, todos los subagentes heredan este modo y no puede ser sobreescrito. Los subagentes pueden tener prompts de sistema diferentes y un comportamiento menos restringido que tu agente principal. Habilitar `bypassPermissions` les otorga acceso completo y autónomo al sistema sin ninguna solicitud de aprobación.

### Establecer el modo de permisos

Puedes establecer el modo de permisos una vez al iniciar una consulta, o cambiarlo dinámicamente mientras la sesión está activa.

#### En el momento de la consulta

Pasa `permission_mode` (Python) o `permissionMode` (TypeScript) al crear una consulta. Este modo se aplica durante toda la sesión, a menos que se cambie dinámicamente.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def main():
    async for message in query(
        prompt="Help me refactor this code",
        options=ClaudeAgentOptions(
            permission_mode="default",  # Set the mode here
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function main() {
  for await (const message of query({
    prompt: "Help me refactor this code",
    options: {
      permissionMode: "default" // Set the mode here
    }
  })) {
    if ("result" in message) {
      console.log(message.result);
    }
  }
}

main();
```

#### Durante el streaming

Llama a `set_permission_mode()` (Python) o `setPermissionMode()` (TypeScript) para cambiar el modo durante la sesión. El nuevo modo surte efecto inmediatamente para todas las solicitudes de herramientas posteriores. Esto te permite comenzar con restricciones y relajar los permisos a medida que se genera confianza; por ejemplo, cambiando a `acceptEdits` después de revisar el enfoque inicial de Claude.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def main():
    q = query(
        prompt="Help me refactor this code",
        options=ClaudeAgentOptions(
            permission_mode="default",  # Start in default mode
        ),
    )

    # Change mode dynamically mid-session
    await q.set_permission_mode("acceptEdits")

    # Process messages with the new permission mode
    async for message in q:
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function main() {
  const q = query({
    prompt: "Help me refactor this code",
    options: {
      permissionMode: "default" // Start in default mode
    }
  });

  // Change mode dynamically mid-session
  await q.setPermissionMode("acceptEdits");

  // Process messages with the new permission mode
  for await (const message of q) {
    if ("result" in message) {
      console.log(message.result);
    }
  }
}

main();
```

### Detalles de los modos

#### Modo accept edits (`acceptEdits`)

Aprueba automáticamente las operaciones de archivo para que Claude pueda editar código sin solicitar confirmación. Las demás herramientas (como los comandos Bash que no son operaciones del sistema de archivos) siguen requiriendo los permisos normales.

**Operaciones aprobadas automáticamente:**
- Ediciones de archivos (herramientas Edit, Write)
- Comandos del sistema de archivos: `mkdir`, `touch`, `rm`, `mv`, `cp`

**Cuándo usarlo:** cuando confías en las ediciones de Claude y quieres iterar más rápido, por ejemplo durante la creación de prototipos o cuando trabajas en un directorio aislado.

#### Modo don't ask (`dontAsk`, solo TypeScript)

Convierte cualquier solicitud de permiso en una denegación. Las herramientas preaprobadas por `allowed_tools`, reglas de allow en `settings.json`, o un hook se ejecutan con normalidad. Todo lo demás se deniega sin llamar a `canUseTool`.

**Cuándo usarlo:** cuando quieres una superficie de herramientas fija y explícita para un agente sin interfaz y prefieres una denegación directa en lugar de depender en silencio de la ausencia de `canUseTool`.

> **Nota:** `dontAsk` solo está disponible en el SDK de TypeScript. En Python no existe un equivalente exacto. Usa `disallowed_tools` para bloquear explícitamente las herramientas que no quieres que Claude use.

#### Modo bypass permissions (`bypassPermissions`)

Aprueba automáticamente todos los usos de herramientas sin solicitudes. Los hooks siguen ejecutándose y pueden bloquear operaciones si es necesario.

> **Advertencia:** Usar con extrema precaución. Claude tiene acceso completo al sistema en este modo. Úsalo solo en entornos controlados donde confíes en todas las operaciones posibles.
>
> `allowed_tools` no restringe este modo. Todas las herramientas se aprueban, no solo las que listaste. Las reglas de denegación (`disallowed_tools`), las reglas explícitas de `ask` y los hooks se evalúan antes de la verificación del modo y aún pueden bloquear una herramienta.

#### Modo plan (`plan`)

Impide la ejecución de herramientas por completo. Claude puede analizar código y crear planes, pero no puede realizar cambios. Claude puede usar `AskUserQuestion` para aclarar requisitos antes de finalizar el plan. Consulta [Gestionar aprobaciones y entrada del usuario](/docs/en/agent-sdk/user-input#handle-clarifying-questions) para saber cómo manejar estos prompts.

**Cuándo usarlo:** cuando quieres que Claude proponga cambios sin ejecutarlos, por ejemplo durante la revisión de código o cuando necesitas aprobar los cambios antes de que se realicen.

## Recursos relacionados

Para los demás pasos del flujo de evaluación de permisos:

- [Gestionar aprobaciones y entrada del usuario](/docs/en/agent-sdk/user-input): prompts de aprobación interactivos y preguntas de aclaración
- [Guía de hooks](/docs/en/agent-sdk/hooks): ejecuta código personalizado en puntos clave del ciclo de vida del agente
- [Reglas de permisos](https://code.claude.com/docs/en/settings#permission-settings): reglas declarativas de allow/deny en `settings.json`
