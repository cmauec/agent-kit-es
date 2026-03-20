# Manejar aprobaciones e input del usuario

Expone las solicitudes de aprobación y preguntas aclaratorias de Claude a los usuarios, y devuelve sus decisiones al SDK.

---

Mientras trabaja en una tarea, Claude a veces necesita consultar con los usuarios. Puede necesitar permiso antes de eliminar archivos, o preguntar qué base de datos usar para un nuevo proyecto. Tu aplicación necesita exponer estas solicitudes a los usuarios para que Claude pueda continuar con su input.

Claude solicita input del usuario en dos situaciones: cuando necesita **permiso para usar una herramienta** (como eliminar archivos o ejecutar comandos), y cuando tiene **preguntas aclaratorias** (a través de la herramienta `AskUserQuestion`). Ambas situaciones activan tu callback `canUseTool`, que pausa la ejecución hasta que devuelvas una respuesta. Esto es diferente de los turnos de conversación normales donde Claude termina y espera tu próximo mensaje.

Para las preguntas aclaratorias, Claude genera las preguntas y las opciones. Tu rol es presentárselas a los usuarios y devolver sus selecciones. No puedes agregar tus propias preguntas a este flujo; si necesitas preguntarle algo a los usuarios tú mismo, hazlo por separado en la lógica de tu aplicación.

Esta guía muestra cómo detectar cada tipo de solicitud y responder de manera apropiada.

## Detectar cuándo Claude necesita input

Pasa un callback `canUseTool` en las opciones de tu query. El callback se activa cada vez que Claude necesita input del usuario, recibiendo el nombre de la herramienta y el input como argumentos:

**Python**
```python
async def handle_tool_request(tool_name, input_data, context):
    # Prompt user and return allow or deny
    ...


options = ClaudeAgentOptions(can_use_tool=handle_tool_request)
```

**TypeScript**
```typescript
async function handleToolRequest(toolName, input) {
  // Prompt user and return allow or deny
}

const options = { canUseTool: handleToolRequest };
```

El callback se activa en dos casos:

1. **La herramienta necesita aprobación**: Claude quiere usar una herramienta que no está auto-aprobada por las [reglas de permisos](/docs/en/agent-sdk/permissions) o los modos. Verifica `tool_name` para identificar la herramienta (ej., `"Bash"`, `"Write"`).
2. **Claude hace una pregunta**: Claude llama a la herramienta `AskUserQuestion`. Verifica si `tool_name == "AskUserQuestion"` para manejarlo de forma diferente. Si especificas un array `tools`, incluye `AskUserQuestion` para que esto funcione. Consulta [Manejar preguntas aclaratorias](#handle-clarifying-questions) para más detalles.

> **Nota:** Para permitir o denegar herramientas automáticamente sin consultar a los usuarios, usa [hooks](/docs/en/agent-sdk/hooks) en su lugar. Los hooks se ejecutan antes que `canUseTool` y pueden permitir, denegar o modificar solicitudes según tu propia lógica. También puedes usar el hook [`PermissionRequest`](/docs/en/agent-sdk/hooks#available-hooks) para enviar notificaciones externas (Slack, email, push) cuando Claude esté esperando aprobación.

## Manejar solicitudes de aprobación de herramientas

Una vez que hayas pasado un callback `canUseTool` en las opciones de tu query, se activa cuando Claude quiere usar una herramienta que no está auto-aprobada. Tu callback recibe dos argumentos:

| Argumento | Descripción |
|----------|-------------|
| `toolName` | El nombre de la herramienta que Claude quiere usar (ej., `"Bash"`, `"Write"`, `"Edit"`) |
| `input` | Los parámetros que Claude está pasando a la herramienta. El contenido varía según la herramienta. |

El objeto `input` contiene parámetros específicos de cada herramienta. Ejemplos comunes:

| Herramienta | Campos de input |
|------|--------------|
| `Bash` | `command`, `description`, `timeout` |
| `Write` | `file_path`, `content` |
| `Edit` | `file_path`, `old_string`, `new_string` |
| `Read` | `file_path`, `offset`, `limit` |

Consulta la referencia del SDK para los esquemas de input completos: [Python](/docs/en/agent-sdk/python#tool-input-output-types) | [TypeScript](/docs/en/agent-sdk/typescript#tool-input-types).

Puedes mostrar esta información al usuario para que decida si permitir o rechazar la acción, y luego devolver la respuesta apropiada.

El siguiente ejemplo le pide a Claude que cree y elimine un archivo de prueba. Cuando Claude intenta cada operación, el callback imprime la solicitud de herramienta en la terminal y pide aprobación y/n.

**Python**
```python
import asyncio

from claude_agent_sdk import ClaudeAgentOptions, query
from claude_agent_sdk.types import (
    HookMatcher,
    PermissionResultAllow,
    PermissionResultDeny,
    ToolPermissionContext,
)


async def can_use_tool(
    tool_name: str, input_data: dict, context: ToolPermissionContext
) -> PermissionResultAllow | PermissionResultDeny:
    # Display the tool request
    print(f"\nTool: {tool_name}")
    if tool_name == "Bash":
        print(f"Command: {input_data.get('command')}")
        if input_data.get("description"):
            print(f"Description: {input_data.get('description')}")
    else:
        print(f"Input: {input_data}")

    # Get user approval
    response = input("Allow this action? (y/n): ")

    # Return allow or deny based on user's response
    if response.lower() == "y":
        # Allow: tool executes with the original (or modified) input
        return PermissionResultAllow(updated_input=input_data)
    else:
        # Deny: tool doesn't execute, Claude sees the message
        return PermissionResultDeny(message="User denied this action")


# Required workaround: dummy hook keeps the stream open for can_use_tool
async def dummy_hook(input_data, tool_use_id, context):
    return {"continue_": True}


async def prompt_stream():
    yield {
        "type": "user",
        "message": {
            "role": "user",
            "content": "Create a test file in /tmp and then delete it",
        },
    }


async def main():
    async for message in query(
        prompt=prompt_stream(),
        options=ClaudeAgentOptions(
            can_use_tool=can_use_tool,
            hooks={"PreToolUse": [HookMatcher(matcher=None, hooks=[dummy_hook])]},
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";
import * as readline from "readline";

// Helper to prompt user for input in the terminal
function prompt(question: string): Promise<string> {
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout
  });
  return new Promise((resolve) =>
    rl.question(question, (answer) => {
      rl.close();
      resolve(answer);
    })
  );
}

for await (const message of query({
  prompt: "Create a test file in /tmp and then delete it",
  options: {
    canUseTool: async (toolName, input) => {
      // Display the tool request
      console.log(`\nTool: ${toolName}`);
      if (toolName === "Bash") {
        console.log(`Command: ${input.command}`);
        if (input.description) console.log(`Description: ${input.description}`);
      } else {
        console.log(`Input: ${JSON.stringify(input, null, 2)}`);
      }

      // Get user approval
      const response = await prompt("Allow this action? (y/n): ");

      // Return allow or deny based on user's response
      if (response.toLowerCase() === "y") {
        // Allow: tool executes with the original (or modified) input
        return { behavior: "allow", updatedInput: input };
      } else {
        // Deny: tool doesn't execute, Claude sees the message
        return { behavior: "deny", message: "User denied this action" };
      }
    }
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

> **Nota:** En Python, `can_use_tool` requiere el [modo streaming](/docs/en/agent-sdk/streaming-vs-single-mode) y un hook `PreToolUse` que devuelva `{"continue_": True}` para mantener el stream abierto. Sin este hook, el stream se cierra antes de que el callback de permisos pueda ser invocado.

Este ejemplo usa un flujo y/n donde cualquier input distinto a `y` se trata como una denegación. En la práctica, podrías construir una interfaz más rica que permita a los usuarios modificar la solicitud, proporcionar feedback, o redirigir a Claude por completo. Consulta [Responder a solicitudes de herramientas](#respond-to-tool-requests) para conocer todas las formas en que puedes responder.

### Responder a solicitudes de herramientas

Tu callback devuelve uno de dos tipos de respuesta:

| Respuesta | Python | TypeScript |
|----------|--------|------------|
| **Permitir** | `PermissionResultAllow(updated_input=...)` | `{ behavior: "allow", updatedInput }` |
| **Denegar** | `PermissionResultDeny(message=...)` | `{ behavior: "deny", message }` |

Al permitir, pasa el input de la herramienta (original o modificado). Al denegar, proporciona un mensaje explicando el motivo. Claude ve este mensaje y puede ajustar su enfoque.

**Python**
```python
from claude_agent_sdk.types import PermissionResultAllow, PermissionResultDeny

# Allow the tool to execute
return PermissionResultAllow(updated_input=input_data)

# Block the tool
return PermissionResultDeny(message="User rejected this action")
```

**TypeScript**
```typescript
// Allow the tool to execute
return { behavior: "allow", updatedInput: input };

// Block the tool
return { behavior: "deny", message: "User rejected this action" };
```

Más allá de permitir o denegar, puedes modificar el input de la herramienta o proporcionar contexto que ayude a Claude a ajustar su enfoque:

- **Aprobar**: deja que la herramienta se ejecute tal como Claude lo solicitó
- **Aprobar con cambios**: modifica el input antes de la ejecución (ej., sanear paths, agregar restricciones)
- **Rechazar**: bloquea la herramienta y dile a Claude por qué
- **Sugerir una alternativa**: bloquea pero guía a Claude hacia lo que el usuario quiere en su lugar
- **Redirigir por completo**: usa el [input en streaming](/docs/en/agent-sdk/streaming-vs-single-mode) para enviarle a Claude una instrucción completamente nueva

#### Aprobar

El usuario aprueba la acción tal como está. Pasa el `input` de tu callback sin cambios y la herramienta se ejecuta exactamente como Claude lo solicitó.

**Python**
```python
async def can_use_tool(tool_name, input_data, context):
    print(f"Claude wants to use {tool_name}")
    approved = await ask_user("Allow this action?")

    if approved:
        return PermissionResultAllow(updated_input=input_data)
    return PermissionResultDeny(message="User declined")
```

**TypeScript**
```typescript
canUseTool: async (toolName, input) => {
  console.log(`Claude wants to use ${toolName}`);
  const approved = await askUser("Allow this action?");

  if (approved) {
    return { behavior: "allow", updatedInput: input };
  }
  return { behavior: "deny", message: "User declined" };
};
```

#### Aprobar con cambios

El usuario aprueba pero quiere modificar la solicitud primero. Puedes cambiar el input antes de que la herramienta se ejecute. Claude ve el resultado pero no se le informa de que cambiaste algo. Útil para sanear parámetros, agregar restricciones o limitar el acceso.

**Python**
```python
async def can_use_tool(tool_name, input_data, context):
    if tool_name == "Bash":
        # User approved, but scope all commands to sandbox
        sandboxed_input = {**input_data}
        sandboxed_input["command"] = input_data["command"].replace(
            "/tmp", "/tmp/sandbox"
        )
        return PermissionResultAllow(updated_input=sandboxed_input)
    return PermissionResultAllow(updated_input=input_data)
```

**TypeScript**
```typescript
canUseTool: async (toolName, input) => {
  if (toolName === "Bash") {
    // User approved, but scope all commands to sandbox
    const sandboxedInput = {
      ...input,
      command: input.command.replace("/tmp", "/tmp/sandbox")
    };
    return { behavior: "allow", updatedInput: sandboxedInput };
  }
  return { behavior: "allow", updatedInput: input };
};
```

#### Rechazar

El usuario no quiere que esta acción ocurra. Bloquea la herramienta y proporciona un mensaje explicando el motivo. Claude ve este mensaje y puede intentar un enfoque diferente.

**Python**
```python
async def can_use_tool(tool_name, input_data, context):
    approved = await ask_user(f"Allow {tool_name}?")

    if not approved:
        return PermissionResultDeny(message="User rejected this action")
    return PermissionResultAllow(updated_input=input_data)
```

**TypeScript**
```typescript
canUseTool: async (toolName, input) => {
  const approved = await askUser(`Allow ${toolName}?`);

  if (!approved) {
    return {
      behavior: "deny",
      message: "User rejected this action"
    };
  }
  return { behavior: "allow", updatedInput: input };
};
```

#### Sugerir una alternativa

El usuario no quiere esta acción específica, pero tiene una idea diferente. Bloquea la herramienta e incluye orientación en tu mensaje. Claude leerá esto y decidirá cómo proceder en base a tu feedback.

**Python**
```python
async def can_use_tool(tool_name, input_data, context):
    if tool_name == "Bash" and "rm" in input_data.get("command", ""):
        # User doesn't want to delete, suggest archiving instead
        return PermissionResultDeny(
            message="User doesn't want to delete files. They asked if you could compress them into an archive instead."
        )
    return PermissionResultAllow(updated_input=input_data)
```

**TypeScript**
```typescript
canUseTool: async (toolName, input) => {
  if (toolName === "Bash" && input.command.includes("rm")) {
    // User doesn't want to delete, suggest archiving instead
    return {
      behavior: "deny",
      message:
        "User doesn't want to delete files. They asked if you could compress them into an archive instead."
    };
  }
  return { behavior: "allow", updatedInput: input };
};
```

#### Redirigir por completo

Para un cambio de dirección completo (no solo un pequeño ajuste), usa el [input en streaming](/docs/en/agent-sdk/streaming-vs-single-mode) para enviarle a Claude una nueva instrucción directamente. Esto omite la solicitud de herramienta actual y le da a Claude instrucciones completamente nuevas a seguir.

## Manejar preguntas aclaratorias

Cuando Claude necesita más orientación en una tarea con múltiples enfoques válidos, llama a la herramienta `AskUserQuestion`. Esto activa tu callback `canUseTool` con `toolName` establecido en `AskUserQuestion`. El input contiene las preguntas de Claude como opciones de selección múltiple, que presentas al usuario y devuelves sus selecciones.

> **Tip:** Las preguntas aclaratorias son especialmente comunes en el [modo `plan`](/docs/en/agent-sdk/permissions#plan-mode-plan), donde Claude explora el código base y hace preguntas antes de proponer un plan. Esto hace que el modo plan sea ideal para flujos de trabajo interactivos donde quieres que Claude recopile requisitos antes de realizar cambios.

Los siguientes pasos muestran cómo manejar las preguntas aclaratorias:

### 1. Pasar un callback canUseTool

Pasa un callback `canUseTool` en las opciones de tu query. Por defecto, `AskUserQuestion` está disponible. Si especificas un array `tools` para restringir las capacidades de Claude (por ejemplo, un agente de solo lectura con únicamente `Read`, `Glob` y `Grep`), incluye `AskUserQuestion` en ese array. De lo contrario, Claude no podrá hacer preguntas aclaratorias:

**Python**
```python
async for message in query(
    prompt="Analyze this codebase",
    options=ClaudeAgentOptions(
        # Include AskUserQuestion in your tools list
        tools=["Read", "Glob", "Grep", "AskUserQuestion"],
        can_use_tool=can_use_tool,
    ),
):
    print(message)
```

**TypeScript**
```typescript
for await (const message of query({
  prompt: "Analyze this codebase",
  options: {
    // Include AskUserQuestion in your tools list
    tools: ["Read", "Glob", "Grep", "AskUserQuestion"],
    canUseTool: async (toolName, input) => {
      // Handle clarifying questions here
    }
  }
})) {
  console.log(message);
}
```

### 2. Detectar AskUserQuestion

En tu callback, verifica si `toolName` es igual a `AskUserQuestion` para manejarlo de forma diferente a otras herramientas:

**Python**
```python
async def can_use_tool(tool_name: str, input_data: dict, context):
    if tool_name == "AskUserQuestion":
        # Your implementation to collect answers from the user
        return await handle_clarifying_questions(input_data)
    # Handle other tools normally
    return await prompt_for_approval(tool_name, input_data)
```

**TypeScript**
```typescript
canUseTool: async (toolName, input) => {
  if (toolName === "AskUserQuestion") {
    // Your implementation to collect answers from the user
    return handleClarifyingQuestions(input);
  }
  // Handle other tools normally
  return promptForApproval(toolName, input);
};
```

### 3. Parsear el input de la pregunta

El input contiene las preguntas de Claude en un array `questions`. Cada pregunta tiene un campo `question` (el texto a mostrar), `options` (las opciones) y `multiSelect` (si se permiten múltiples selecciones):

```json
{
  "questions": [
    {
      "question": "How should I format the output?",
      "header": "Format",
      "options": [
        { "label": "Summary", "description": "Brief overview" },
        { "label": "Detailed", "description": "Full explanation" }
      ],
      "multiSelect": false
    },
    {
      "question": "Which sections should I include?",
      "header": "Sections",
      "options": [
        { "label": "Introduction", "description": "Opening context" },
        { "label": "Conclusion", "description": "Final summary" }
      ],
      "multiSelect": true
    }
  ]
}
```

Consulta [Formato de pregunta](#question-format) para descripciones completas de los campos.

### 4. Recopilar las respuestas del usuario

Presenta las preguntas al usuario y recoge sus selecciones. La forma de hacerlo depende de tu aplicación: un prompt en la terminal, un formulario web, un diálogo móvil, etc.

### 5. Devolver las respuestas a Claude

Construye el objeto `answers` como un registro donde cada clave es el texto del campo `question` y cada valor es el `label` de la opción seleccionada:

| Del objeto de pregunta | Usar como |
|--------------------------|--------|
| Campo `question` (ej., `"How should I format the output?"`) | Clave |
| Campo `label` de la opción seleccionada (ej., `"Summary"`) | Valor |

Para preguntas de selección múltiple, une múltiples labels con `", "`. Si [admites input de texto libre](#support-free-text-input), usa el texto personalizado del usuario como valor.

**Python**
```python
return PermissionResultAllow(
    updated_input={
        "questions": input_data.get("questions", []),
        "answers": {
            "How should I format the output?": "Summary",
            "Which sections should I include?": "Introduction, Conclusion",
        },
    }
)
```

**TypeScript**
```typescript
return {
  behavior: "allow",
  updatedInput: {
    questions: input.questions,
    answers: {
      "How should I format the output?": "Summary",
      "Which sections should I include?": "Introduction, Conclusion"
    }
  }
};
```

### Formato de pregunta

El input contiene las preguntas generadas por Claude en un array `questions`. Cada pregunta tiene estos campos:

| Campo | Descripción |
|-------|-------------|
| `question` | El texto completo de la pregunta a mostrar |
| `header` | Etiqueta corta para la pregunta (máximo 12 caracteres) |
| `options` | Array de 2-4 opciones, cada una con `label` y `description`. TypeScript: opcionalmente `preview` (ver [más abajo](#option-previews-type-script)) |
| `multiSelect` | Si es `true`, los usuarios pueden seleccionar múltiples opciones |

La estructura que recibe tu callback:

```json
{
  "questions": [
    {
      "question": "How should I format the output?",
      "header": "Format",
      "options": [
        { "label": "Summary", "description": "Brief overview of key points" },
        { "label": "Detailed", "description": "Full explanation with examples" }
      ],
      "multiSelect": false
    }
  ]
}
```

#### Previsualizaciones de opciones (TypeScript)

`toolConfig.askUserQuestion.previewFormat` agrega un campo `preview` a cada opción para que tu app pueda mostrar una maqueta visual junto al label. Sin esta configuración, Claude no genera previsualizaciones y el campo estará ausente.

| `previewFormat` | `preview` contiene |
|:----------------|:-----------------------|
| sin definir (por defecto) | El campo está ausente. Claude no genera previsualizaciones. |
| `"markdown"` | Arte ASCII y bloques de código delimitados |
| `"html"` | Un fragmento `<div>` con estilos (el SDK rechaza `<script>`, `<style>` y `<!DOCTYPE>` antes de que se ejecute tu callback) |

El formato aplica a todas las preguntas de la sesión. Claude incluye `preview` en las opciones donde una comparación visual ayuda (elecciones de diseño, esquemas de color) y lo omite donde no aportaría valor (confirmaciones sí/no, elecciones solo de texto). Verifica si es `undefined` antes de renderizar.

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Help me choose a card layout",
  options: {
    toolConfig: {
      askUserQuestion: { previewFormat: "html" }
    },
    canUseTool: async (toolName, input) => {
      // input.questions[].options[].preview is an HTML string or undefined
      return { behavior: "allow", updatedInput: input };
    }
  }
})) {
  // ...
}
```

Una opción con una previsualización HTML:

```json
{
  "label": "Compact",
  "description": "Title and metric value only",
  "preview": "<div style=\"padding:12px;border:1px solid #ddd;border-radius:8px\"><div style=\"font-size:12px;color:#666\">Active users</div><div style=\"font-size:28px;font-weight:600\">1,284</div></div>"
}
```

### Formato de respuesta

Devuelve un objeto `answers` que mapee el campo `question` de cada pregunta al `label` de la opción seleccionada:

| Campo | Descripción |
|-------|-------------|
| `questions` | Pasa el array de preguntas original (requerido para el procesamiento de la herramienta) |
| `answers` | Objeto donde las claves son el texto de la pregunta y los valores son los labels seleccionados |

Para preguntas de selección múltiple, une múltiples labels con `", "`. Para input de texto libre, usa el texto personalizado del usuario directamente.

```json
{
  "questions": [
    // ...
  ],
  "answers": {
    "How should I format the output?": "Summary",
    "Which sections should I include?": "Introduction, Conclusion"
  }
}
```

#### Admitir input de texto libre

Las opciones predefinidas de Claude no siempre cubrirán lo que los usuarios quieren. Para permitir que los usuarios escriban su propia respuesta:

- Muestra una opción adicional "Other" después de las opciones de Claude que acepte input de texto
- Usa el texto personalizado del usuario como valor de la respuesta (no la palabra "Other")

Consulta el [ejemplo completo](#complete-example) a continuación para una implementación completa.

### Ejemplo completo

Claude hace preguntas aclaratorias cuando necesita input del usuario para continuar. Por ejemplo, cuando se le pide ayuda para decidir el stack tecnológico de una app móvil, Claude podría preguntar sobre multiplataforma vs nativo, preferencias de backend, o plataformas objetivo. Estas preguntas ayudan a Claude a tomar decisiones que se alineen con las preferencias del usuario en lugar de adivinar.

Este ejemplo maneja esas preguntas en una aplicación de terminal. Esto es lo que ocurre en cada paso:

1. **Enrutar la solicitud**: el callback `canUseTool` verifica si el nombre de la herramienta es `"AskUserQuestion"` y enruta a un handler dedicado
2. **Mostrar las preguntas**: el handler recorre el array `questions` e imprime cada pregunta con opciones numeradas
3. **Recopilar el input**: el usuario puede ingresar un número para seleccionar una opción, o escribir texto libre directamente (ej., "jquery", "i don't know")
4. **Mapear las respuestas**: el código verifica si el input es numérico (usa el label de la opción) o texto libre (usa el texto directamente)
5. **Devolver a Claude**: la respuesta incluye tanto el array `questions` original como el mapeo de `answers`

**Python**
```python
import asyncio

from claude_agent_sdk import ClaudeAgentOptions, query
from claude_agent_sdk.types import HookMatcher, PermissionResultAllow


def parse_response(response: str, options: list) -> str:
    """Parse user input as option number(s) or free text."""
    try:
        indices = [int(s.strip()) - 1 for s in response.split(",")]
        labels = [options[i]["label"] for i in indices if 0 <= i < len(options)]
        return ", ".join(labels) if labels else response
    except ValueError:
        return response


async def handle_ask_user_question(input_data: dict) -> PermissionResultAllow:
    """Display Claude's questions and collect user answers."""
    answers = {}

    for q in input_data.get("questions", []):
        print(f"\n{q['header']}: {q['question']}")

        options = q["options"]
        for i, opt in enumerate(options):
            print(f"  {i + 1}. {opt['label']} - {opt['description']}")
        if q.get("multiSelect"):
            print("  (Enter numbers separated by commas, or type your own answer)")
        else:
            print("  (Enter a number, or type your own answer)")

        response = input("Your choice: ").strip()
        answers[q["question"]] = parse_response(response, options)

    return PermissionResultAllow(
        updated_input={
            "questions": input_data.get("questions", []),
            "answers": answers,
        }
    )


async def can_use_tool(
    tool_name: str, input_data: dict, context
) -> PermissionResultAllow:
    # Route AskUserQuestion to our question handler
    if tool_name == "AskUserQuestion":
        return await handle_ask_user_question(input_data)
    # Auto-approve other tools for this example
    return PermissionResultAllow(updated_input=input_data)


async def prompt_stream():
    yield {
        "type": "user",
        "message": {
            "role": "user",
            "content": "Help me decide on the tech stack for a new mobile app",
        },
    }


# Required workaround: dummy hook keeps the stream open for can_use_tool
async def dummy_hook(input_data, tool_use_id, context):
    return {"continue_": True}


async def main():
    async for message in query(
        prompt=prompt_stream(),
        options=ClaudeAgentOptions(
            can_use_tool=can_use_tool,
            hooks={"PreToolUse": [HookMatcher(matcher=None, hooks=[dummy_hook])]},
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";
import * as readline from "readline/promises";

// Helper to prompt user for input in the terminal
async function prompt(question: string): Promise<string> {
  const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
  const answer = await rl.question(question);
  rl.close();
  return answer;
}

// Parse user input as option number(s) or free text
function parseResponse(response: string, options: any[]): string {
  const indices = response.split(",").map((s) => parseInt(s.trim()) - 1);
  const labels = indices
    .filter((i) => !isNaN(i) && i >= 0 && i < options.length)
    .map((i) => options[i].label);
  return labels.length > 0 ? labels.join(", ") : response;
}

// Display Claude's questions and collect user answers
async function handleAskUserQuestion(input: any) {
  const answers: Record<string, string> = {};

  for (const q of input.questions) {
    console.log(`\n${q.header}: ${q.question}`);

    const options = q.options;
    options.forEach((opt: any, i: number) => {
      console.log(`  ${i + 1}. ${opt.label} - ${opt.description}`);
    });
    if (q.multiSelect) {
      console.log("  (Enter numbers separated by commas, or type your own answer)");
    } else {
      console.log("  (Enter a number, or type your own answer)");
    }

    const response = (await prompt("Your choice: ")).trim();
    answers[q.question] = parseResponse(response, options);
  }

  // Return the answers to Claude (must include original questions)
  return {
    behavior: "allow",
    updatedInput: { questions: input.questions, answers }
  };
}

async function main() {
  for await (const message of query({
    prompt: "Help me decide on the tech stack for a new mobile app",
    options: {
      canUseTool: async (toolName, input) => {
        // Route AskUserQuestion to our question handler
        if (toolName === "AskUserQuestion") {
          return handleAskUserQuestion(input);
        }
        // Auto-approve other tools for this example
        return { behavior: "allow", updatedInput: input };
      }
    }
  })) {
    if ("result" in message) console.log(message.result);
  }
}

main();
```

## Limitaciones

- **Subagentes**: `AskUserQuestion` no está disponible actualmente en subagentes generados mediante la herramienta Agent
- **Límites de preguntas**: cada llamada a `AskUserQuestion` admite entre 1 y 4 preguntas con entre 2 y 4 opciones cada una

## Otras formas de obtener input del usuario

El callback `canUseTool` y la herramienta `AskUserQuestion` cubren la mayoría de los escenarios de aprobación y aclaración, pero el SDK ofrece otras formas de obtener input de los usuarios:

### Input en streaming

Usa el [input en streaming](/docs/en/agent-sdk/streaming-vs-single-mode) cuando necesites:

- **Interrumpir al agente a mitad de la tarea**: enviar una señal de cancelación o cambiar de dirección mientras Claude está trabajando
- **Proporcionar contexto adicional**: agregar información que Claude necesita sin esperar a que la solicite
- **Construir interfaces de chat**: permitir que los usuarios envíen mensajes de seguimiento durante operaciones de larga duración

El input en streaming es ideal para interfaces conversacionales donde los usuarios interactúan con el agente a lo largo de toda la ejecución, no solo en los puntos de aprobación.

### Herramientas personalizadas

Usa [herramientas personalizadas](/docs/en/agent-sdk/custom-tools) cuando necesites:

- **Recopilar input estructurado**: construir formularios, asistentes o flujos de trabajo de varios pasos que vayan más allá del formato de selección múltiple de `AskUserQuestion`
- **Integrar sistemas de aprobación externos**: conectar con plataformas de tickets, flujos de trabajo o aprobación existentes
- **Implementar interacciones específicas del dominio**: crear herramientas adaptadas a las necesidades de tu aplicación, como interfaces de revisión de código o checklists de despliegue

Las herramientas personalizadas te dan control total sobre la interacción, pero requieren más trabajo de implementación que usar el callback integrado `canUseTool`.

## Recursos relacionados

- [Configurar permisos](/docs/en/agent-sdk/permissions): configurar modos y reglas de permisos
- [Controlar la ejecución con hooks](/docs/en/agent-sdk/hooks): ejecutar código personalizado en puntos clave del ciclo de vida del agente
- [Referencia del SDK de TypeScript](/docs/en/agent-sdk/typescript#can-use-tool): documentación completa de la API canUseTool
