# Hosting the Agent SDK

Despliega y aloja el Claude Agent SDK en entornos de producción

---

El Claude Agent SDK se diferencia de las APIs de LLM tradicionales sin estado en que mantiene el estado conversacional y ejecuta comandos en un entorno persistente. Esta guía cubre la arquitectura, las consideraciones de alojamiento y las mejores prácticas para desplegar agentes basados en el SDK en producción.

> **Info:** Para el endurecimiento de seguridad más allá del sandboxing básico (incluyendo controles de red, gestión de credenciales y opciones de aislamiento), consulta [Secure Deployment](/docs/en/agent-sdk/secure-deployment).

## Requisitos de Alojamiento

### Sandboxing Basado en Contenedores

Por seguridad y aislamiento, el SDK debe ejecutarse dentro de un entorno de contenedor con sandbox. Esto proporciona aislamiento de procesos, límites de recursos, control de red y sistemas de archivos efímeros.

El SDK también admite [configuración programática del sandbox](/docs/en/agent-sdk/typescript#sandbox-settings) para la ejecución de comandos.

### Requisitos del Sistema

Cada instancia del SDK requiere:

- **Dependencias de tiempo de ejecución**
  - Python 3.10+ (para el SDK de Python) o Node.js 18+ (para el SDK de TypeScript)
  - Node.js (requerido por Claude Code CLI)
  - Claude Code CLI: `npm install -g @anthropic-ai/claude-code`

- **Asignación de recursos**
  - Recomendado: 1GiB de RAM, 5GiB de disco y 1 CPU (ajusta según las necesidades de tu tarea)

- **Acceso a red**
  - HTTPS saliente hacia `api.anthropic.com`
  - Opcional: acceso a servidores MCP o herramientas externas

## Entendiendo la Arquitectura del SDK

A diferencia de las llamadas a API sin estado, el Claude Agent SDK opera como un **proceso de larga duración** que:
- **Ejecuta comandos** en un entorno de shell persistente
- **Gestiona operaciones de archivos** dentro de un directorio de trabajo
- **Maneja la ejecución de herramientas** con contexto de interacciones previas

## Opciones de Proveedores de Sandbox

Varios proveedores se especializan en entornos de contenedores seguros para la ejecución de código de IA:

- **[Modal Sandbox](https://modal.com/docs/guide/sandbox)** - [implementación de ejemplo](https://modal.com/docs/examples/claude-slack-gif-creator)
- **[Cloudflare Sandboxes](https://github.com/cloudflare/sandbox-sdk)**
- **[Daytona](https://www.daytona.io/)**
- **[E2B](https://e2b.dev/)**
- **[Fly Machines](https://fly.io/docs/machines/)**
- **[Vercel Sandbox](https://vercel.com/docs/functions/sandbox)**

Para opciones auto-alojadas (Docker, gVisor, Firecracker) y configuración detallada de aislamiento, consulta [Isolation Technologies](/docs/en/agent-sdk/secure-deployment#isolation-technologies).

## Patrones de Despliegue en Producción

### Patrón 1: Sesiones Efímeras

Crea un nuevo contenedor para cada tarea del usuario y destrúyelo al completarse.

Ideal para tareas puntuales; el usuario puede seguir interactuando con la IA mientras la tarea se completa, pero una vez finalizada el contenedor se destruye.

**Ejemplos:**
- Investigación y corrección de errores: depurar y resolver un problema específico con el contexto relevante
- Procesamiento de facturas: extraer y estructurar datos de recibos/facturas para sistemas contables
- Tareas de traducción: traducir documentos o lotes de contenido entre idiomas
- Procesamiento de imágenes/video: aplicar transformaciones, optimizaciones o extraer metadatos de archivos multimedia

### Patrón 2: Sesiones de Larga Duración

Mantén instancias de contenedores persistentes para tareas de larga duración. A menudo se ejecutan _múltiples_ procesos de Claude Agent dentro del contenedor según la demanda.

Ideal para agentes proactivos que toman acción sin la intervención del usuario, agentes que sirven contenido o agentes que procesan grandes volúmenes de mensajes.

**Ejemplos:**
- Agente de correo electrónico: monitorea correos entrantes y de forma autónoma los clasifica, responde o toma acciones según el contenido
- Constructor de sitios: aloja sitios web personalizados por usuario con capacidades de edición en vivo servidas a través de puertos del contenedor
- Chatbots de alta frecuencia: maneja flujos de mensajes continuos desde plataformas como Slack donde los tiempos de respuesta rápidos son críticos

### Patrón 3: Sesiones Híbridas

Contenedores efímeros que se hidratan con historial y estado, posiblemente desde una base de datos o desde las funciones de reanudación de sesión del SDK.

Ideal para contenedores con interacción intermitente del usuario que inicia el trabajo y se detiene al completarse, pero puede reanudarse.

**Ejemplos:**
- Gestor de proyectos personal: ayuda a gestionar proyectos en curso con revisiones intermitentes, mantiene el contexto de tareas, decisiones y progreso
- Investigación profunda: realiza tareas de investigación de varias horas, guarda los hallazgos y reanuda la investigación cuando el usuario regresa
- Agente de soporte al cliente: maneja tickets de soporte que abarcan múltiples interacciones, carga el historial del ticket y el contexto del cliente

### Patrón 4: Contenedores Únicos

Ejecuta múltiples procesos del Claude Agent SDK en un único contenedor global.

Ideal para agentes que deben colaborar estrechamente entre sí. Este es probablemente el patrón menos popular, ya que tendrás que evitar que los agentes se sobreescriban entre sí.

**Ejemplos:**
- **Simulaciones**: agentes que interactúan entre sí en simulaciones como videojuegos.

## Preguntas Frecuentes

### ¿Cómo me comunico con mis sandboxes?
Al alojar en contenedores, expone puertos para comunicarte con tus instancias del SDK. Tu aplicación puede exponer endpoints HTTP/WebSocket para clientes externos mientras el SDK se ejecuta internamente dentro del contenedor.

### ¿Cuál es el costo de alojar un contenedor?
El costo dominante de servir agentes son los tokens; los contenedores varían según lo que aprovisiones, pero el costo mínimo es de aproximadamente 5 centavos por hora en ejecución.

### ¿Cuándo debo apagar los contenedores inactivos vs. mantenerlos activos?
Esto probablemente depende del proveedor; diferentes proveedores de sandbox te permiten establecer distintos criterios de tiempo de espera por inactividad tras los cuales un sandbox puede detenerse.
Deberás ajustar este tiempo de espera en función de la frecuencia con que esperas que el usuario responda.

### ¿Con qué frecuencia debo actualizar el Claude Code CLI?
El Claude Code CLI usa versionado semver, por lo que cualquier cambio que rompa la compatibilidad estará versionado.

### ¿Cómo monitoreo la salud del contenedor y el rendimiento del agente?
Dado que los contenedores son simplemente servidores, la misma infraestructura de logging que usas para el backend funcionará para los contenedores.

### ¿Cuánto tiempo puede ejecutarse una sesión de agente antes de que expire?
Una sesión de agente no expirará, pero considera establecer una propiedad `maxTurns` para evitar que Claude quede atrapado en un bucle.

## Próximos Pasos

- [Secure Deployment](/docs/en/agent-sdk/secure-deployment) - Controles de red, gestión de credenciales y endurecimiento del aislamiento
- [TypeScript SDK - Sandbox Settings](/docs/en/agent-sdk/typescript#sandbox-settings) - Configurar el sandbox de forma programática
- [Sessions Guide](/docs/en/agent-sdk/sessions) - Aprende sobre la gestión de sesiones
- [Permissions](/docs/en/agent-sdk/permissions) - Configura los permisos de herramientas
- [Cost Tracking](/docs/en/agent-sdk/cost-tracking) - Monitorea el uso de la API
- [MCP Integration](/docs/en/agent-sdk/mcp) - Extiende con herramientas personalizadas
