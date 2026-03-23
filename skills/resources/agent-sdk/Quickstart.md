# Quickstart

Comienza con el Agent SDK para Python o TypeScript y construye agentes de IA que trabajen de forma autónoma

---

Usa el Agent SDK para construir un agente de IA que lea tu código, encuentre errores y los corrija, todo sin intervención manual.

**Lo que harás:**
1. Configurar un proyecto con el Agent SDK
2. Crear un archivo con código defectuoso
3. Ejecutar un agente que encuentre y corrija los errores automáticamente

## Requisitos previos

- **Node.js 18+** o **Python 3.10+**
- Una **cuenta de Anthropic** ([regístrate aquí](https://platform.claude.com/))

## Configuración

### 1. Crear una carpeta de proyecto

Crea un nuevo directorio para este quickstart:

```bash
mkdir my-agent && cd my-agent
```

Para tus propios proyectos, puedes ejecutar el SDK desde cualquier carpeta; tendrá acceso a los archivos de ese directorio y sus subdirectorios de forma predeterminada.

### 2. Instalar el SDK

Instala el paquete del Agent SDK para tu lenguaje:

#### TypeScript

```bash
npm install @anthropic-ai/claude-agent-sdk
```

#### Python (uv)

[uv Python package manager](https://docs.astral.sh/uv/) es un gestor de paquetes de Python rápido que maneja entornos virtuales automáticamente:
```bash
uv init && uv add claude-agent-sdk
```

#### Python (pip)

Primero crea un entorno virtual y luego instala:
```bash
python3 -m venv .venv && source .venv/bin/activate
pip3 install claude-agent-sdk
```

### 3. Configurar tu API key

Obtén una API key desde la [Claude Console](https://platform.claude.com/) y luego crea un archivo `.env` en el directorio de tu proyecto:

```bash
ANTHROPIC_API_KEY=your-api-key
```

El SDK también admite autenticación a través de proveedores de API de terceros:

- **Amazon Bedrock**: establece la variable de entorno `CLAUDE_CODE_USE_BEDROCK=1` y configura las credenciales de AWS
- **Google Vertex AI**: establece la variable de entorno `CLAUDE_CODE_USE_VERTEX=1` y configura las credenciales de Google Cloud
- **Microsoft Azure**: establece la variable de entorno `CLAUDE_CODE_USE_FOUNDRY=1` y configura las credenciales de Azure

Consulta las guías de configuración para [Bedrock](https://code.claude.com/docs/en/amazon-bedrock), [Vertex AI](https://code.claude.com/docs/en/google-vertex-ai) o [Azure AI Foundry](https://code.claude.com/docs/en/azure-ai-foundry) para más detalles.

> **Nota:** Salvo aprobación previa, Anthropic no permite que desarrolladores de terceros ofrezcan el inicio de sesión de claude.ai ni sus límites de uso para sus productos, incluyendo agentes construidos sobre el Claude Agent SDK. Utiliza en su lugar los métodos de autenticación por API key descritos en este documento.

## Crear un archivo con errores

Este quickstart te guía en la construcción de un agente capaz de encontrar y corregir errores en el código. Primero, necesitas un archivo con algunos errores intencionales para que el agente los corrija. Crea `utils.py` en el directorio `my-agent` y pega el siguiente código:

```python
def calculate_average(numbers):
    total = 0
    for num in numbers:
        total += num
    return total / len(numbers)


def get_user_name(user):
    return user["name"].upper()
```

Este código tiene dos errores:
1. `calculate_average([])` falla con división por cero
2. `get_user_name(None)` falla con un TypeError

## Construir un agente que encuentre y corrija errores

Crea `agent.py` si usas el SDK de Python, o `agent.ts` para TypeScript:

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, ResultMessage


async def main():
    # Agentic loop: streams messages as Claude works
    async for message in query(
        prompt="Review utils.py for bugs that would cause crashes. Fix any issues you find.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Glob"],  # Tools Claude can use
            permission_mode="acceptEdits",  # Auto-approve file edits
        ),
    ):
        # Print human-readable output
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if hasattr(block, "text"):
                    print(block.text)  # Claude's reasoning
                elif hasattr(block, "name"):
                    print(f"Tool: {block.name}")  # Tool being called
        elif isinstance(message, ResultMessage):
            print(f"Done: {message.subtype}")  # Final result


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Agentic loop: streams messages as Claude works
for await (const message of query({
  prompt: "Review utils.py for bugs that would cause crashes. Fix any issues you find.",
  options: {
    allowedTools: ["Read", "Edit", "Glob"], // Tools Claude can use
    permissionMode: "acceptEdits" // Auto-approve file edits
  }
})) {
  // Print human-readable output
  if (message.type === "assistant" && message.message?.content) {
    for (const block of message.message.content) {
      if ("text" in block) {
        console.log(block.text); // Claude's reasoning
      } else if ("name" in block) {
        console.log(`Tool: ${block.name}`); // Tool being called
      }
    }
  } else if (message.type === "result") {
    console.log(`Done: ${message.subtype}`); // Final result
  }
}
```

Este código tiene tres partes principales:

1. **`query`**: el punto de entrada principal que crea el agentic loop. Devuelve un iterador asíncrono, por lo que usas `async for` para recibir mensajes en streaming a medida que Claude trabaja. Consulta la API completa en la referencia del SDK de Python o TypeScript.

2. **`prompt`**: lo que quieres que Claude haga. Claude determina qué herramientas usar según la tarea.

3. **`options`**: configuración para el agente. Este ejemplo usa `allowedTools` para aprobar previamente `Read`, `Edit` y `Glob`, y `permissionMode: "acceptEdits"` para aprobar automáticamente los cambios en archivos. Otras opciones incluyen `systemPrompt`, `mcpServers` y más. Consulta todas las opciones en la referencia del SDK de Python o TypeScript.

El bucle `async for` continúa ejecutándose mientras Claude piensa, llama herramientas, observa resultados y decide qué hacer a continuación. Cada iteración produce un mensaje: el razonamiento de Claude, una llamada a herramienta, un resultado de herramienta o el resultado final. El SDK se encarga de la orquestación (ejecución de herramientas, gestión del contexto, reintentos) para que simplemente consumas el stream. El bucle termina cuando Claude completa la tarea o encuentra un error.

El manejo de mensajes dentro del bucle filtra para mostrar solo la salida legible por humanos. Sin filtrado, verías objetos de mensaje en bruto, incluyendo la inicialización del sistema y el estado interno, lo cual es útil para depuración pero ruidoso en otros casos.

> **Nota:** Este ejemplo usa streaming para mostrar el progreso en tiempo real. Si no necesitas salida en vivo (por ejemplo, para tareas en segundo plano o pipelines de CI), puedes recopilar todos los mensajes de una vez. Consulta [Streaming vs. single-turn mode](./Guides/Streaming%20Input.md) para más detalles.

### Ejecutar tu agente

Tu agente está listo. Ejecútalo con el siguiente comando:

#### Python

```bash
python3 agent.py
```

#### TypeScript

```bash
npx tsx agent.ts
```

Después de ejecutarlo, revisa `utils.py`. Verás código defensivo que maneja listas vacías y usuarios nulos. Tu agente de forma autónoma:

1. **Leyó** `utils.py` para entender el código
2. **Analizó** la lógica e identificó los casos límite que causarían fallos
3. **Editó** el archivo para agregar el manejo de errores adecuado

Esto es lo que hace diferente al Agent SDK: Claude ejecuta las herramientas directamente en lugar de pedirte que las implementes.

> **Nota:** Si ves "API key not found", asegúrate de haber definido la variable de entorno `ANTHROPIC_API_KEY` en tu archivo `.env` o en el entorno del shell. Consulta la [guía completa de resolución de problemas](https://code.claude.com/docs/en/troubleshooting) para obtener más ayuda.

### Prueba otros prompts

Ahora que tu agente está configurado, prueba algunos prompts diferentes:

- `"Add docstrings to all functions in utils.py"`
- `"Add type hints to all functions in utils.py"`
- `"Create a README.md documenting the functions in utils.py"`

### Personaliza tu agente

Puedes modificar el comportamiento de tu agente cambiando las opciones. Aquí hay algunos ejemplos:

**Agregar capacidad de búsqueda web:**

**Python**
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Glob", "WebSearch"], permission_mode="acceptEdits"
)
```

**TypeScript**
```typescript
const _ = {
  options: {
    allowedTools: ["Read", "Edit", "Glob", "WebSearch"],
    permissionMode: "acceptEdits"
  }
};
```

**Darle a Claude un system prompt personalizado:**

**Python**
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Glob"],
    permission_mode="acceptEdits",
    system_prompt="You are a senior Python developer. Always follow PEP 8 style guidelines.",
)
```

**TypeScript**
```typescript
const _ = {
  options: {
    allowedTools: ["Read", "Edit", "Glob"],
    permissionMode: "acceptEdits",
    systemPrompt: "You are a senior Python developer. Always follow PEP 8 style guidelines."
  }
};
```

**Ejecutar comandos en el terminal:**

**Python**
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Glob", "Bash"], permission_mode="acceptEdits"
)
```

**TypeScript**
```typescript
const _ = {
  options: {
    allowedTools: ["Read", "Edit", "Glob", "Bash"],
    permissionMode: "acceptEdits"
  }
};
```

Con `Bash` habilitado, prueba: `"Write unit tests for utils.py, run them, and fix any failures"`

## Conceptos clave

**Tools** controla lo que tu agente puede hacer:

| Tools | Lo que el agente puede hacer |
|-------|------------------------------|
| `Read`, `Glob`, `Grep` | Análisis de solo lectura |
| `Read`, `Edit`, `Glob` | Analizar y modificar código |
| `Read`, `Edit`, `Bash`, `Glob`, `Grep` | Automatización completa |

**Permission modes** controla cuánta supervisión humana deseas:

| Mode | Comportamiento | Caso de uso |
|------|----------------|-------------|
| `acceptEdits` | Aprueba automáticamente las ediciones de archivos, solicita confirmación para otras acciones | Flujos de trabajo de desarrollo confiables |
| `dontAsk` (solo TypeScript) | Deniega todo lo que no esté en `allowedTools` | Agentes headless con restricciones estrictas |
| `bypassPermissions` | Ejecuta todas las herramientas sin solicitudes de confirmación | CI en sandbox, entornos completamente confiables |
| `default` | Requiere un callback `canUseTool` para gestionar la aprobación | Flujos de aprobación personalizados |

El ejemplo anterior usa el modo `acceptEdits`, que aprueba automáticamente las operaciones de archivos para que el agente pueda ejecutarse sin solicitudes interactivas. Si deseas pedir aprobación a los usuarios, usa el modo `default` y proporciona un [callback `canUseTool`](./Guides/Handle%20approvals%20and%20user%20input.md) que recopile la entrada del usuario. Para mayor control, consulta [Permissions](./Guides/Configure%20permissions.md).

## Próximos pasos

Ahora que has creado tu primer agente, aprende cómo ampliar sus capacidades y adaptarlo a tu caso de uso:

- **[Permissions](./Guides/Configure%20permissions.md)**: controla lo que tu agente puede hacer y cuándo necesita aprobación
- **[Hooks](./Guides/Intercept%20and%20control%20agent%20behavior%20with%20hooks.md)**: ejecuta código personalizado antes o después de las llamadas a herramientas
- **[Sessions](./Core%20concepts/Work%20with%20sessions.md)**: construye agentes multi-turno que mantienen el contexto
- **[MCP servers](./Guides/Connect%20to%20external%20tools%20with%20MCP.md)**: conéctate a bases de datos, navegadores, APIs y otros sistemas externos
- **[Hosting](./Guides/Hosting%20the%20Agent%20SDK.md)**: despliega agentes en Docker, la nube y CI/CD
- **[Example agents](https://github.com/anthropics/claude-agent-sdk-demos)**: ve ejemplos completos: asistente de correo electrónico, agente de investigación y más
