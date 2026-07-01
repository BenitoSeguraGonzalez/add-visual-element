# Preview — Catálogo persistente de componentes

Módulo de gestión del catálogo HTML persistente de NurUI (o librería equivalente).
Los previews efímeros por componente no existen — cada adición actualiza el catálogo completo.

## Arquitectura del catálogo (3 optimizaciones aplicadas)

```
<scope>/
├── catalog.html              ← abre con file:// directo, sin servidor
└── previews/
    ├── catalog.react.js      ← React + ReactDOM + jsx-runtime (~190KB) — NUNCA REBUILD
    └── catalog.app.js        ← componentes + UI del catálogo (~29KB) — rebuild por cambio
```

`catalog.html` carga los dos scripts en orden:
```html
<script src="previews/catalog.react.js"></script>   <!-- built once -->
<script src="previews/catalog.app.js"></script>      <!-- rebuilt per change -->
```

### Por qué dos bundles (Optimización 1)

| Bundle | Tamaño | Cuándo se recompila |
|--------|--------|---------------------|
| `catalog.react.js` | ~190KB | Solo si cambia la versión de React |
| `catalog.app.js` | ~29KB base | En cada adición/modificación de componente |

Añadir un componente recompila solo `catalog.app.js` (29KB + tamaño del nuevo componente).
Sin la separación, cada adición recompila ~218KB incluyendo React.

---

## Optimización 2: Entry file incremental con marcadores

`catalog.app.entry.tsx` tiene tres bloques con marcadores para append incremental.
Al añadir un componente: NO se reescribe el archivo completo — solo se insertan líneas dentro de cada bloque.

```tsx
// @nur-ui:start imports
import { BookCard } from '../nur-ui/BookCard';
// → nuevo: import { NuevoComponente } from '../nur-ui/NuevoComponente';
// @nur-ui:end imports

// @nur-ui:start previews
function BookCardPreview({ theme }) { ... }
// → nuevo: function NuevoComponentePreview({ theme }) { ... }
// @nur-ui:end previews

// @nur-ui:start registry
const COMPONENTS: ComponentEntry[] = [
  { name: 'BookCard', ... },
  // → nuevo: { name: 'NuevoComponente', ... },
];
// @nur-ui:end registry
```

**Regla**: siempre insertar antes de `// @nur-ui:end <bloque>`, nunca reemplazar el archivo entero.

---

## Optimización 3: Skip por hash

Antes de recompilar, verificar si el componente ya está en `manifest.json` con el mismo hash.

```bash
# Obtener hash del archivo del componente
COMPONENT_HASH=$(sha256sum <LIB_DIR>/<Nombre>.tsx | cut -d' ' -f1)
# Leer hash guardado en manifest.json para este componente
SAVED_HASH=$(jq -r '.components["<Nombre>"].hash // empty' manifest.json 2>/dev/null)

if [ "$COMPONENT_HASH" = "$SAVED_HASH" ]; then
  echo "⟳ <Nombre> sin cambios — skip"
else
  # Compilar y actualizar manifest
  ...
  jq '.components["<Nombre>"].hash = "'$COMPONENT_HASH'"' manifest.json > manifest.tmp && mv manifest.tmp manifest.json
fi
```

Aplica en `--rebuild` sobre una librería sin cambios reales: skip total por componente que no cambió.

---

## Flujo de actualización (Paso 3.5 del pipeline)

Se ejecuta al añadir un componente nuevo, después de crear el `.tsx`.

### 1. Hash check (Optimización 3)

```bash
COMPONENT_HASH=$(sha256sum <LIB_DIR>/<Nombre>.tsx | cut -d' ' -f1)
SAVED_HASH=$(jq -r '.components["<Nombre>"].hash // empty' manifest.json 2>/dev/null)
```

Si coinciden → skip compilación, el componente ya está en la app entry. Continuar al Paso 4.

### 2. Insertar en catalog.app.entry.tsx (Optimización 2)

Con marcadores `@nur-ui:start/end`:

**a) Import** — insertar antes de `// @nur-ui:end imports`:
```tsx
import { <Nombre> } from '../nur-ui/<Nombre>';
```

**b) Preview function** — insertar antes de `// @nur-ui:end previews`:
```tsx
function <Nombre>Preview({ theme }: { theme: Theme }) {
  const bg = theme === 'dark' ? '#0f172a' : '#f8fafc';
  return (
    <div style={{ padding: 24, background: bg, display: 'flex', gap: 12, flexWrap: 'wrap', justifyContent: 'center', alignItems: 'center', minHeight: 200 }}>
      {/* Renderizar variantes del componente según su definición */}
      <<Nombre> variant="<variante-1>" ... />
      <<Nombre> variant="<variante-2>" ... />
    </div>
  );
}
```

**c) Registry entry** — insertar antes de `// @nur-ui:end registry`, marcando `isNew: true` SOLO en este componente y `isNew: false` en los demás:
```tsx
{
  name: '<Nombre>',
  category: '<categoría>',
  description: '<descripción de 1-2 líneas>',
  variants: ['<v1>', '<v2>'],
  isNew: true,
  Preview: <Nombre>Preview,
},
```

Cambiar `isNew: true` → `isNew: false` en el componente que tenía el badge anterior.

### 3. Rebuild de catalog.app.js

```bash
# Desde el root del frontend (donde está package.json y node_modules)
npx esbuild <previews-dir>/catalog.app.entry.tsx \
  --bundle --minify --format=iife \
  --jsx=automatic \
  --define:process.env.NODE_ENV='"production"' \
  --external:react \
  --external:react/jsx-runtime \
  --external:react-dom/client \
  '--banner:js=var require=function(m){return{"react":window.__nurui_React,"react/jsx-runtime":window.__nurui_JSXRuntime,"react-dom/client":window.__nurui_ReactDOM}[m]||{}};' \
  --outfile=<previews-dir>/catalog.app.js
```

**NO reconstruir `catalog.react.js`** salvo que React haya cambiado de versión.

### 4. Backup atómico del catálogo actual

```bash
cp <catalog.html> <catalog.html>.bak
```

Si no existe → no hacer backup (primer componente).

### 5. Abrir catálogo sin servidor

```bash
# Windows
start <ruta-absoluta-a-catalog.html>

# macOS/Linux
open <ruta-absoluta-a-catalog.html>
```

El catálogo funciona directo con `file://`. No se necesita servidor HTTP.

### 6. Confirmación

| Decisión | Acción |
|----------|--------|
| **Confirma** | `rm <catalog.html>.bak` → actualizar hash en manifest.json → continuar al Paso 4 del pipeline |
| **Rechaza** | `mv <catalog.html>.bak <catalog.html>` → revertir inserción en catalog.app.entry.tsx → el componente NO se indexa |

---

## Comando `--view`

```
Usuario: "ver componentes" / "muéstrame la librería" / "--view"
```

1. Determinar scope (global o local)
2. Verificar que existe `catalog.html`
   - Si no existe → auto-trigger `--rebuild` para ese scope, luego continuar
3. Si existe → abrir directo (sin servidor):
   ```bash
   # Windows
   start <ruta-absoluta-a-catalog.html>
   # macOS/Linux
   open <ruta-absoluta-a-catalog.html>
   ```

**No recompilar, no regenerar** si el catálogo ya existe.

---

## Comando `--rebuild`

```
Usuario: "rehace el catálogo" / "regenera la librería" / "--rebuild"
```

Recompila todos los componentes y regenera el catálogo desde cero.
No usa backup ni pide confirmación — el usuario lo solicitó explícitamente.

### Flujo

1. **Determinar scope**: `--scope global` → vault global. `--scope project` (default) → proyecto activo.

2. **Leer fuente de componentes**:
   - Leer `manifest.json` del scope → lista de componentes con metadata
   - Si no existe → escanear `<LIB_DIR>/` buscando archivos `*.tsx` con header `// @nur-ui:` o exports nombrados

3. **Si no hay componentes**:
   - Crear `catalog.html` vacío con el template base
   - Crear `previews/` si no existe
   - Inicializar `catalog.app.entry.tsx` con el template vacío (solo preamble + markers vacíos)
   - Fin.

4. **Verificar catalog.react.js**:
   - Si no existe → compilarlo primero (ver `## Compilar catalog.react.js`)
   - Si existe → reutilizar (nunca forzar rebuild de React sin cambio de versión)

5. **Reconstruir catalog.app.entry.tsx completo**:
   - Para cada componente: aplicar hash check (Optimización 3) — skip si no cambió
   - Reescribir el entry file completo con TODOS los componentes actualizados
   - Marcar `isNew: true` solo al componente con fecha `updated` más reciente en manifest.json

6. **Compilar catalog.app.js** (mismo comando que el Paso 3 del flujo de adición)

7. **Abrir catálogo**:
   ```bash
   start <ruta-absoluta-a-catalog.html>  # Windows
   open <ruta-absoluta-a-catalog.html>   # macOS/Linux
   ```

8. **Reportar**:
   ```
   Catálogo regenerado.
   Scope: [global | proyecto]
   Componentes: N (M skipped — sin cambios)
   Último añadido: <Nombre>
   React bundle: reutilizado (sin rebuild)
   Catálogo: <ruta>
   ```

---

## Compilar catalog.react.js

Solo ejecutar si no existe o si React cambió de versión.

**1. Crear entry file** `previews/catalog.react.entry.ts`:
```typescript
import * as React from 'react';
import * as ReactDOM from 'react-dom/client';
import * as JSXRuntime from 'react/jsx-runtime';

(window as any).__nurui_React = React;
(window as any).__nurui_ReactDOM = ReactDOM;
(window as any).__nurui_JSXRuntime = JSXRuntime;
```

**2. Compilar**:
```bash
npx esbuild <previews-dir>/catalog.react.entry.ts \
  --bundle --minify --format=iife \
  --define:process.env.NODE_ENV='"production"' \
  --outfile=<previews-dir>/catalog.react.js
```

---

## Template base de catalog.html

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{LIB_NAME}} — Component Catalog</title>
  <meta name="author" content="{{AUTHOR_NAME}}">
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body, #root { height: 100%; }
    body { overflow-x: hidden; }
    input[type="search"]::-webkit-search-cancel-button { display: none; }
    input:focus { outline: 2px solid #6366f1; outline-offset: 1px; }
    button:focus-visible { outline: 2px solid #6366f1; outline-offset: 2px; }
    ::-webkit-scrollbar { width: 6px; height: 6px; }
    ::-webkit-scrollbar-track { background: transparent; }
    ::-webkit-scrollbar-thumb { background: #334155; border-radius: 3px; }
  </style>
</head>
<body>
  <div id="root"></div>
  <!-- catalog.react.js: React runtime — built once, never rebuilt -->
  <script src="previews/catalog.react.js"></script>
  <!-- catalog.app.js: components (~29KB base) — rebuilt on each change -->
  <script src="previews/catalog.app.js"></script>
</body>
</html>
```

`{{AUTHOR_NAME}}` = nombre del autor configurado (Ing. Benito Segura por defecto en este proyecto).

---

## Template base de catalog.app.entry.tsx

```tsx
// @nur-ui:catalog-app v1 — React is provided by catalog.react.js (external)
import { useState, useMemo, useCallback } from 'react';
import { createRoot } from 'react-dom/client';

// @nur-ui:start imports
// @nur-ui:end imports

type Theme = 'light' | 'dark';

interface ComponentEntry {
  name: string;
  category: string;
  description: string;
  variants: string[];
  isNew: boolean;
  Preview: React.FC<{ theme: Theme }>;
}

// @nur-ui:start previews
// @nur-ui:end previews

// @nur-ui:start registry
const COMPONENTS: ComponentEntry[] = [];
// @nur-ui:end registry

const CATEGORIES = ['all', ...Array.from(new Set(COMPONENTS.map(c => c.category))).sort()];

// ... (tokens T, Catalog function, createRoot mount — ver archivo real del proyecto)
```

El cuerpo de `Catalog` (tokens, render) no cambia al añadir componentes — solo los bloques con marcadores.

---

## Firma del catálogo

El catálogo debe incluir la firma del autor de forma discreta:

1. **HTML**: `<meta name="author" content="Ing. Benito Segura">`
2. **Sidebar** (en el React app): texto en la parte inferior del nav, color muy sutil:
   ```tsx
   <div style={{ marginTop: 'auto', paddingTop: '1.5rem', fontSize: '0.6rem', color: '#1e3a5f', textAlign: 'center', userSelect: 'none' }}>
     Ing. Benito Segura
   </div>
   ```

---

## Auto-creación (self-healing)

Cualquier operación que necesite `catalog.html` debe verificar su existencia.
Si no existe → lanzar `--rebuild` automáticamente para ese scope antes de continuar.

Aplica a: `--view`, Paso 3.5 (cuando intenta usar el catálogo), `--check`.

---

## Cleanup automático

Al invocar el skill, eliminar backups viejos:

```bash
# Backups con más de 1 hora
find ~/.claude/nur-ui/ -name "catalog.html.bak" -mmin +60 -delete 2>/dev/null
find "$(dirname <LIB_DIR>)" -name "catalog.html.bak" -mmin +60 -delete 2>/dev/null
```

---

## Dependencias

- `esbuild` → `npx esbuild`
- `node` → para hash check y jq alternative con `node -e`
- **No requiere** python, tailwindcss, ni servidor HTTP
