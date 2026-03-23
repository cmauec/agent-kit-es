---
name: create-project
description: Crea un nuevo proyecto con la estructura de carpetas estándar y descarga los settings de VSCode desde GitHub. Úsalo cuando el usuario escriba /create-project, quiera crear un nuevo proyecto, o mencione que necesita inicializar un proyecto con configuración de VSCode. Acepta un nombre obligatorio y opcionalmente una URL de un repo privado de GitHub para clonarlo dentro del proyecto.
---

# create-project

Extrae el `<nombre>` y el `[repo_url]` del mensaje del usuario, luego sigue la lógica de abajo.

## Lógica principal

### Si el usuario SÍ pasó `repo_url`

Ejecuta directamente:

```bash
NAME="<nombre>"
REPO_URL="<repo_url>"
PROJECT_FOLDER="${NAME}-project"
INNER_FOLDER="${PROJECT_FOLDER}/${NAME}"
WORKSPACE_URL="https://raw.githubusercontent.com/cmauec/settings-vsc/main/settings-vsc.code-workspace"

mkdir -p "${PROJECT_FOLDER}"
echo "Clonando repo ${REPO_URL}..."
gh repo clone "$REPO_URL" "$INNER_FOLDER"

echo "Descargando settings desde GitHub..."
curl -k -s "$WORKSPACE_URL" -o "${PROJECT_FOLDER}/${NAME}.code-workspace"

echo ""
echo "Proyecto creado:"
echo "  ${PROJECT_FOLDER}/"
echo "  ├── ${NAME}.code-workspace"
echo "  └── ${NAME}/"
```

---

### Si el usuario NO pasó `repo_url`

Sigue este flujo interactivo paso a paso:

**Paso 1 — Preguntar si desea crear un repo en GitHub**

Pregunta al usuario:
> ¿Deseas crear un repositorio de GitHub para este proyecto? (sí/no)

---

**Paso 2a — Si responde NO**

Crea el proyecto localmente sin repo:

```bash
NAME="<nombre>"
PROJECT_FOLDER="${NAME}-project"
INNER_FOLDER="${PROJECT_FOLDER}/${NAME}"
WORKSPACE_URL="https://raw.githubusercontent.com/cmauec/settings-vsc/main/settings-vsc.code-workspace"

mkdir -p "${INNER_FOLDER}/.vscode"
curl -k -s "$WORKSPACE_URL" -o "${PROJECT_FOLDER}/${NAME}.code-workspace"

echo "Proyecto creado:"
echo "  ${PROJECT_FOLDER}/"
echo "  ├── ${NAME}.code-workspace"
echo "  └── ${NAME}/"
```

---

**Paso 2b — Si responde SÍ**

Busca repos similares en GitHub por nombre usando `gh`:

```bash
gh repo list --limit 100 --json name,url,description \
  | jq -r '.[] | select(.name | test("<nombre>"; "i")) | "\(.name) — \(.description // "sin descripción") → \(.url)"'
```

Si hay coincidencias, preséntale la lista numerada al usuario:

> Encontré los siguientes repositorios similares en tu cuenta de GitHub:
>
> 1. `nombre-repo` — descripción → URL
> 2. `otro-repo` — descripción → URL
> …
>
> ¿Quieres usar alguno de estos? Escribe el número, o "no" para crear uno nuevo.

---

**Paso 3a — Si el usuario escoge un repo de la lista**

Clona ese repo y crea el proyecto:

```bash
NAME="<nombre>"
REPO_URL="<url_del_repo_elegido>"
PROJECT_FOLDER="${NAME}-project"
INNER_FOLDER="${PROJECT_FOLDER}/${NAME}"
WORKSPACE_URL="https://raw.githubusercontent.com/cmauec/settings-vsc/main/settings-vsc.code-workspace"

mkdir -p "${PROJECT_FOLDER}"
gh repo clone "$REPO_URL" "$INNER_FOLDER"
curl -k -s "$WORKSPACE_URL" -o "${PROJECT_FOLDER}/${NAME}.code-workspace"

echo "Proyecto creado con repo existente:"
echo "  ${PROJECT_FOLDER}/"
echo "  ├── ${NAME}.code-workspace"
echo "  └── ${NAME}/"
```

---

**Paso 3b — Si el usuario dice "no" (crear repo nuevo)**

Crea el repo en GitHub y el proyecto local:

```bash
NAME="<nombre>"
PROJECT_FOLDER="${NAME}-project"
INNER_FOLDER="${PROJECT_FOLDER}/${NAME}"
WORKSPACE_URL="https://raw.githubusercontent.com/cmauec/settings-vsc/main/settings-vsc.code-workspace"

mkdir -p "${PROJECT_FOLDER}"
gh repo create "$NAME" --private --clone --local-dir "$INNER_FOLDER"
curl -k -s "$WORKSPACE_URL" -o "${PROJECT_FOLDER}/${NAME}.code-workspace"

echo "Proyecto creado con nuevo repo GitHub:"
echo "  ${PROJECT_FOLDER}/"
echo "  ├── ${NAME}.code-workspace"
echo "  └── ${NAME}/"
```

---

## Ejemplos

Sin repo (flujo interactivo):
```
/create-project agents
```

Con repo privado existente (flujo directo):
```
/create-project agents https://github.com/cmauec/agents
```
