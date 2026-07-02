# Catalog — Barrel index, design.md y manifest.json

Módulo de indexación y documentación. Mantiene sincronizados el barrel file, design.md, y manifest.json.

## Template de componente React

Todo componente nuevo sigue esta estructura:

```tsx
// @nur-ui: source=project created=YYYY-MM-DD category=<categoría>

import { type ReactNode } from 'react';

/* ── Variant tokens ────────────────────────────────────────────────
   <descripción de origen si es de terceros>
   <atribución si aplica: "Based on UIverse <nombre> (by <autor>).">

   Variants:
   - <variant1>: <descripción>
   - <variant2>: <descripción>
   ─────────────────────────────────────────────────────────────── */

// ── Types ──────────────────────────────────────────────────────

type Variant = '<variant1>' | '<variant2>';

interface VariantTokens {
  // tokens específicos de cada variante
}

const tokens: Record<Variant, VariantTokens> = {
  // ...
};

// ── Component props ────────────────────────────────────────────

export interface NombreProps {
  children: ReactNode;
  variant?: Variant;
  className?: string;
  // ... otras props
}

// ── Component ──────────────────────────────────────────────────

/**
 * <descripción breve del componente y su propósito>
 *
 * @example
 * import { Nombre } from '<LIB_IMPORT>';
 *
 * <Nombre variant="<default>">
 *   <p>contenido</p>
 * </Nombre>
 */
export function Nombre({ children, variant = '<default>', className = '' }: NombreProps) {
  const t = tokens[variant];

  return (
    <div className={className} style={{ /* estilos dinámicos */ }}>
      {children}
    </div>
  );
}
```

**Reglas estrictas:**
- **Sin tamaño fijo**: nunca `width` o `height` fijos en el componente base. Se adapta al contenedor.
- **Estilos dinámicos**: los tokens de variante van en `style={{}}` inline, no en clases Tailwind condicionales.
- **Interface exportada**: `NombreProps` siempre exportado.
- **Atribución**: si el diseño viene de UIverse u otra fuente, mencionarlo en el comentario superior.
- **Sin default export**: solo named exports.

## Barrel index (`index.ts`)

### Agregar componente

Insertar en orden alfabético entre los exports existentes:

```ts
export { DragonflyCard } from './DragonflyCard';
export type { DragonflyCardProps } from './DragonflyCard';

export { KpiCard } from './KpiCard';          // ← nuevo, ordenado
export type { KpiCardProps } from './KpiCard'; // ← nuevo, ordenado

export { StatusBadge } from './StatusBadge';
export type { StatusBadgeProps } from './StatusBadge';
```

**Regla**: `export { Componente }` primero, `export type { ComponenteProps }` después. Ambos en orden alfabético.

### Eliminar componente

Quitar ambas líneas (componente + tipo). No dejar gaps.

## Fuente de verdad del catálogo

`docs/design/design.md` es la única fuente de verdad del catálogo NurUI. Los archivos que se actualizan al agregar un componente son exactamente estos tres:

1. `docs/design/design.md` — catálogo + enforcement registry
2. `<LIB_DIR>/index.ts` — barrel
3. `~/.claude/nur-ui/vault.json` — vault global (solo si scope=global o ambos)

`CLAUDE.md` es pasarela hacia `design.md` y no se toca. Si `CLAUDE.md` tiene una tabla NurUI, es una referencia mínima de ejemplo — no se modifica. Lo mismo aplica a `README.md` u otros `.md` del proyecto.

## design.md — Marcadores idempotentes

### Inicialización (si no existen)

Insertar al final de `design.md`:

```markdown
<!-- @nur-ui:start catalog -->
## Catálogo de componentes NurUI

| Componente | Categoría | Variantes | Archivo |
|-----------|-----------|-----------|---------|

<!-- @nur-ui:end catalog -->

<!-- @nur-ui:start enforcement -->
## Enforcement Registry

| UI Type | Componente | Regla |
|---------|-----------|-------|

<!-- @nur-ui:end enforcement -->
```

### Actualizar catálogo

Siempre editar **entre** los marcadores `<!-- @nur-ui:start catalog -->` y `<!-- @nur-ui:end catalog -->`.
Nunca tocar contenido fuera de estos marcadores.

Para agregar un componente:
1. Leer el bloque delimitado por los marcadores
2. Agregar fila a la tabla en orden alfabético
3. Reemplazar el bloque entero

```markdown
<!-- @nur-ui:start catalog -->
## Catálogo de componentes NurUI

| Componente | Categoría | Variantes | Archivo |
|-----------|-----------|-----------|---------|
| DragonflyCard | paneles | light, dark, amber | `frontend/src/nur-ui/DragonflyCard.tsx` |
| KpiCard | data-display | default, compact, hero | `frontend/src/nur-ui/KpiCard.tsx` |
<!-- @nur-ui:end catalog -->
```

### Actualizar enforcement

Mismo mecanismo: editar entre `<!-- @nur-ui:start enforcement -->` y `<!-- @nur-ui:end enforcement -->`.

```markdown
<!-- @nur-ui:start enforcement -->
## Enforcement Registry

| UI Type | Componente | Regla |
|---------|-----------|-------|
| KPI Card | KpiCard | Usar KpiCard de NurUI. No crear tarjetas KPI manualmente. |
| Panel / Card Container | DragonflyCard | Usar DragonflyCard para cualquier contenedor tipo card con sombra. |
<!-- @nur-ui:end enforcement -->
```

## component-map.json — LEY DE OBLIGADO CUMPLIMIENTO

> **NUNCA ignorar. NUNCA saltar. Aplica en toda operación del skill sin excepción.**

`component-map.json` es el mapa de uso exacto de cada componente NurUI en el proyecto.
Vive en `<LIB_DIR>/component-map.json`. Se genera al crear la librería y se actualiza en TODA operación posterior.

### Cuándo actualizar (SIEMPRE)

| Operación | Acción sobre component-map.json |
|-----------|--------------------------------|
| Crear componente nuevo | Agregar entrada con `usages: []` |
| Sustituir elemento existente por componente NurUI | Agregar cada instancia reemplazada a `usages` del componente |
| Eliminar componente | Eliminar su entrada completa |
| Renombrar componente | Renombrar su clave |
| Detectar nuevo consumidor al correr `--check` | Agregar a `usages` del componente correspondiente |

### Estructura

```json
{
  "generated": "ISO-8601",
  "library": "NurUI",
  "lib_dir": "frontend/src/nur-ui/",
  "lib_import": "@/nur-ui",
  "_note": "Auto-generado por add-visual-element. NUNCA editar manualmente.",
  "components": {
    "DragonflyCard": {
      "usages": [
        {
          "file": "modules/superadmin/components/SuperAdminKpiCard.tsx",
          "line": 28,
          "context": "descripción breve del uso en ese punto"
        }
      ]
    }
  }
}
```

### Reglas de contenido

- `file`: ruta relativa desde `frontend/src/`
- `line`: línea del JSX `<ComponenteName` (no el import, la instancia)
- `context`: 1 frase que describe qué hace el componente en ese punto
- Si el componente no tiene usages aún → `"usages": []` (nunca omitir la entrada)
- `generated` se actualiza con cada escritura

### Cómo detectar usages

```bash
# Encontrar todos los archivos que importan desde @/nur-ui
rg "from '@/nur-ui'" --glob "*.tsx" --glob "*.ts" -l

# Para cada archivo, encontrar línea de JSX de cada componente
rg "<NombreComponente[\s>]" --glob "*.tsx" -n
```

Tras una sustitución masiva: escanear todos los archivos modificados y actualizar `usages` con archivo + línea + contexto de cada instancia nueva.

---

## manifest.json

Auto-generado después de cada operación del skill. Vive en `<LIB_DIR>/manifest.json`.

```json
{
  "library": "NurUI",
  "version": "1.0.0",
  "generated": "2026-07-01T15:30:00Z",
  "components": {
    "DragonflyCard": {
      "file": "DragonflyCard.tsx",
      "category": "paneles",
      "variants": ["light", "dark", "amber"],
      "props": ["children", "variant", "className"],
      "dependencies": [],
      "source": "global",
      "hash": "a1b2c3d4e5f6",
      "consumers": [
        "modules/superadmin/components/SuperAdminKpiCard.tsx"
      ]
    }
  },
  "categories": {
    "paneles": ["DragonflyCard"],
    "data-display": ["KpiCard"]
  },
  "enforcement": [
    { "uiType": "KPI Card", "component": "KpiCard", "rule": "Usar KpiCard de NurUI." },
    { "uiType": "Panel / Card Container", "component": "DragonflyCard", "rule": "Usar DragonflyCard para cards con sombra." }
  ]
}
```

### Regeneración

Se regenera completo (no incremental) después de:
- Agregar componente
- Eliminar componente
- Renombrar componente
- Cambiar enforcement registry
- `--check` (se actualizan hashes y consumers)

### Consumidores

Para detectar consumidores: `rg "from '<LIB_IMPORT>'" --glob "*.tsx" --glob "*.ts"` y parsear qué componentes se importan.

## Migración de design.md existente

Si el proyecto ya tiene `design.md` sin marcadores NurUI:

1. **Backup**: `cp docs/design/design.md docs/design/design.md.bak-YYYYMMDD`
2. **Insertar marcadores** al final del archivo (sin tocar contenido existente):
   ```
   ---
   (contenido existente intacto)
   ---
   <!-- @nur-ui:start catalog -->...<!-- @nur-ui:end catalog -->
   <!-- @nur-ui:start enforcement -->...<!-- @nur-ui:end enforcement -->
   ```
3. **Sembrar catálogo inicial**: escanear `<LIB_DIR>/` y poblar la tabla de catálogo con componentes existentes
4. **Reportar**: "N componentes detectados. 0 enforcement rules. El contenido existente de design.md no se modificó."
5. **Usuario valida**: si rechaza, restaurar backup.
