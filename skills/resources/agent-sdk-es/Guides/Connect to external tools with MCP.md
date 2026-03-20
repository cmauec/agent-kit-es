# Conectar con herramientas externas mediante MCP

Configura servidores MCP para ampliar tu agente con herramientas externas. Cubre tipos de transporte, búsqueda de herramientas para conjuntos grandes, autenticación y manejo de errores.

---

El [Model Context Protocol (MCP)](https://modelcontextprotocol.io/docs/getting-started/intro) es un estándar abierto para conectar agentes de IA con herramientas externas y fuentes de datos. Con MCP, tu agente puede consultar bases de datos, integrarse con APIs como Slack y GitHub, y conectarse a otros servicios sin necesidad de escribir implementaciones de herramientas personalizadas.

Los servidores MCP pueden ejecutarse como procesos locales, conectarse mediante HTTP, o ejecutarse directamente dentro de tu aplicación SDK.

## Inicio rápido

Este ejemplo se conecta al servidor MCP de la [documentación de Claude Code](https://code.claude.com/docs) usando [transporte HTTP](#httpsse-servers) y utiliza [`allowedTools`](#allow-mcp-tools) con un comodín para permitir todas las herramientas del servidor.

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Use the docs MCP server to explain what hooks are in Claude Code",
  options: {
    mcpServers: {
      "claude-code-docs": {
        type: "http",
        url: "https://code.claude.com/docs/mcp"
      }
    },
    allowedTools: ["mcp__claude-code-docs__*"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage


async def main():
    options = ClaudeAgentOptions(
        mcp_servers={
            "claude-code-docs": {
                "type": "http",
                "url": "https://code.claude.com/docs/mcp",
            }
        },
        allowed_tools=["mcp__claude-code-docs__*"],
    )

    async for message in query(
        prompt="Use the docs MCP server to explain what hooks are in Claude Code",
        options=options,
    ):
        if isinstance(message, ResultMessage) and message.subtype == "success":
            print(message.result)


asyncio.run(main())
```

El agente se conecta al servidor de documentación, busca información sobre hooks y devuelve los resultados.

## Agregar un servidor MCP

Puedes configurar servidores MCP en código al llamar a `query()`, o en un archivo `.mcp.json` que el SDK carga automáticamente.

### En código

Pasa los servidores MCP directamente en la opción `mcpServers`:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "List files in my project",
  options: {
    mcpServers: {
      filesystem: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
      }
    },
    allowedTools: ["mcp__filesystem__*"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage


async def main():
    options = ClaudeAgentOptions(
        mcp_servers={
            "filesystem": {
                "command": "npx",
                "args": [
                    "-y",
                    "@modelcontextprotocol/server-filesystem",
                    "/Users/me/projects",
                ],
            }
        },
        allowed_tools=["mcp__filesystem__*"],
    )

    async for message in query(prompt="List files in my project", options=options):
        if isinstance(message, ResultMessage) and message.subtype == "success":
            print(message.result)


asyncio.run(main())
```

### Desde un archivo de configuración

Crea un archivo `.mcp.json` en la raíz de tu proyecto. El SDK lo carga automáticamente:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    }
  }
}
```

## Permitir herramientas MCP

Las herramientas MCP requieren permiso explícito antes de que Claude pueda usarlas. Sin permiso, Claude verá que hay herramientas disponibles pero no podrá invocarlas.

### Convención de nombres de herramientas

Las herramientas MCP siguen el patrón de nombres `mcp__<nombre-servidor>__<nombre-herramienta>`. Por ejemplo, un servidor GitHub llamado `"github"` con una herramienta `list_issues` se convierte en `mcp__github__list_issues`.

### Otorgar acceso con allowedTools

Usa `allowedTools` para especificar qué herramientas MCP puede usar Claude:

```typescript hidelines={1,-1}
const _ = {
  options: {
    mcpServers: {
      // your servers
    },
    allowedTools: [
      "mcp__github__*", // All tools from the github server
      "mcp__db__query", // Only the query tool from db server
      "mcp__slack__send_message" // Only send_message from slack server
    ]
  }
};
```

Los comodines (`*`) permiten autorizar todas las herramientas de un servidor sin listar cada una individualmente.

### Alternativa: cambiar el modo de permisos

En lugar de listar las herramientas permitidas, puedes cambiar el modo de permisos para otorgar un acceso más amplio:

- `permissionMode: "acceptEdits"`: aprueba automáticamente el uso de herramientas (aún solicita confirmación para operaciones destructivas)
- `permissionMode: "bypassPermissions"`: omite todas las confirmaciones de seguridad, incluidas las de operaciones destructivas como eliminación de archivos o ejecución de comandos de shell. Úsalo con precaución, especialmente en producción. Este modo se propaga a los subagentes creados por la herramienta Agent.

```typescript hidelines={1,-1}
const _ = {
  options: {
    mcpServers: {
      // your servers
    },
    permissionMode: "acceptEdits" // No need for allowedTools
  }
};
```

Consulta [Permissions](/docs/en/agent-sdk/permissions) para más detalles sobre los modos de permisos.

### Descubrir las herramientas disponibles

Para ver qué herramientas ofrece un servidor MCP, consulta la documentación del servidor o conéctate a él e inspecciona el mensaje `system` init:

```typescript
for await (const message of query({ prompt: "...", options })) {
  if (message.type === "system" && message.subtype === "init") {
    console.log("Available MCP tools:", message.mcp_servers);
  }
}
```

## Tipos de transporte

Los servidores MCP se comunican con tu agente usando diferentes protocolos de transporte. Consulta la documentación del servidor para saber cuál soporta:

- Si la documentación te indica un **comando para ejecutar** (como `npx @modelcontextprotocol/server-github`), usa stdio
- Si la documentación te proporciona una **URL**, usa HTTP o SSE
- Si estás construyendo tus propias herramientas en código, usa un servidor MCP del SDK

### Servidores stdio

Procesos locales que se comunican mediante stdin/stdout. Úsalos para servidores MCP que ejecutas en la misma máquina:

#### En código

**TypeScript**
```typescript
const _ = {
  options: {
    mcpServers: {
      github: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-github"],
        env: {
          GITHUB_TOKEN: process.env.GITHUB_TOKEN
        }
      }
    },
    allowedTools: ["mcp__github__list_issues", "mcp__github__search_issues"]
  }
};
```

**Python**
```python
options = ClaudeAgentOptions(
    mcp_servers={
        "github": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": os.environ["GITHUB_TOKEN"]},
        }
    },
    allowed_tools=["mcp__github__list_issues", "mcp__github__search_issues"],
)
```

#### .mcp.json

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### Servidores HTTP/SSE

Usa HTTP o SSE para servidores MCP alojados en la nube y APIs remotas:

#### En código

**TypeScript**
```typescript
const _ = {
  options: {
    mcpServers: {
      "remote-api": {
        type: "sse",
        url: "https://api.example.com/mcp/sse",
        headers: {
          Authorization: `Bearer ${process.env.API_TOKEN}`
        }
      }
    },
    allowedTools: ["mcp__remote-api__*"]
  }
};
```

**Python**
```python
options = ClaudeAgentOptions(
    mcp_servers={
        "remote-api": {
            "type": "sse",
            "url": "https://api.example.com/mcp/sse",
            "headers": {"Authorization": f"Bearer {os.environ['API_TOKEN']}"},
        }
    },
    allowed_tools=["mcp__remote-api__*"],
)
```

#### .mcp.json

```json
{
  "mcpServers": {
    "remote-api": {
      "type": "sse",
      "url": "https://api.example.com/mcp/sse",
      "headers": {
        "Authorization": "Bearer ${API_TOKEN}"
      }
    }
  }
}
```

Para HTTP (sin streaming), usa `"type": "http"` en su lugar.

### Servidores MCP del SDK

Define herramientas personalizadas directamente en el código de tu aplicación en lugar de ejecutar un proceso de servidor separado. Consulta la [guía de herramientas personalizadas](/docs/en/agent-sdk/custom-tools) para detalles de implementación.

## Búsqueda de herramientas MCP

Cuando tienes muchas herramientas MCP configuradas, las definiciones de herramientas pueden consumir una parte significativa de tu ventana de contexto. La búsqueda de herramientas MCP resuelve esto cargando herramientas de forma dinámica bajo demanda, en lugar de precargarlas todas.

### Cómo funciona

La búsqueda de herramientas se ejecuta en modo automático por defecto. Se activa cuando las descripciones de tus herramientas MCP consumirían más del 10% de la ventana de contexto. Cuando se activa:

1. Las herramientas MCP se marcan con `defer_loading: true` en lugar de cargarse en el contexto de antemano
2. Claude usa una herramienta de búsqueda para descubrir las herramientas MCP relevantes cuando las necesita
3. Solo se cargan en el contexto las herramientas que Claude realmente necesita

La búsqueda de herramientas requiere modelos que soporten bloques `tool_reference`: Sonnet 4 y posteriores, u Opus 4 y posteriores. Los modelos Haiku no soportan la búsqueda de herramientas.

### Configurar la búsqueda de herramientas

Controla el comportamiento de la búsqueda de herramientas con la variable de entorno `ENABLE_TOOL_SEARCH`:

| Valor | Comportamiento |
|:------|:---------|
| `auto` | Se activa cuando las herramientas MCP superan el 10% del contexto (por defecto) |
| `auto:5` | Se activa con un umbral del 5% (personalizable) |
| `true` | Siempre habilitado |
| `false` | Deshabilitado; todas las herramientas MCP se cargan de antemano |

Establece el valor en la opción `env`:

**TypeScript**
```typescript
const options = {
  mcpServers: {
    // your MCP servers
  },
  env: {
    ENABLE_TOOL_SEARCH: "auto:5" // Enable at 5% threshold
  }
};
```

**Python**
```python
options = ClaudeAgentOptions(
    mcp_servers={...},  # your MCP servers
    env={
        "ENABLE_TOOL_SEARCH": "auto:5"  # Enable at 5% threshold
    },
)
```

## Autenticación

La mayoría de los servidores MCP requieren autenticación para acceder a servicios externos. Pasa las credenciales mediante variables de entorno en la configuración del servidor.

### Pasar credenciales mediante variables de entorno

Usa el campo `env` para pasar claves de API, tokens y otras credenciales al servidor MCP:

#### En código

**TypeScript**
```typescript
const _ = {
  options: {
    mcpServers: {
      github: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-github"],
        env: {
          GITHUB_TOKEN: process.env.GITHUB_TOKEN
        }
      }
    },
    allowedTools: ["mcp__github__list_issues"]
  }
};
```

**Python**
```python
options = ClaudeAgentOptions(
    mcp_servers={
        "github": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": os.environ["GITHUB_TOKEN"]},
        }
    },
    allowed_tools=["mcp__github__list_issues"],
)
```

#### .mcp.json

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

La sintaxis `${GITHUB_TOKEN}` expande las variables de entorno en tiempo de ejecución.

Consulta [List issues from a repository](#list-issues-from-a-repository) para ver un ejemplo funcional completo con registro de depuración.

### Cabeceras HTTP para servidores remotos

Para servidores HTTP y SSE, pasa las cabeceras de autenticación directamente en la configuración del servidor:

#### En código

**TypeScript**
```typescript
const _ = {
  options: {
    mcpServers: {
      "secure-api": {
        type: "http",
        url: "https://api.example.com/mcp",
        headers: {
          Authorization: `Bearer ${process.env.API_TOKEN}`
        }
      }
    },
    allowedTools: ["mcp__secure-api__*"]
  }
};
```

**Python**
```python
options = ClaudeAgentOptions(
    mcp_servers={
        "secure-api": {
            "type": "http",
            "url": "https://api.example.com/mcp",
            "headers": {"Authorization": f"Bearer {os.environ['API_TOKEN']}"},
        }
    },
    allowed_tools=["mcp__secure-api__*"],
)
```

#### .mcp.json

```json
{
  "mcpServers": {
    "secure-api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${API_TOKEN}"
      }
    }
  }
}
```

La sintaxis `${API_TOKEN}` expande las variables de entorno en tiempo de ejecución.

### Autenticación OAuth2

La [especificación MCP soporta OAuth 2.1](https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization) para autorización. El SDK no gestiona los flujos OAuth de forma automática, pero puedes pasar tokens de acceso mediante cabeceras una vez completado el flujo OAuth en tu aplicación:

**TypeScript**
```typescript
// After completing OAuth flow in your app
const accessToken = await getAccessTokenFromOAuthFlow();

const options = {
  mcpServers: {
    "oauth-api": {
      type: "http",
      url: "https://api.example.com/mcp",
      headers: {
        Authorization: `Bearer ${accessToken}`
      }
    }
  },
  allowedTools: ["mcp__oauth-api__*"]
};
```

**Python**
```python
# After completing OAuth flow in your app
access_token = await get_access_token_from_oauth_flow()

options = ClaudeAgentOptions(
    mcp_servers={
        "oauth-api": {
            "type": "http",
            "url": "https://api.example.com/mcp",
            "headers": {"Authorization": f"Bearer {access_token}"},
        }
    },
    allowed_tools=["mcp__oauth-api__*"],
)
```

## Ejemplos

### Listar issues de un repositorio

Este ejemplo se conecta al [servidor MCP de GitHub](https://github.com/modelcontextprotocol/servers/tree/main/src/github) para listar los issues recientes. El ejemplo incluye registro de depuración para verificar la conexión MCP y las llamadas a herramientas.

Antes de ejecutarlo, crea un [token de acceso personal de GitHub](https://github.com/settings/tokens) con el scope `repo` y configúralo como variable de entorno:

```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "List the 3 most recent issues in anthropics/claude-code",
  options: {
    mcpServers: {
      github: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-github"],
        env: {
          GITHUB_TOKEN: process.env.GITHUB_TOKEN
        }
      }
    },
    allowedTools: ["mcp__github__list_issues"]
  }
})) {
  // Verify MCP server connected successfully
  if (message.type === "system" && message.subtype === "init") {
    console.log("MCP servers:", message.mcp_servers);
  }

  // Log when Claude calls an MCP tool
  if (message.type === "assistant") {
    for (const block of message.content) {
      if (block.type === "tool_use" && block.name.startsWith("mcp__")) {
        console.log("MCP tool called:", block.name);
      }
    }
  }

  // Print the final result
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

**Python**
```python
import asyncio
import os
from claude_agent_sdk import (
    query,
    ClaudeAgentOptions,
    ResultMessage,
    SystemMessage,
    AssistantMessage,
)


async def main():
    options = ClaudeAgentOptions(
        mcp_servers={
            "github": {
                "command": "npx",
                "args": ["-y", "@modelcontextprotocol/server-github"],
                "env": {"GITHUB_TOKEN": os.environ["GITHUB_TOKEN"]},
            }
        },
        allowed_tools=["mcp__github__list_issues"],
    )

    async for message in query(
        prompt="List the 3 most recent issues in anthropics/claude-code",
        options=options,
    ):
        # Verify MCP server connected successfully
        if isinstance(message, SystemMessage) and message.subtype == "init":
            print("MCP servers:", message.data.get("mcp_servers"))

        # Log when Claude calls an MCP tool
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if hasattr(block, "name") and block.name.startswith("mcp__"):
                    print("MCP tool called:", block.name)

        # Print the final result
        if isinstance(message, ResultMessage) and message.subtype == "success":
            print(message.result)


asyncio.run(main())
```

### Consultar una base de datos

Este ejemplo utiliza el [servidor MCP de Postgres](https://github.com/modelcontextprotocol/servers/tree/main/src/postgres) para consultar una base de datos. La cadena de conexión se pasa como argumento al servidor. El agente descubre automáticamente el esquema de la base de datos, escribe la consulta SQL y devuelve los resultados:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Connection string from environment variable
const connectionString = process.env.DATABASE_URL;

for await (const message of query({
  // Natural language query - Claude writes the SQL
  prompt: "How many users signed up last week? Break it down by day.",
  options: {
    mcpServers: {
      postgres: {
        command: "npx",
        // Pass connection string as argument to the server
        args: ["-y", "@modelcontextprotocol/server-postgres", connectionString]
      }
    },
    // Allow only read queries, not writes
    allowedTools: ["mcp__postgres__query"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

**Python**
```python
import asyncio
import os
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage


async def main():
    # Connection string from environment variable
    connection_string = os.environ["DATABASE_URL"]

    options = ClaudeAgentOptions(
        mcp_servers={
            "postgres": {
                "command": "npx",
                # Pass connection string as argument to the server
                "args": [
                    "-y",
                    "@modelcontextprotocol/server-postgres",
                    connection_string,
                ],
            }
        },
        # Allow only read queries, not writes
        allowed_tools=["mcp__postgres__query"],
    )

    # Natural language query - Claude writes the SQL
    async for message in query(
        prompt="How many users signed up last week? Break it down by day.",
        options=options,
    ):
        if isinstance(message, ResultMessage) and message.subtype == "success":
            print(message.result)


asyncio.run(main())
```

## Manejo de errores

Los servidores MCP pueden fallar al conectarse por diversas razones: el proceso del servidor puede no estar instalado, las credenciales pueden ser inválidas, o un servidor remoto puede ser inalcanzable.

El SDK emite un mensaje `system` con subtipo `init` al inicio de cada consulta. Este mensaje incluye el estado de conexión de cada servidor MCP. Comprueba el campo `status` para detectar fallos de conexión antes de que el agente comience a trabajar:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Process data",
  options: {
    mcpServers: {
      "data-processor": dataServer
    }
  }
})) {
  if (message.type === "system" && message.subtype === "init") {
    const failedServers = message.mcp_servers.filter((s) => s.status !== "connected");

    if (failedServers.length > 0) {
      console.warn("Failed to connect:", failedServers);
    }
  }

  if (message.type === "result" && message.subtype === "error_during_execution") {
    console.error("Execution failed");
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, SystemMessage, ResultMessage


async def main():
    options = ClaudeAgentOptions(mcp_servers={"data-processor": data_server})

    async for message in query(prompt="Process data", options=options):
        if isinstance(message, SystemMessage) and message.subtype == "init":
            failed_servers = [
                s
                for s in message.data.get("mcp_servers", [])
                if s.get("status") != "connected"
            ]

            if failed_servers:
                print(f"Failed to connect: {failed_servers}")

        if (
            isinstance(message, ResultMessage)
            and message.subtype == "error_during_execution"
        ):
            print("Execution failed")


asyncio.run(main())
```

## Solución de problemas

### El servidor muestra estado "failed"

Comprueba el mensaje `init` para ver qué servidores fallaron al conectarse:

```typescript
if (message.type === "system" && message.subtype === "init") {
  for (const server of message.mcp_servers) {
    if (server.status === "failed") {
      console.error(`Server ${server.name} failed to connect`);
    }
  }
}
```

Causas comunes:

- **Variables de entorno faltantes**: asegúrate de que los tokens y credenciales requeridos estén configurados. Para servidores stdio, verifica que el campo `env` coincida con lo que espera el servidor.
- **Servidor no instalado**: para comandos `npx`, verifica que el paquete exista y que Node.js esté en tu PATH.
- **Cadena de conexión inválida**: para servidores de base de datos, verifica el formato de la cadena de conexión y que la base de datos sea accesible.
- **Problemas de red**: para servidores remotos HTTP/SSE, comprueba que la URL sea alcanzable y que los cortafuegos permitan la conexión.

### Las herramientas no se invocan

Si Claude ve las herramientas pero no las usa, verifica que hayas otorgado permiso con `allowedTools` o [cambiando el modo de permisos](#alternative-change-the-permission-mode):

```typescript hidelines={1,-1}
const _ = {
  options: {
    mcpServers: {
      // your servers
    },
    allowedTools: ["mcp__servername__*"] // Required for Claude to use the tools
  }
};
```

### Tiempos de espera de conexión

El SDK de MCP tiene un tiempo de espera predeterminado de 60 segundos para las conexiones del servidor. Si tu servidor tarda más en arrancar, la conexión fallará. Para servidores que necesitan más tiempo de inicio, considera:

- Usar un servidor más ligero si está disponible
- Pre-calentar el servidor antes de iniciar tu agente
- Revisar los logs del servidor para identificar causas de inicialización lenta

## Recursos relacionados

- **[Guía de herramientas personalizadas](/docs/en/agent-sdk/custom-tools)**: construye tu propio servidor MCP que se ejecuta en el mismo proceso que tu aplicación SDK
- **[Permissions](/docs/en/agent-sdk/permissions)**: controla qué herramientas MCP puede usar tu agente con `allowedTools` y `disallowedTools`
- **[Referencia del SDK TypeScript](/docs/en/agent-sdk/typescript)**: referencia completa de la API, incluidas las opciones de configuración MCP
- **[Referencia del SDK Python](/docs/en/agent-sdk/python)**: referencia completa de la API, incluidas las opciones de configuración MCP
- **[Directorio de servidores MCP](https://github.com/modelcontextprotocol/servers)**: explora los servidores MCP disponibles para bases de datos, APIs y más
