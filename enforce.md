# Enforce — Enforcement registry y refactor

Módulo de enforcement, auditoría y refactor quirúrgico. El enforcement registry es el mecanismo que garantiza consistencia visual: una vez que un UI type se asigna a un componente de librería, toda creación nueva de ese UI type debe usar el componente registrado.

## Enforcement Registry

### Ubicación

En `docs/design/design.md`, entre los marcadores:
```markdown
<!-- @nur-ui:start enforcement -->
...
<!-- @nur-ui:end enforcement -->
```

### Formato

```markdown
| UI Type | Componente | Regla |
|---------|-----------|-------|
| KPI Card | KpiCard | Usar KpiCard de NurUI. No crear tarjetas KPI manualmente. |
| Panel / Card Container | DragonflyCard | Usar DragonflyCard para cualquier contenedor tipo card con sombra. |
| Status Badge | StatusBadge | Usar StatusBadge. No usar spans/badges inline. |
```

### Precedencia

1. **Enforcement registry** — si un UI type está registrado, su componente es OBLIGATORIO
2. **Catálogo descriptivo** — sugiere componentes disponibles pero no obliga
3. **Nuevo desarrollo** — si no hay match en el registry, se crea libremente

El registry GANA sobre cualquier otra sección de design.md. Si hay conflicto, el registry tiene prioridad.

### Interceptación transversal

**No se limita a `add-visual-element`.** Cualquier skill o agente que vaya a crear UI debe:

1. Antes de crear un elemento visual, leer el enforcement registry
2. Si el UI type solicitado tiene un componente registrado → usar ese componente
3. Si no tiene → crear libremente, y al finalizar preguntar si quiere registrar el nuevo UI type

Esto se implementa vía una regla en CLAUDE.md:

```markdown
## Interceptación visual (NurUI enforcement)

Antes de crear CUALQUIER elemento visual (componente, tarjeta, badge, panel, botón, etc.):
1. Leer `<!-- @nur-ui:start enforcement -->` en `docs/design/design.md`
2. Si el UI type está en el registry → usar el componente registrado
3. Si no está → crear normalmente y proponer registro al finalizar
```

### Registrar un nuevo UI type

Después de crear un componente y que el usuario lo apruebe:

1. Preguntar: "¿Querés registrar `KPI Card → KpiCard` como enforcement? Esto significa que en el futuro todo KPI card deberá usar KpiCard."
2. Si acepta → agregar fila al enforcement registry
3. Si rechaza → solo queda en el catálogo descriptivo

### Modificar una regla existente

Solo con confirmación explícita. Si el usuario pide cambiar el componente asignado a un UI type:
1. Mostrar la regla actual y la propuesta
2. Listar consumidores actuales que se verían afectados
3. Pedir confirmación
4. Actualizar el registry

## Auditoría dry-run

```
Usuario: "auditá los componentes visuales del proyecto"
        → o: "qué patrones visuales están duplicados?"
```

1. Escanear todos los `.tsx` en busca de patrones visuales recurrentes:
   - Cards con `bg-white shadow-sm rounded-xl`
   - Badges con `px-2 py-1 rounded-full text-xs`
   - Botones con estilos repetidos
   - KPIs con estructuras similares
2. Agrupar por similitud estructural (mismo patrón de clases Tailwind)
3. Para cada grupo, verificar si existe un componente NurUI que lo cubra
4. Reportar:

```
Auditoría visual — 3 patrones duplicados encontrados:

1. KPI Cards (8 instancias)
   - superadmin/components/SuperAdminKpiCard.tsx
   - modules/reportes/components/ReporteKpi.tsx
   - modules/facturacion/components/FacturaMetrics.tsx
   → Ya existe KpiCard en NurUI. Candidato a refactor.

2. Status Badges (5 instancias)
   - Span con clases inline en 5 archivos
   → No existe componente. Candidato a extraer.

3. Panel containers (12 instancias)
   - bg-white shadow-sm rounded-xl en 12 archivos
   → Ya existe DragonflyCard. Candidato a refactor.
```

## Refactor quirúrgico

```
Usuario: "refactorizá todos los KPI cards para usar KpiCard"
```

### Fase 1: Auditoría (dry-run)

- Escanear consumidores
- Identificar variantes necesarias (¿alguno usa props que KpiCard no soporta?)
- Reportar hallazgos SIN modificar nada

### Fase 2: Planificación

- Agrupar consumidores en batches independientes
- Un batch por UI type
- Cada batch es atómico: se completa o se revierte

### Fase 3: Ejecutar batch

Para cada batch:
1. Reemplazar implementación inline por uso del componente
2. Verificar tipos: `npx tsc --noEmit`
3. Si falla → corregir o revertir el batch
4. Si pasa → commit del batch

### Fase 4: Checkpoint

Después de cada batch:
- `tsc --noEmit` limpio
- Preview de regresión (si hay `.preview.tsx` disponibles)
- El usuario confirma antes del próximo batch

### Fase 5: Preview de regresión

Al final de todos los batches:
- Generar previews de todas las páginas afectadas
- El usuario revisa visualmente que nada se rompió

### Rollback granular

Si un batch falla:
1. `git checkout -- <archivos del batch>` — revertir solo ese batch
2. Los batches anteriores (ya commiteados) no se tocan
3. Reportar qué batch falló y por qué
4. El usuario decide: corregir y reintentar, o saltar ese batch

## Migración inicial de proyecto existente

Cuando se corre `add-visual-element` por primera vez en un proyecto que ya tiene componentes visuales pero sin NurUI:

1. **Detectar estructura actual**: buscar directorios con componentes visuales reutilizables
2. **Backup de design.md**: `design.md.bak-YYYYMMDD` (solo si existe)
3. **Insertar marcadores** en design.md (nuevo o existente)
4. **Sembrar catálogo**: escanear componentes existentes y registrarlos
5. **No crear enforcement rules** — eso lo decide el usuario gradualmente
6. **Reportar**:
   ```
   NurUI inicializado en este proyecto.
   - 5 componentes detectados y catalogados
   - 0 enforcement rules (las creás vos cuando quieras fijar un UI type)
   - design.md actualizado con marcadores
   - Backup: docs/design/design.md.bak-20260701
   ```
7. **Validación**: el usuario revisa. Si rechaza → restaurar backup.

## Regla de modularidad en CLAUDE.md

Al inicializar o migrar, insertar en CLAUDE.md:

```markdown
## Componentes visuales — Modularidad obligatoria

- Usar componentes de `<LIB_NAME>` para todo elemento visual repetible.
- Si un patrón visual aparece 2+ veces → DEBE ser un componente de librería.
- No crear estilos inline que repliquen un componente existente.
- Antes de crear un componente visual nuevo, verificar el enforcement registry en `<DESIGN_MD>`.

### Delegación de decisiones visuales

Toda decisión de diseño visual se delega a `<DESIGN_MD>`. En caso de conflicto entre este archivo y design.md, **design.md gana**.
```

Esta inserción se hace automáticamente al crear la librería. Si CLAUDE.md ya existe, se agrega al final. Si ya tiene la sección, se actualiza con los paths correctos.
