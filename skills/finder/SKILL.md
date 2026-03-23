---
name: finder
description: Abre una ventana del Finder en una ruta específica. Úsalo cuando el usuario diga "/finder <ruta>", "abre Finder en <ruta>", "abre el folder X en Finder", o cualquier variante que implique abrir el Finder en un directorio concreto.
---

## Tu tarea

Abrir una nueva ventana del Finder en la ruta que el usuario indique.

## Resolución de la ruta

1. Si el usuario proporciona una ruta absoluta (empieza por `/`), úsala tal cual.
2. Si el usuario proporciona solo un nombre (ej. `services-hub`), búscalo primero en `/Users/kriptos/Repositories/`. Si no existe ahí, avísale.
3. Verifica que la ruta existe antes de abrir Finder. Si no existe, díselo al usuario.

## Cómo abrir Finder

Usa AppleScript para abrir una nueva ventana del Finder en la ruta indicada:

```bash
osascript << 'EOF'
tell application "Finder"
    activate
    open POSIX file "<RUTA_RESUELTA>"
end tell
EOF
```

Reemplaza `<RUTA_RESUELTA>` con la ruta absoluta resultante.

## Respuesta al usuario

Tras ejecutar el script, confirma brevemente qué ventana se abrió, por ejemplo:
> ✅ Finder abierto en `/Users/kriptos/Repositories/services-hub`

Si hay algún error (ruta no encontrada, etc.), explícalo con claridad y sugiere una solución.
