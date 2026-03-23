# Referencia Python — Claude Agent SDK

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
- Python 3.10+
- Una API key de Anthropic (obtener en https://platform.claude.com/)

### Con uv (recomendado)
```bash
mkdir mi-agente && cd mi-agente
uv init && uv add claude-agent-sdk
```

### Con pip
```bash
mkdir mi-agente && cd mi-agente
python3 -m venv .venv && source .venv/bin/activate
pip install claude-agent-sdk
```

### Archivo .env
```bash
ANTHROPIC_API_KEY=tu-api-key-aqui
```

> El SDK carga automáticamente el `.env` del directorio de trabajo.

---

## Agente básico {#agente-basico}

El patrón mínimo: importar `query`, definir opciones, iterar mensajes.

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, ResultMessage


async def main():
    async for message in query(
        prompt="Revisa los archivos .py de este directorio y dime qué hace cada uno.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep"],
            permission_mode="default",
        ),
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if hasattr(block, "text"):
                    print(block.text)
        elif isinstance(message, ResultMessage):
            print(f"\nTerminado: {message.subtype}")
            if message.total_cost_usd:
                print(f"Costo: ${message.total_cost_usd:.4f}")


asyncio.run(main())
```

### Ejecutar
```bash
# Con uv
uv run python agent.py

# Con pip / venv activo
python agent.py
```

---

## Opciones de configuración {#opciones}

`ClaudeAgentOptions` acepta estos parámetros clave:

```python
from claude_agent_sdk import ClaudeAgentOptions

options = ClaudeAgentOptions(
    # Herramientas que Claude puede usar
    allowed_tools=["Read", "Edit", "Glob", "Grep", "Bash", "WebSearch", "WebFetch"],

    # Herramientas explícitamente bloqueadas (opcional)
    disallowed_tools=["Bash"],

    # Modo de permisos
    # "default"           → pide confirmación para acciones sensibles
    # "acceptEdits"       → aprueba ediciones de archivo automáticamente
    # "bypassPermissions" → ejecuta todo sin preguntar (solo en entornos seguros)
    permission_mode="acceptEdits",

    # Prompt de sistema que define el rol del agente (opcional)
    system_prompt="Eres un experto en Python. Siempre sigue PEP 8.",

    # Modelo a usar (por defecto usa el más reciente disponible)
    model="claude-sonnet-4-6",

    # Número máximo de turnos del loop (opcional)
    max_turns=10,

    # Presupuesto máximo en USD (opcional)
    max_budget_usd=0.50,
)
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
| `Agent` | Invoca subagentes (requerido para usar `agents=`) |

---

## Manejo de mensajes {#mensajes}

El loop de `query()` emite distintos tipos de mensajes. Filtra los que te interesan:

```python
from claude_agent_sdk import (
    AssistantMessage,
    ResultMessage,
    SystemMessage,
    TextBlock,
    ToolUseBlock,
)

async for message in query(prompt="...", options=options):
    # Respuesta y razonamiento de Claude
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text)
            elif isinstance(block, ToolUseBlock):
                print(f"Usando herramienta: {block.name}")

    # Resultado final con métricas
    elif isinstance(message, ResultMessage):
        print(f"Estado: {message.subtype}")          # "success" o "error_*"
        print(f"Resultado: {message.result}")
        print(f"Costo: ${message.total_cost_usd:.4f}")
        session_id = message.session_id              # Guardar para continuar después
```

---

## Sesiones multi-turno {#sesiones}

Para agentes conversacionales que mantienen contexto entre prompts.

### Con ClaudeSDKClient (automático)
```python
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, AssistantMessage, TextBlock


async def main():
    options = ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Glob"])

    async with ClaudeSDKClient(options=options) as client:
        # Primera consulta — crea la sesión
        await client.query("Analiza el módulo de autenticación")
        async for message in client.receive_response():
            if isinstance(message, AssistantMessage):
                for block in message.content:
                    if isinstance(block, TextBlock):
                        print(block.text)

        # Segunda consulta — continúa automáticamente la misma sesión
        await client.query("Ahora refactorízalo para usar JWT")
        async for message in client.receive_response():
            if isinstance(message, AssistantMessage):
                for block in message.content:
                    if isinstance(block, TextBlock):
                        print(block.text)


asyncio.run(main())
```

### Retomar sesión por ID
```python
# Primera ejecución — captura el session_id
session_id = None
async for message in query(prompt="Analiza el código", options=options):
    if isinstance(message, ResultMessage):
        session_id = message.session_id

# Segunda ejecución — retoma donde quedó
async for message in query(
    prompt="Ahora implementa las mejoras que sugeriste",
    options=ClaudeAgentOptions(
        resume=session_id,
        allowed_tools=["Read", "Edit", "Write"],
    ),
):
    if isinstance(message, ResultMessage):
        print(message.result)
```

---

## Subagentes {#subagentes}

Los subagentes son instancias separadas que el agente principal puede lanzar para tareas especializadas. Son útiles para aislar contexto, correr tareas en paralelo, o aplicar instrucciones específicas.

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition


async def main():
    async for message in query(
        prompt="Revisa el módulo de autenticación: busca problemas de seguridad y también corre los tests",
        options=ClaudeAgentOptions(
            # "Agent" es obligatorio para poder invocar subagentes
            allowed_tools=["Read", "Grep", "Glob", "Agent"],
            agents={
                "revisor-seguridad": AgentDefinition(
                    description="Especialista en seguridad. Úsalo para revisar código en busca de vulnerabilidades.",
                    prompt="""Eres un experto en seguridad de aplicaciones.
Analiza el código buscando:
- Vulnerabilidades de inyección (SQL, comandos, etc.)
- Manejo inseguro de credenciales o tokens
- Problemas de autenticación y autorización
Sé específico en tus hallazgos.""",
                    tools=["Read", "Grep", "Glob"],  # solo lectura
                    model="sonnet",
                ),
                "ejecutor-tests": AgentDefinition(
                    description="Ejecutor de tests. Úsalo para correr la suite de pruebas y analizar resultados.",
                    prompt="Ejecuta los tests del proyecto y reporta cuáles fallan y por qué.",
                    tools=["Bash", "Read", "Grep"],  # puede ejecutar comandos
                ),
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

> **Nota:** Los subagentes no pueden lanzar sus propios subagentes. No incluyas `Agent` en el `tools` de un subagente.

---

## Herramientas custom (MCP) {#herramientas-custom}

Extiende al agente con tus propias herramientas usando el protocolo MCP integrado.

```python
import asyncio
from claude_agent_sdk import query, tool, create_sdk_mcp_server, ClaudeAgentOptions
from typing import Any
import aiohttp


# Define una herramienta con el decorador @tool
@tool(
    "obtener_clima",
    "Obtiene la temperatura actual para una ubicación",
    {"ciudad": str, "pais": str},
)
async def obtener_clima(args: dict[str, Any]) -> dict[str, Any]:
    # Tu lógica aquí — llama APIs, bases de datos, lo que necesites
    return {
        "content": [
            {"type": "text", "text": f"Clima en {args['ciudad']}: 22°C, soleado"}
        ]
    }


# Crea el servidor MCP con tus herramientas
servidor_custom = create_sdk_mcp_server(
    name="mis-herramientas",
    version="1.0.0",
    tools=[obtener_clima],
)


# Las herramientas custom requieren streaming input (generador async)
async def mensajes():
    yield {
        "type": "user",
        "message": {"role": "user", "content": "¿Qué clima hace en Madrid, España?"},
    }


async def main():
    async for message in query(
        prompt=mensajes(),  # generador, no string
        options=ClaudeAgentOptions(
            mcp_servers={"mis-herramientas": servidor_custom},
            # El nombre de la herramienta sigue el formato: mcp__{servidor}__{herramienta}
            allowed_tools=["mcp__mis-herramientas__obtener_clima"],
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

---

## System prompts {#system-prompts}

El system prompt define la personalidad y restricciones del agente. Úsalo para darle un rol específico:

```python
# Agente de análisis de código
system_prompt = """Eres un senior developer con 10 años de experiencia.
Cuando analices código:
- Identifica problemas de rendimiento y seguridad
- Sugiere mejoras concretas con ejemplos
- Explica el "por qué" de cada sugerencia
- Usa terminología técnica precisa"""

# Agente de escritura de documentación
system_prompt = """Eres un technical writer experto.
Escribe documentación clara y concisa siguiendo estas reglas:
- Usa ejemplos de código para cada concepto
- Estructura con headers jerárquicos
- Escribe en español a menos que se indique lo contrario"""

# Agente de automatización de archivos
system_prompt = """Eres un asistente de automatización.
- Antes de modificar cualquier archivo, muestra un resumen de los cambios planeados
- Crea backups antes de editar archivos críticos
- Reporta cada acción completada"""
```

---

## Ejemplos por caso de uso {#ejemplos}

### Agente de análisis de código
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep"],
    permission_mode="default",
    system_prompt="Eres un revisor de código experto. Analiza sin modificar.",
)
prompt = "Revisa todos los archivos Python de este proyecto y lista los problemas que encuentres."
```

### Agente de corrección automática
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Glob", "Bash"],
    permission_mode="acceptEdits",
    system_prompt="Eres un developer Python. Sigue PEP 8 y añade type hints.",
)
prompt = "Corrige todos los bugs en utils.py y añade manejo de errores donde falte."
```

### Agente con búsqueda web
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "WebSearch", "WebFetch", "Write"],
    permission_mode="acceptEdits",
)
prompt = "Investiga las mejores prácticas de autenticación JWT en 2024 y crea un resumen en SECURITY.md"
```

### Agente de automatización de terminal
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Bash", "Glob"],
    permission_mode="bypassPermissions",  # solo en entornos controlados
    system_prompt="Eres un DevOps engineer. Automatiza el setup del proyecto.",
)
prompt = "Instala las dependencias, corre los tests y genera el reporte de cobertura."
```
