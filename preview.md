# Preview — Generación de preview visual

Módulo de generación de previews HTML para componentes NurUI (o librería equivalente).
Usa esbuild + Tailwind CSS compilado on-demand.

## Principio

Antes de indexar un componente, el usuario debe verlo y aprobarlo.
El preview se sirve vía servidor HTTP temporal con cleanup automático.

## Flujo de preview

### 1. Preparación

1. Leer el `.tsx` del componente creado en Paso 3
2. Detectar si existe `.preview.tsx` companion → usarlo
3. Si no existe `.preview.tsx` → auto-generar uno básico con defaults razonables
4. El `.preview.tsx` define: modos, variantes, datos mock, triggers interactivos

### 2. Compilación

```bash
npx esbuild <preview-entry>.tsx \
  --bundle \
  --outfile=<tmpdir>/preview.js \
  --external:react \
  --external:react-dom \
  --external:lucide-react \
  --external:@/lib/utils \
  --loader:.css=text \
  --define:__PREVIEW__=true
```

React y ReactDOM se sirven como `<script>` externos (CDN) en el HTML — no se bundlean.

### 3. Tailwind CSS

```bash
npx tailwindcss -i <tmpdir>/tailwind-entry.css -o <tmpdir>/preview.css --minify
```

El `tailwind-entry.css` referencia el `tailwind.config` del proyecto para heredar tokens.

### 4. Generar HTML

Template HTML que carga:
- React + ReactDOM desde CDN (esm.sh)
- `preview.css` (Tailwind compilado)
- `preview.js` (esbuild bundle)
- Montaje en `#root`

### 5. Servir y mostrar

1. Detectar puerto libre (base 48771, incrementar hasta encontrar libre, máx 10 intentos)
2. Levantar servidor HTTP estático: `python3 -m http.server <port> --directory <tmpdir>`
3. Abrir preview en navegador o mostrar URL
4. El servidor se mantiene hasta que el usuario confirma o cierra

### 6. Cleanup

Dos mecanismos:
- **Señal de cierre**: el HTML incluye `fetch('/__nur-ui-close')` en `window.addEventListener('beforeunload', ...)`. El servidor escucha esta ruta y se apaga.
- **TTL cleanup**: al iniciar el skill por próxima vez, limpiar directorios temporales de previews anteriores (>1h de antigüedad).

## `.preview.tsx` companion file

Archivo opcional que vive junto al componente y define cómo se previsualiza.
Si no existe, el skill auto-genera uno básico.

### Estructura

```tsx
// @nur-ui preview for <ComponentName>
import { ComponentName } from './ComponentName';

export const previewMeta = {
  component: 'ComponentName',
  mode: 'gallery' as 'gallery' | 'single',
  viewport: { width: 1200, height: 800 },
};

// ── Gallery mode: múltiples escenarios ──
export const scenarios = [
  {
    label: 'Default (light)',
    props: { variant: 'light', children: '...' },
    wrapper: { background: '#f8fafc', padding: '2rem' },
  },
  {
    label: 'Dark variant',
    props: { variant: 'dark', children: '...' },
    wrapper: { background: '#1e1f24', padding: '2rem' },
  },
  {
    label: 'Empty state',
    props: { variant: 'light', children: null },
    wrapper: { background: '#f8fafc', padding: '2rem' },
  },
];

// ── Single mode: un solo escenario con controls ──
export const singleMode = {
  defaultProps: { variant: 'light', label: 'Example' },
  controls: [
    {
      type: 'select',
      label: 'Variant',
      prop: 'variant',
      options: ['light', 'dark', 'amber'],
    },
    {
      type: 'text',
      label: 'Label',
      prop: 'label',
    },
    {
      type: 'boolean',
      label: 'Disabled',
      prop: 'disabled',
    },
  ],
};

// ── Interactive triggers (opcional) ──
export const interactive = {
  enabled: true,
  triggers: [
    {
      type: 'button',
      label: 'Toggle active',
      action: 'toggle-prop',
      prop: 'active',
    },
    {
      type: 'hover',
      target: '.nurui-card',
      action: 'add-class',
      value: 'hovered',
    },
    {
      type: 'click-outside',
      action: 'reset',
    },
  ],
};

// ── Mock data (opcional) ──
export const mockData = [
  { id: 1, name: 'Clínica A', revenue: 12500 },
  { id: 2, name: 'Clínica B', revenue: 8700 },
];

// ── Mock fetch (opcional — simula async data loading) ──
export const mockFetch = {
  delay: 800,           // ms antes de resolver
  response: { data: mockData },
  states: ['loading', 'error', 'empty', 'normal'] as string[],
};
```

### Auto-detección (sin `.preview.tsx`)

Si no hay companion file:
1. Leer las props de la interface del componente
2. Generar un escenario por cada variante declarada
3. Usar valores default razonables para cada prop type
4. Nombre del wrapper: "Auto-generated — create `.preview.tsx` for better control"

## Modos de preview

### Gallery mode
- Muestra todos los escenarios en una grilla
- Cada escenario tiene su label, fondo, y props
- Útil para componentes con múltiples variantes/estados
- El usuario ve todo de una vez

### Single mode
- Un solo escenario con panel de controles
- Selectores, inputs, toggles para cambiar props en vivo
- Útil para iterar sobre un diseño específico
- Los controles mutan props y re-renderizan en caliente

## Triggers interactivos

Permiten probar comportamiento sin escribir código:

| Tipo | Acción |
|------|--------|
| `button` | Agrega un botón en la UI de preview que ejecuta una acción |
| `toggle` | Toggle boolean que cambia una prop |
| `hover` | Simula hover en un selector CSS |
| `click-outside` | Simula click fuera del componente |

## Mock data y async states

Si el componente consume datos asíncronos:
- `mockData`: array de datos que se pasa como props
- `mockFetch`: simula loading → response, con delay configurable
- `states`: el preview muestra cada estado (loading skeleton, error, empty, normal) en secuencia o en tabs

## Resolución de puertos

```
Base: 48771
Intentos: máx 10
Algoritmo: base → base+1 → base+2 ... hasta encontrar libre
Si los 10 están ocupados → error y sugerir liberar puertos
```

## Cleanup automático

Al iniciar el skill:
```bash
# Eliminar directorios de preview con >1h de antigüedad
fd --type d '^preview-' <tmpdir> --changed-before 1h | xargs rm -rf
```

## Dependencias necesarias

El proyecto debe tener:
- `esbuild` (o instalarlo temporalmente: `npx esbuild`)
- `tailwindcss` (o usar el del proyecto)
- `python3` (para el servidor HTTP)
