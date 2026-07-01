# Vault — Gestión de componentes globales vs locales

Módulo de gestión del vault global (`~/.claude/nur-ui/`) y su relación con proyectos locales.

## Principio fundamental

> Un componente global NUNCA se usa directamente. Siempre se copia al proyecto local.

El vault global es un **template repository**. Los proyectos locales tienen su propia copia independiente. Esto evita acoplamiento entre proyectos y permite adaptaciones locales.

## Estructura del vault

```
~/.claude/nur-ui/
├── vault.json              ← índice de componentes globales
├── categories.json         ← categorías globales y sus componentes
├── catalog.html            ← catálogo persistente renderizado (global)
├── components/
│   ├── DragonflyCard.tsx
│   ├── KpiCard.tsx
│   └── ...
└── previews/               ← preview HTML compilado por componente (srcdoc del catálogo)
    ├── DragonflyCard.html
    ├── KpiCard.html
    └── ...
```

El catálogo local del proyecto vive en `<LIB_DIR>/../catalog.html` y `<LIB_DIR>/../previews/`.
Ejemplo: si `LIB_DIR` es `frontend/src/nur-ui/`, el catálogo es `frontend/src/catalog.html`.

## vault.json

```json
{
  "version": "1.0.0",
  "components": {
    "DragonflyCard": {
      "hash": "a1b2c3d4e5f6",
      "created": "2026-07-01",
      "updated": "2026-07-01",
      "category": "paneles",
      "variants": ["light", "dark", "amber"],
      "dependencies": [],
      "file": "components/DragonflyCard.tsx"
    }
  }
}
```

## Flujo: crear componente global

```
Usuario: "creá un componente global KpiCard"
         → o: Paso 2 responde --scope global
```

1. Crear componente en `~/.claude/nur-ui/components/Nombre.tsx`
2. Calcular hash (SHA256 de los primeros 1024 bytes del archivo)
3. Registrar en `vault.json`
4. Actualizar `categories.json`
5. Ejecutar Paso 3.5 del pipeline sobre el scope global → actualizar `~/.claude/nur-ui/catalog.html` (→ `preview.md`)
6. Reportar: "Componente `Nombre` creado en vault global. Usá `--scope project` o `--check` para copiarlo a proyectos locales."

## Flujo: usar componente global en proyecto

```
Usuario: "usá DragonflyCard en este proyecto"
        → o: el enforcement registry sugiere un global
```

1. Verificar que existe en vault (`vault.json` + hash)
2. Verificar si ya existe copia local en `<LIB_DIR>/`
   - Si existe: comparar hashes. Si difieren → preguntar cuál versión usar
   - Si no existe: copiar del vault al proyecto
3. Copiar archivo a `<LIB_DIR>/Nombre.tsx`
4. Insertar metadata embebida (ver abajo)
5. Indexar en barrel (→ `catalog.md`)
6. Actualizar enforcement registry si aplica (→ `enforce.md`)

## Metadata embebida

Todo componente copiado del vault recibe este header:

```tsx
// @nur-ui: source=global hash=a1b2c3d4e5f6 copied=2026-07-01 category-global=paneles category-local=contenedores
```

Campos:
- `source`: `global` | `project`
- `hash`: hash del archivo fuente en el vault (para `--check`)
- `copied`: fecha de copia ISO
- `category-global`: categoría en el vault
- `category-local`: categoría asignada en este proyecto (puede diferir)

Componentes creados localmente (sin origen global) usan:

```tsx
// @nur-ui: source=project created=2026-07-01 category=paneles
```

## Comando `--check`

```
Usuario: "add-visual-element --check"
```

1. Escanear todos los archivos en `<LIB_DIR>/` que tengan metadata `source=global`
2. Para cada uno, buscar el original en `~/.claude/nur-ui/components/`
3. Comparar hash local vs hash actual del vault
4. Reportar:

```
Componentes locales: 5
Componentes con origen global: 3

  DragonflyCard     ✓ actualizado (hash match)
  KpiCard           ✗ DESACTUALIZADO — local: a1b2c3 / vault: d4e5f6
  StatusBadge       ✓ actualizado (hash match)

1 componente desactualizado. Usá --update para sincronizar.
```

## Comando `--update <Componente>`

1. Mostrar diff entre versión local y vault
2. Pedir confirmación estricta
3. Si confirma: reemplazar archivo local con versión del vault, actualizar metadata (`copied` date)
4. Si rechaza: mantener versión local, registrar que el usuario rechazó la actualización

## Promover componente local a global

```
Usuario: "promové SuperAdminKpiCard a global"
```

1. Verificar que el componente existe en `<LIB_DIR>/`
2. Extraer metadata actual
3. Preguntar categoría global (o usar la local)
4. Copiar a `~/.claude/nur-ui/components/Nombre.tsx`
5. Registrar en `vault.json` con hash nuevo
6. Actualizar metadata del archivo local: `source=global` (ya era `source=project`)
7. Reportar: "`Nombre` ahora es global. Disponible para cualquier proyecto."

## Renombrar componente

```
Usuario: "renombrá KpiCard a MetricCard"
```

Operación atómica multi-archivo:

1. Buscar todas las referencias: barrel exports, design.md, manifest.json, imports en consumidores
2. Mostrar diff de todo lo que se va a cambiar
3. Pedir confirmación
4. Si confirma, ejecutar todos los cambios en secuencia:
   a. Renombrar archivo `<LIB_DIR>/KpiCard.tsx` → `<LIB_DIR>/MetricCard.tsx`
   b. Actualizar export en `index.ts`
   c. Actualizar nombre de función/componente en el archivo
   d. Actualizar referencias en `design.md` (todas las secciones)
   e. Actualizar `manifest.json`
   f. Corregir imports en consumidores conocidos
5. Si algún paso falla → rollback completo
6. Si es global: también renombrar en vault y `vault.json`

## Eliminar componente

```
Usuario: "eliminá StatusBadge de la librería"
```

1. Buscar TODOS los consumidores: `rg "StatusBadge" --glob "*.tsx" --glob "*.ts"`
2. Reportar lista de archivos que importan este componente
3. **No auto-corregir consumidores** — el usuario debe decidir
4. Si el usuario confirma:
   a. Eliminar archivo de `<LIB_DIR>/`
   b. Eliminar export de `index.ts`
   c. Eliminar entradas de `design.md`
   d. Actualizar `manifest.json`
   e. Listar consumidores rotos (quedarán con imports huérfanos)
5. Si es global: preguntar si eliminar también del vault

## Conflictos: versión local vs global

Cuando un `--check` o `--update` detecta diferencias:

1. Mostrar diff unificado lado a lado
2. Opciones:
   - Usar versión del vault (sobrescribir local)
   - Mantener versión local (ignorar update)
   - Mantener local y desvincular del vault (cambiar metadata a `source=project`)
   - Merge manual (abrir ambos archivos para que el usuario edite)
3. Si el usuario elige mantener local y desvincular → actualizar metadata, quitar `source=global`

## Componentes huérfanos

Si un componente en `<LIB_DIR>/` no está en `index.ts` ni es referenciado por nadie:
- `--check` lo reporta como "posible huérfano"
- No se elimina automáticamente
- El usuario decide si indexarlo o eliminarlo
