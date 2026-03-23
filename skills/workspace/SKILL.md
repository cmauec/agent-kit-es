---
name: workspace
description: Inicia el entorno de desarrollo completo para un proyecto: abre iTerm2 y Visual Studio Code apuntando al proyecto indicado. Úsalo cuando el usuario diga "inicia el entorno de X", "abre el entorno de trabajo de X", "prepara el workspace de X", "setup de X", "empieza a trabajar en X", o cualquier variante que implique iniciar el entorno de un proyecto concreto.
---

## Tu tarea

Iniciar el entorno de desarrollo completo para el proyecto que el usuario indique, abriendo simultáneamente iTerm2, Visual Studio Code y el Finder.

## Paso 1: Resolver la ruta del proyecto

1. El usuario proporcionará un nombre de proyecto.
2. Lista los folders en el directorio de trabajo actual (obtenido con `pwd`) y encuentra el que más se parezca al nombre dado (coincidencia parcial, case-insensitive).
3. Si no encuentras ninguno similar, díselo al usuario y muéstrale los proyectos disponibles.
4. Si la ruta resuelta no existe, avísale.

## Paso 2: Abrir iTerm2

Usa AppleScript para abrir una nueva ventana en la instancia existente de iTerm:

```bash
osascript << 'EOF'
tell application "iTerm"
    activate
    set newWindow to (create window with default profile)
    tell newWindow
        tell current session
            set name to "<NOMBRE_FOLDER>"
            write text "cd '<RUTA_RESUELTA>' && clear"
        end tell
    end tell
end tell
EOF
```

Reemplaza `<RUTA_RESUELTA>` con la ruta absoluta del proyecto y `<NOMBRE_FOLDER>` con el último componente de esa ruta.

## Paso 3: Abrir Visual Studio Code

1. Busca si existe algún archivo `.code-workspace` dentro del folder del proyecto.
2. **Si existe**, ábrelo con:
   ```bash
   open -a "Visual Studio Code" <ruta-completa>/<archivo>.code-workspace
   ```
3. **Si no existe**, abre VS Code con el folder del proyecto:
   ```bash
   open -a "Visual Studio Code" <RUTA_RESUELTA>
   ```

## Paso 4: Abrir Finder

Usa AppleScript para abrir una ventana del Finder en la ruta del proyecto:

```bash
osascript << 'EOF'
tell application "Finder"
    activate
    open POSIX file "<RUTA_RESUELTA>"
end tell
EOF
```

## Respuesta al usuario

Confirma brevemente lo que se abrió, por ejemplo:

> iTerm2 abierto en `<RUTA_RESUELTA>`
> VS Code abierto con workspace `<NOMBRE_WORKSPACE>`
> Finder abierto en `<RUTA_RESUELTA>`

Si algo falla (proyecto no encontrado, app no instalada), explícalo con claridad.
