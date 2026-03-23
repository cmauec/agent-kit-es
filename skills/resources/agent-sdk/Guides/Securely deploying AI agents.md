# Despliegue seguro de agentes de IA

Una guía para asegurar despliegues de Claude Code y el Agent SDK con aislamiento, gestión de credenciales y controles de red

---

Claude Code y el Agent SDK son herramientas poderosas que pueden ejecutar código, acceder a archivos e interactuar con servicios externos en tu nombre. Como cualquier herramienta con estas capacidades, desplegarlas de forma reflexiva garantiza que obtengas los beneficios mientras mantienes los controles adecuados.

A diferencia del software tradicional que sigue rutas de código predeterminadas, estas herramientas generan sus acciones de forma dinámica según el contexto y los objetivos. Esta flexibilidad es lo que las hace útiles, pero también significa que su comportamiento puede verse influenciado por el contenido que procesan: archivos, páginas web o entradas del usuario. Esto se conoce a veces como prompt injection. Por ejemplo, si el README de un repositorio contiene instrucciones inusuales, Claude Code podría incorporarlas en sus acciones de maneras que el operador no anticipó. Esta guía cubre formas prácticas de reducir este riesgo.

La buena noticia es que asegurar un despliegue de agentes no requiere infraestructura exótica. Los mismos principios que aplican al ejecutar cualquier código semifiable aplican aquí: aislamiento, mínimo privilegio y defensa en profundidad. Claude Code incluye varias funciones de seguridad que ayudan con las preocupaciones más comunes, y esta guía las recorre junto con opciones adicionales de endurecimiento para quienes las necesiten.

No todos los despliegues necesitan la máxima seguridad. Un desarrollador que ejecuta Claude Code en su laptop tiene requisitos diferentes a los de una empresa que procesa datos de clientes en un entorno multi-tenant. Esta guía presenta opciones que van desde las funciones de seguridad integradas de Claude Code hasta arquitecturas de producción endurecidas, para que puedas elegir lo que se adapta a tu situación.

## Modelo de amenazas

Los agentes pueden tomar acciones no intencionadas debido a prompt injection (instrucciones incrustadas en el contenido que procesan) o a errores del modelo. Los modelos de Claude están diseñados para resistir esto y, como se analiza en la [tarjeta del modelo](https://assets.anthropic.com/m/64823ba7485345a7/Claude-Opus-4-5-System-Card.pdf), Claude Opus 4.6 es el modelo de frontera más robusto disponible.

La defensa en profundidad sigue siendo una buena práctica. Por ejemplo, si un agente procesa un archivo malicioso que le indica que envíe datos de clientes a un servidor externo, los controles de red pueden bloquear esa solicitud por completo.

## Funciones de seguridad integradas

Claude Code incluye varias funciones de seguridad que abordan preocupaciones comunes. Consulta la [documentación de seguridad](https://code.claude.com/docs/en/security) para obtener detalles completos.

- **Sistema de permisos**: Cada herramienta y comando bash puede configurarse para permitir, bloquear o solicitar la aprobación del usuario. Usa patrones glob para crear reglas como "permitir todos los comandos npm" o "bloquear cualquier comando con sudo". Las organizaciones pueden establecer políticas que se apliquen a todos los usuarios. Consulta [control de acceso y permisos](https://code.claude.com/docs/en/iam#access-control-and-permissions).
- **Análisis estático**: Antes de ejecutar comandos bash, Claude Code realiza un análisis estático para identificar operaciones potencialmente riesgosas. Los comandos que modifican archivos del sistema o acceden a directorios sensibles son marcados y requieren aprobación explícita del usuario.
- **Resumen de búsqueda web**: Los resultados de búsqueda se resumen en lugar de pasar el contenido sin procesar directamente al contexto, lo que reduce el riesgo de prompt injection proveniente de contenido web malicioso.
- **Modo sandbox**: Los comandos bash pueden ejecutarse en un entorno sandbox que restringe el acceso al sistema de archivos y a la red. Consulta la [documentación de sandboxing](https://code.claude.com/docs/en/sandboxing) para más detalles.

## Principios de seguridad

Para despliegues que requieren un endurecimiento adicional más allá de los valores predeterminados de Claude Code, estos principios guían las opciones disponibles.

### Límites de seguridad

Un límite de seguridad separa componentes con diferentes niveles de confianza. Para despliegues de alta seguridad, puedes colocar recursos sensibles (como credenciales) fuera del límite que contiene al agente. Si algo sale mal en el entorno del agente, los recursos fuera de ese límite permanecen protegidos.

Por ejemplo, en lugar de dar al agente acceso directo a una clave de API, podrías ejecutar un proxy fuera del entorno del agente que inyecte la clave en las solicitudes. El agente puede hacer llamadas a la API, pero nunca ve la credencial en sí. Este patrón es útil para despliegues multi-tenant o cuando se procesa contenido no confiable.

### Mínimo privilegio

Cuando sea necesario, puedes restringir al agente solo a las capacidades requeridas para su tarea específica:

| Recurso | Opciones de restricción |
|----------|---------------------|
| Sistema de archivos | Montar solo los directorios necesarios, preferir solo lectura |
| Red | Restringir a endpoints específicos mediante proxy |
| Credenciales | Inyectar mediante proxy en lugar de exponer directamente |
| Capacidades del sistema | Eliminar Linux capabilities en contenedores |

### Defensa en profundidad

Para entornos de alta seguridad, aplicar múltiples controles en capas proporciona protección adicional. Las opciones incluyen:

- Aislamiento mediante contenedores
- Restricciones de red
- Controles del sistema de archivos
- Validación de solicitudes en un proxy

La combinación adecuada depende de tu modelo de amenazas y los requisitos operativos.

## Tecnologías de aislamiento

Las diferentes tecnologías de aislamiento ofrecen distintas compensaciones entre fortaleza de seguridad, rendimiento y complejidad operativa.

> **Info:** En todas estas configuraciones, Claude Code (o tu aplicación del Agent SDK) se ejecuta dentro del límite de aislamiento (el sandbox, contenedor o VM). Los controles de seguridad descritos a continuación restringen lo que el agente puede acceder desde dentro de ese límite.

| Tecnología | Fortaleza de aislamiento | Sobrecarga de rendimiento | Complejidad |
|------------|-------------------|---------------------|------------|
| Sandbox runtime | Buena (valores seguros por defecto) | Muy baja | Baja |
| Contenedores (Docker) | Dependiente de la configuración | Baja | Media |
| gVisor | Excelente (con configuración correcta) | Media/Alta | Media |
| VMs (Firecracker, QEMU) | Excelente (con configuración correcta) | Alta | Media/Alta |

### Sandbox runtime

Para aislamiento ligero sin contenedores, [sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime) aplica restricciones de sistema de archivos y red a nivel del sistema operativo.

La principal ventaja es la simplicidad: no se requiere configuración de Docker, imágenes de contenedor ni configuración de red. El proxy y las restricciones del sistema de archivos están integrados. Proporcionas un archivo de configuración que especifica los dominios y rutas permitidos.

**Cómo funciona:**
- **Sistema de archivos**: Usa primitivas del SO (`bubblewrap` en Linux, `sandbox-exec` en macOS) para restringir el acceso de lectura/escritura a rutas configuradas
- **Red**: Elimina el namespace de red (Linux) o usa perfiles Seatbelt (macOS) para enrutar el tráfico de red a través de un proxy integrado
- **Configuración**: Listas de permitidos basadas en JSON para dominios y rutas del sistema de archivos

**Configuración:**
```bash
npm install @anthropic-ai/sandbox-runtime
```

Luego crea un archivo de configuración que especifique las rutas y dominios permitidos.

**Consideraciones de seguridad:**

1. **Kernel del mismo host**: A diferencia de las VMs, los procesos en sandbox comparten el kernel del host. Una vulnerabilidad del kernel podría teóricamente permitir una escapada. Para algunos modelos de amenazas esto es aceptable, pero si necesitas aislamiento a nivel de kernel, usa gVisor o una VM separada.

2. **Sin inspección TLS**: El proxy permite dominios pero no inspecciona el tráfico cifrado. Si el agente tiene credenciales permisivas para un dominio permitido, asegúrate de que no sea posible usar ese dominio para desencadenar otras solicitudes de red o para exfiltrar datos.

Para muchos casos de uso de un solo desarrollador y CI/CD, sandbox-runtime eleva significativamente el nivel de seguridad con una configuración mínima. Las secciones siguientes cubren contenedores y VMs para despliegues que requieren un aislamiento más robusto.

### Contenedores

Los contenedores proporcionan aislamiento mediante namespaces de Linux. Cada contenedor tiene su propia vista del sistema de archivos, árbol de procesos y pila de red, mientras comparte el kernel del host.

Una configuración de contenedor con endurecimiento de seguridad podría verse así:

```bash
docker run \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --security-opt seccomp=/path/to/seccomp-profile.json \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --tmpfs /home/agent:rw,noexec,nosuid,size=500m \
  --network none \
  --memory 2g \
  --cpus 2 \
  --pids-limit 100 \
  --user 1000:1000 \
  -v /path/to/code:/workspace:ro \
  -v /var/run/proxy.sock:/var/run/proxy.sock:ro \
  agent-image
```

Esto es lo que hace cada opción:

| Opción | Propósito |
|--------|---------|
| `--cap-drop ALL` | Elimina Linux capabilities como `NET_ADMIN` y `SYS_ADMIN` que podrían permitir escalada de privilegios |
| `--security-opt no-new-privileges` | Evita que los procesos adquieran privilegios a través de binarios setuid |
| `--security-opt seccomp=...` | Restringe las syscalls disponibles; el valor predeterminado de Docker bloquea ~44, los perfiles personalizados pueden bloquear más |
| `--read-only` | Hace que el sistema de archivos raíz del contenedor sea inmutable, evitando que el agente persista cambios |
| `--tmpfs /tmp:...` | Proporciona un directorio temporal con escritura que se limpia cuando el contenedor se detiene |
| `--network none` | Elimina todas las interfaces de red; el agente se comunica a través del socket Unix montado a continuación |
| `--memory 2g` | Limita el uso de memoria para evitar el agotamiento de recursos |
| `--pids-limit 100` | Limita el conteo de procesos para prevenir fork bombs |
| `--user 1000:1000` | Se ejecuta como usuario no root |
| `-v ...:/workspace:ro` | Monta el código de solo lectura para que el agente pueda analizarlo pero no modificarlo. **Evita montar directorios sensibles del host como `~/.ssh`, `~/.aws` o `~/.config`** |
| `-v .../proxy.sock:...` | Monta un socket Unix conectado a un proxy que se ejecuta fuera del contenedor (ver más abajo) |

**Arquitectura de socket Unix:**

Con `--network none`, el contenedor no tiene interfaces de red en absoluto. La única forma en que el agente puede llegar al mundo exterior es a través del socket Unix montado, que se conecta a un proxy que se ejecuta en el host. Este proxy puede aplicar listas de permitidos de dominios, inyectar credenciales y registrar todo el tráfico.

Esta es la misma arquitectura utilizada por [sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime). Incluso si el agente es comprometido mediante prompt injection, no puede exfiltrar datos a servidores arbitrarios. Solo puede comunicarse a través del proxy, que controla qué dominios son accesibles. Para más detalles, consulta la [publicación del blog sobre sandboxing de Claude Code](https://www.anthropic.com/engineering/claude-code-sandboxing).

**Opciones adicionales de endurecimiento:**

| Opción | Propósito |
|--------|---------|
| `--userns-remap` | Mapea el root del contenedor a un usuario sin privilegios del host; requiere configuración del daemon pero limita el daño en caso de escapada del contenedor |
| `--ipc private` | Aísla la comunicación entre procesos para prevenir ataques entre contenedores |

### gVisor

Los contenedores estándar comparten el kernel del host: cuando el código dentro de un contenedor realiza una llamada al sistema, va directamente al mismo kernel que ejecuta el host. Esto significa que una vulnerabilidad del kernel podría permitir la escapada del contenedor. gVisor aborda esto interceptando las llamadas al sistema en el espacio de usuario antes de que lleguen al kernel del host, implementando su propia capa de compatibilidad que maneja la mayoría de las syscalls sin involucrar el kernel real.

Si un agente ejecuta código malicioso (quizás debido a prompt injection), ese código se ejecuta en el contenedor y podría intentar exploits del kernel. Con gVisor, la superficie de ataque es mucho menor: el código malicioso necesitaría primero explotar la implementación en espacio de usuario de gVisor y tendría acceso limitado al kernel real.

Para usar gVisor con Docker, instala el runtime `runsc` y configura el daemon:

```json
// /etc/docker/daemon.json
{
  "runtimes": {
    "runsc": {
      "path": "/usr/local/bin/runsc"
    }
  }
}
```

Luego ejecuta contenedores con:

```bash
docker run --runtime=runsc agent-image
```

**Consideraciones de rendimiento:**

| Carga de trabajo | Sobrecarga |
|----------|----------|
| Cómputo intensivo en CPU | ~0% (sin intercepción de syscalls) |
| Syscalls simples | ~2× más lento |
| E/S de archivos intensiva | Hasta 10-200× más lento para patrones pesados de open/close |

Para entornos multi-tenant o cuando se procesa contenido no confiable, el aislamiento adicional suele valer la sobrecarga.

### Máquinas virtuales

Las VMs proporcionan aislamiento a nivel de hardware mediante extensiones de virtualización de CPU. Cada VM ejecuta su propio kernel, creando un límite robusto. Una vulnerabilidad en el kernel invitado no compromete directamente al host. Sin embargo, las VMs no son automáticamente "más seguras" que alternativas como gVisor. La seguridad de las VMs depende en gran medida del hipervisor y del código de emulación de dispositivos.

Firecracker está diseñado para el aislamiento ligero de microVMs. Puede arrancar VMs en menos de 125ms con menos de 5 MiB de sobrecarga de memoria, eliminando la emulación de dispositivos innecesaria para reducir la superficie de ataque.

Con este enfoque, la VM del agente no tiene interfaz de red externa. En cambio, se comunica a través de `vsock` (virtual sockets). Todo el tráfico se enruta a través de vsock hacia un proxy en el host, que aplica listas de permitidos e inyecta credenciales antes de reenviar las solicitudes.

### Despliegues en la nube

Para despliegues en la nube, puedes combinar cualquiera de las tecnologías de aislamiento anteriores con controles de red nativos de la nube:

1. Ejecuta los contenedores del agente en una subred privada sin gateway a internet
2. Configura reglas de firewall en la nube (AWS Security Groups, GCP VPC firewall) para bloquear todo el egreso excepto hacia tu proxy
3. Ejecuta un proxy (como [Envoy](https://www.envoyproxy.io/) con su filtro `credential_injector`) que valide solicitudes, aplique listas de permitidos de dominios, inyecte credenciales y reenvíe a APIs externas
4. Asigna permisos IAM mínimos a la cuenta de servicio del agente, enrutando el acceso sensible a través del proxy cuando sea posible
5. Registra todo el tráfico en el proxy con fines de auditoría

## Gestión de credenciales

Los agentes a menudo necesitan credenciales para llamar a APIs, acceder a repositorios o interactuar con servicios en la nube. El desafío es proporcionar este acceso sin exponer las credenciales en sí mismas.

### El patrón de proxy

El enfoque recomendado es ejecutar un proxy fuera del límite de seguridad del agente que inyecte credenciales en las solicitudes salientes. El agente envía solicitudes sin credenciales, el proxy las añade y reenvía la solicitud a su destino.

Este patrón tiene varias ventajas:

1. El agente nunca ve las credenciales reales
2. El proxy puede aplicar una lista de permitidos de endpoints autorizados
3. El proxy puede registrar todas las solicitudes para auditoría
4. Las credenciales se almacenan en un único lugar seguro en lugar de distribuirse a cada agente

### Configurar Claude Code para usar un proxy

Claude Code admite dos métodos para enrutar las solicitudes de muestreo a través de un proxy:

**Opción 1: ANTHROPIC_BASE_URL (simple pero solo para solicitudes de la API de muestreo)**

```bash
export ANTHROPIC_BASE_URL="http://localhost:8080"
```

Esto le indica a Claude Code y al Agent SDK que envíen las solicitudes de muestreo a tu proxy en lugar de directamente a la API de Claude. Tu proxy recibe solicitudes HTTP en texto plano, puede inspeccionarlas y modificarlas (incluyendo inyectar credenciales), y luego las reenvía a la API real.

**Opción 2: HTTP_PROXY / HTTPS_PROXY (para todo el sistema)**

```bash
export HTTP_PROXY="http://localhost:8080"
export HTTPS_PROXY="http://localhost:8080"
```

Claude Code y el Agent SDK respetan estas variables de entorno estándar, enrutando todo el tráfico HTTP a través del proxy. Para HTTPS, el proxy crea un túnel CONNECT cifrado: no puede ver ni modificar el contenido de las solicitudes sin inspección TLS.

### Implementar un proxy

Puedes construir tu propio proxy o usar uno existente:

- [Envoy Proxy](https://www.envoyproxy.io/): proxy de grado de producción con filtro `credential_injector` para añadir cabeceras de autenticación
- [mitmproxy](https://mitmproxy.org/): proxy con terminación TLS para inspeccionar y modificar tráfico HTTPS
- [Squid](http://www.squid-cache.org/): proxy de caché con listas de control de acceso
- [LiteLLM](https://github.com/BerriAI/litellm): gateway para LLMs con inyección de credenciales y limitación de velocidad

### Credenciales para otros servicios

Más allá del muestreo desde la API de Claude, los agentes a menudo necesitan acceso autenticado a otros servicios, como repositorios git, bases de datos y APIs internas. Existen dos enfoques principales:

#### Herramientas personalizadas

Proporciona acceso a través de un servidor MCP o herramienta personalizada que enrute las solicitudes a un servicio que se ejecuta fuera del límite de seguridad del agente. El agente llama a la herramienta, pero la solicitud autenticada real ocurre afuera. Las llamadas a la herramienta van a un proxy que inyecta las credenciales.

Por ejemplo, un servidor MCP de git podría aceptar comandos del agente pero reenviarlos a un proxy git que se ejecuta en el host, el cual añade autenticación antes de contactar el repositorio remoto. El agente nunca ve las credenciales.

Ventajas:
- **Sin inspección TLS**: El servicio externo realiza solicitudes autenticadas directamente
- **Las credenciales permanecen afuera**: El agente solo ve la interfaz de la herramienta, no las credenciales subyacentes

#### Reenvío de tráfico

Para llamadas a la API de Claude, `ANTHROPIC_BASE_URL` te permite enrutar solicitudes a un proxy que puede inspeccionarlas y modificarlas en texto plano. Pero para otros servicios HTTPS (GitHub, registros npm, APIs internas), el tráfico suele estar cifrado de extremo a extremo. Incluso si lo enrutas a través de un proxy mediante `HTTP_PROXY`, el proxy solo ve un túnel TLS opaco y no puede inyectar credenciales.

Para modificar el tráfico HTTPS hacia servicios arbitrarios, sin usar una herramienta personalizada, necesitas un proxy con terminación TLS que descifre el tráfico, lo inspeccione o modifique, y luego lo vuelva a cifrar antes de reenviarlo. Esto requiere:

1. Ejecutar el proxy fuera del contenedor del agente
2. Instalar el certificado CA del proxy en el almacén de confianza del agente (para que el agente confíe en los certificados del proxy)
3. Configurar `HTTP_PROXY`/`HTTPS_PROXY` para enrutar el tráfico a través del proxy

Este enfoque maneja cualquier servicio basado en HTTP sin escribir herramientas personalizadas, pero añade complejidad en la gestión de certificados.

Ten en cuenta que no todos los programas respetan `HTTP_PROXY`/`HTTPS_PROXY`. La mayoría de las herramientas (curl, pip, npm, git) sí lo hacen, pero algunas pueden saltarse estas variables y conectarse directamente. Por ejemplo, `fetch()` de Node.js ignora estas variables por defecto; en Node 24+ puedes establecer `NODE_USE_ENV_PROXY=1` para habilitar el soporte. Para una cobertura completa, puedes usar [proxychains](https://github.com/haad/proxychains) para interceptar llamadas de red, o configurar iptables para redirigir el tráfico saliente a un proxy transparente.

> **Info:** Un **proxy transparente** intercepta el tráfico a nivel de red, por lo que el cliente no necesita estar configurado para usarlo. Los proxies regulares requieren que los clientes se conecten explícitamente y hablen HTTP CONNECT o SOCKS. Los proxies transparentes (como Squid o mitmproxy en modo transparente) pueden manejar conexiones TCP redirigidas sin procesar.

Ambos enfoques aún requieren el proxy con terminación TLS y el certificado CA de confianza. Solo garantizan que el tráfico llegue efectivamente al proxy.

## Configuración del sistema de archivos

Los controles del sistema de archivos determinan qué archivos puede leer y escribir el agente.

### Montaje de código de solo lectura

Cuando el agente necesita analizar código pero no modificarlo, monta el directorio en modo de solo lectura:

```bash
docker run -v /path/to/code:/workspace:ro agent-image
```

> **Advertencia:** Incluso el acceso de solo lectura a un directorio de código puede exponer credenciales. Archivos comunes que debes excluir o limpiar antes de montar:
>
> | Archivo | Riesgo |
> |------|------|
> | `.env`, `.env.local` | Claves de API, contraseñas de bases de datos, secretos |
> | `~/.git-credentials` | Contraseñas/tokens de git en texto plano |
> | `~/.aws/credentials` | Claves de acceso de AWS |
> | `~/.config/gcloud/application_default_credentials.json` | Tokens ADC de Google Cloud |
> | `~/.azure/` | Credenciales de la CLI de Azure |
> | `~/.docker/config.json` | Tokens de autenticación de registros Docker |
> | `~/.kube/config` | Credenciales del clúster de Kubernetes |
> | `.npmrc`, `.pypirc` | Tokens de registros de paquetes |
> | `*-service-account.json` | Claves de cuenta de servicio de GCP |
> | `*.pem`, `*.key` | Claves privadas |
>
> Considera copiar solo los archivos fuente necesarios, o usar filtrado al estilo `.dockerignore`.

### Ubicaciones con escritura

Si el agente necesita escribir archivos, tienes algunas opciones dependiendo de si quieres que los cambios persistan:

Para espacios de trabajo efímeros en contenedores, usa montajes `tmpfs` que existen solo en memoria y se limpian cuando el contenedor se detiene:

```bash
docker run \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --tmpfs /workspace:rw,noexec,size=500m \
  agent-image
```

Si quieres revisar los cambios antes de persistirlos, un sistema de archivos overlay permite al agente escribir sin modificar los archivos subyacentes. Los cambios se almacenan en una capa separada que puedes inspeccionar, aplicar o descartar. Para una salida completamente persistente, monta un volumen dedicado pero mantenlo separado de los directorios sensibles.

## Lecturas adicionales

- [Documentación de seguridad de Claude Code](https://code.claude.com/docs/en/security)
- [Alojamiento del Agent SDK](./Hosting%20the%20Agent%20SDK.md)
- [Gestión de permisos](./Configure%20permissions.md)
- [Sandbox runtime](https://github.com/anthropic-experimental/sandbox-runtime)
- [The Lethal Trifecta for AI Agents](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [gVisor Documentation](https://gvisor.dev/docs/)
- [Firecracker Documentation](https://firecracker-microvm.github.io/)
