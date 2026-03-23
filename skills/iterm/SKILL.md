---
name: iterm
description: Abre una nueva ventana de iTerm en una ruta específica. Úsalo cuando el usuario diga "/iterm <ruta>", "abre iTerm en <ruta>", "abre el proyecto X en iTerm", o cualquier variante que implique abrir iTerm en un directorio concreto.
---

## Tu tarea

Abrir una nueva ventana de iTerm en la ruta que el usuario indique.

## Resolución de la ruta

1. Si el usuario proporciona una ruta absoluta (empieza por `/`), úsala tal cual.
2. Si el usuario proporciona solo un nombre, búscalo primero en el directorio de trabajo actual (obtenido con `pwd`). Si no existe ahí, avísale.
3. Verifica que la ruta existe antes de abrir iTerm. Si no existe, díselo al usuario.

## Cómo abrir iTerm

Usa AppleScript para abrir una nueva ventana en la instancia existente (sin duplicar icono en Dock), estableciendo como título el nombre del folder (última parte de la ruta):

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

Reemplaza `<RUTA_RESUELTA>` con la ruta absoluta resultante y `<NOMBRE_FOLDER>` con el último componente de esa ruta.

## Respuesta al usuario

Tras ejecutar el script, confirma brevemente qué ventana se abrió, por ejemplo:
> ✅ Nueva ventana de iTerm abierta en `<RUTA_RESUELTA>`

Si hay algún error (ruta no encontrada, iTerm no instalado, etc.), explícalo con claridad y sugiere una solución.
