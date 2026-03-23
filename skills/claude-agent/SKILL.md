---
name: claude-agent
description: Crea un agente completo con el Claude Agent SDK desde cero — setup, código funcional, configuración de herramientas, permisos, y patrones avanzados. Úsalo cuando el usuario quiera construir un agente con el Agent SDK, crear un bot autónomo, automatizar tareas con Claude, usar `query()` o `ClaudeSDKClient`, trabajar con subagentes, herramientas custom, sesiones o MCP. Activa esta skill aunque el usuario no mencione explícitamente "Agent SDK" — si quiere que Claude haga algo de forma autónoma en código, aquí está la guía.
---

# claude-agent

Ayuda al usuario a construir un agente funcional con el Claude Agent SDK. El objetivo es que al terminar tengan archivos reales en su proyecto que puedan ejecutar.

## Paso 1 — Recopilar parámetros

Antes de escribir código, determina estos dos datos. Si el usuario ya los mencionó, úsalos directamente; si no, pregunta:

1. **Lenguaje**: ¿Python o TypeScript?
2. **Propósito del agente**: ¿Qué quieres que haga? (ej: analizar código, buscar en la web, automatizar archivos, conectar a una API, etc.)

Con eso, ya puedes construir el agente.

## Paso 2 — Leer la guía del lenguaje

Lee el archivo de referencia correspondiente antes de escribir cualquier código:

- **Python** → `references/python.md`
- **TypeScript** → `references/typescript.md`

Cada archivo tiene el setup completo, patrones de código probados, y ejemplos para los casos más comunes.

## Paso 3 — Crear el agente

Sigue esta secuencia en el directorio del usuario:

### 3a. Setup del proyecto

Crea la carpeta y los archivos base según el lenguaje (ver referencia). Incluye siempre:
- Archivo de instalación de dependencias
- Archivo `.env` con `ANTHROPIC_API_KEY=tu-api-key` (como plantilla, sin la clave real)
- Archivo principal del agente

### 3b. Elegir herramientas y permisos

Selecciona las herramientas según lo que el agente necesita hacer:

| Objetivo | Herramientas sugeridas | Permission mode |
|----------|----------------------|-----------------|
| Leer y analizar archivos | `Read`, `Glob`, `Grep` | `default` |
| Modificar código | `Read`, `Edit`, `Glob` | `acceptEdits` |
| Automatización completa | `Read`, `Edit`, `Bash`, `Glob`, `Grep` | `acceptEdits` |
| Buscar en internet | `Read`, `WebSearch`, `WebFetch` | `default` |
| Agente con subagentes | añadir `Agent` a cualquier lista | igual al anterior |

El `permission_mode` controla cuánta autonomía tiene el agente:
- `acceptEdits` — aprueba cambios de archivo automáticamente (ideal para desarrollo)
- `bypassPermissions` — ejecuta todo sin preguntar (solo en entornos controlados)
- `default` — pide confirmación para acciones sensibles

### 3c. Escribir el código del agente

Escribe el código completo y funcional basándote en la referencia del lenguaje. El código debe:
- Importar correctamente el SDK
- Definir las opciones del agente (herramientas, permisos, system prompt si aplica)
- Iterar sobre los mensajes del loop agentico
- Mostrar el output de forma legible

### 3d. Personalización según el propósito

Ajusta el prompt inicial y el `system_prompt` al caso de uso específico del usuario. Un buen system prompt define el rol y las restricciones del agente. Ver ejemplos en la referencia.

## Paso 4 — Instrucciones para ejecutar

Muestra al usuario exactamente cómo correr su agente:
- Cómo instalar dependencias
- Cómo configurar la API key
- El comando para ejecutar

## Patrones avanzados

Si el usuario necesita algo más complejo, lee la sección correspondiente en la referencia del lenguaje:

- **Sesiones multi-turno** — para agentes conversacionales que mantienen contexto
- **Subagentes** — para delegar tareas especializadas en paralelo
- **Herramientas custom (MCP)** — para conectar el agente a APIs o servicios externos
- **Streaming** — para mostrar el progreso en tiempo real
- **Checkpointing** — para guardar y revertir cambios de archivos

## Tono y estilo

El usuario está aprendiendo. Explica brevemente *por qué* cada decisión (¿por qué estas herramientas? ¿por qué este permission mode?). No abrumes con teoría — el objetivo es que tengan código corriendo lo antes posible, y que entiendan lo suficiente para modificarlo.
