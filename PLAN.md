# Plan de implementación — Miti Miti (MVP local)

Objetivo de este plan: una PWA que funcione 100 % en el celular, sin backend ni login, donde quien pagó la cuenta pueda dividirla (en partes iguales o por ítem) y llevar el control de quién le debe, cuánto, y quién ya pagó.

Sigue las reglas de `CLAUDE.md`. Trabaja una fase a la vez y detente al final de cada una para revisión.

## Fuera de alcance por ahora

No implementes nada de esto, ni dejes preparado código para ello: logo e identidad visual definitiva, backend (Supabase), link compartido por WhatsApp, escaneo de facturas con IA, login y sincronización, app de tiendas (Capacitor), anuncios y monetización. Llegarán en planes posteriores.

---

## Fase 0 — Proyecto y herramientas

1. Crear el proyecto con Vite (plantilla `vue-ts`) en la raíz del repositorio.
2. Instalar y configurar Tailwind v4, Pinia, Vue Router, `idb`, `vite-plugin-pwa`, Vitest, ESLint (flat config con `typescript-eslint` y `eslint-plugin-vue`), Prettier y Knip.
3. `tsconfig` con las opciones estrictas de `CLAUDE.md`.
4. Scripts en `package.json`: `dev`, `build`, `preview`, `test`, `lint`, `format`, `knip`, `typecheck` y `check` (corre typecheck, lint, knip y test en ese orden).
5. Borrar todo el contenido de ejemplo de la plantilla (HelloWorld, logos de Vite y Vue, estilos por defecto, assets sin uso).
6. `src/style.css` con la paleta provisional como tokens de Tailwind (`@theme`):
   - `--color-brand`: un amarillo cálido provisional
   - `--color-owed`: coral, para lo que deben
   - `--color-paid`: verde, para lo pagado
   - `--color-ink` y `--color-surface`: texto y fondo
   - 8 colores para las etiquetas de personas (`--color-person-1` … `--color-person-8`)
7. App vacía con `App.vue` y router con una sola ruta `/` que muestre el título "Miti Miti".

**Listo cuando:** `npm run dev` muestra la página, `npm run check` pasa limpio y no queda ningún archivo de la plantilla sin uso.

---

## Fase 1 — Dominio (lógica pura con pruebas)

Todo en `src/domain/`, sin importar Vue ni APIs del navegador. Cada archivo con su `*.test.ts` al lado.

### 1.1 Tipos (`types.ts`)

```ts
type PaymentMethod = 'nequi' | 'daviplata' | 'breb' | 'cash' | 'other'
type SplitMode = 'equal' | 'itemized'
type Tip =
  | { kind: 'none' }
  | { kind: 'percent'; value: number }   // 10 = 10 %
  | { kind: 'amount'; value: number }    // pesos

interface Person  { id: string; name: string; colorIndex: number }   // colorIndex 1..8
interface Item    { id: string; name: string; unitPrice: number; quantity: number; shares: Record<string, number> }
                  // shares: personId -> peso relativo (1 = una parte). Varios = ítem compartido
interface Payment { id: string; personId: string; amount: number; method: PaymentMethod; paidAt: string }
interface Outing  {
  id: string; name: string; date: string; createdAt: string
  people: Person[]; payerId: string
  mode: SplitMode; items: Item[]
  billTotal: number | null   // total impreso en la factura antes de propina; null = suma de ítems
  tip: Tip; payments: Payment[]; archived: boolean
}
```

Si al implementar algún campo resulta innecesario, no lo agregues.

### 1.2 Dinero (`money.ts`)

- `formatCOP(amount)`: con `Intl.NumberFormat('es-CO', { style: 'currency', currency: 'COP', maximumFractionDigits: 0 })`.
- `parseCOP(text)`: acepta `"20000"`, `"20.000"`, `"$ 20.000"`; devuelve entero o `null` si no es válido. Rechaza negativos y decimales.
- `allocate(total, weights, step, remainderIndex)`: reparte un entero `total` proporcionalmente a `weights`:
  1. Calcula cuántos "pasos" (`step`, por ejemplo 100 pesos) caben en `total`.
  2. Reparte los pasos con el método de mayor residuo (empates: gana el de menor índice).
  3. Multiplica por `step` y suma la diferencia restante (menor que `step`) al índice `remainderIndex`.
  4. La suma del resultado es **siempre** exactamente `total`. Si todos los pesos son 0, lanza error.

### 1.3 División (`split.ts`)

- `itemTotal(item)` = `unitPrice × quantity`.
- `baseTotal(outing)` = `billTotal` si existe; si no, la suma de `itemTotal` de todos los ítems.
- `tipAmount(outing)`: `none` → 0; `percent` → `round(base × value / 100)`; `amount` → `value`.
- `grandTotal(outing)` = base + propina.
- `personSubtotals(outing)`: en modo `itemized`, cada ítem se reparte entre sus `shares` con `allocate(itemTotal, pesos, 1, 0)` y se suma por persona.
- `shares(outing)`: lo que le toca pagar a cada persona, como `Record<personId, number>`:
  - `equal`: `allocate(grandTotal, [1, 1, …], 100, índiceDelPagador)`.
  - `itemized`: `allocate(grandTotal, personSubtotals, 100, índiceDelPagador)`. Así la propina y los impuestos incluidos en `billTotal` se reparten en proporción a lo que cada uno consumió.
- `validate(outing)`: lista de problemas para mostrar en la UI: ítems sin asignar, `billTotal` menor que la suma de ítems, total 0, menos de 2 personas.

### 1.4 Deudas (`debts.ts`)

- `balances(outing)`: para cada persona que **no** es el pagador: `{ personId, share, paid, pending, status }` donde `paid` = suma de sus pagos, `pending = max(share − paid, 0)`, `status` es `'pending'` (no ha pagado nada), `'partial'` o `'paid'`. Si pagó de más, incluir `overpaid`.
- `isSettled(outing)`: todos en `'paid'`.
- `debtorsAcrossOutings(outings)`: para el tablero global "¿Quién me debe?". Toma las salidas no archivadas, agrupa por nombre normalizado (minúsculas, sin espacios extra ni tildes) y devuelve por persona el total pendiente y el detalle por salida, ordenado de mayor a menor deuda.

### Pruebas mínimas

- `allocate`: la suma siempre cuadra; reparto exacto; residuos; redondeo a 100 con sobrante al pagador; pesos 0 lanza error.
- `parseCOP` y `formatCOP` con los formatos colombianos.
- Caso de referencia: Miguel pide cazuela 20.000, Laura una bandeja 18.000, ambos comparten una jarra de 12.000; propina 10 %. Subtotales 26.000 y 24.000, total 55.000, y las partes suman exactamente 55.000.
- Ítem con cantidad 2 asignado a dos personas distintas.
- `billTotal` mayor que la suma de ítems (impoconsumo incluido): se reparte proporcional.
- `balances` con pago completo, abono parcial, pago de más y sin pagos.
- `debtorsAcrossOutings` agrupando "Miguel" y "miguel " como la misma persona.

**Listo cuando:** todas las funciones tienen pruebas y `npm run check` pasa.

---

## Fase 2 — Persistencia y estado

1. `src/storage/db.ts`: base IndexedDB `miti-miti`, almacén `outings` con `keyPath: 'id'`. Solo las funciones que use el store: listar todas, guardar una, borrar una.
2. `src/stores/outings.ts` (Pinia, setup store):
   - Carga todas las salidas al iniciar la app.
   - Acciones: crear salida, renombrar, agregar/editar/quitar persona, cambiar pagador, cambiar modo, agregar/editar/quitar ítem, asignar/desasignar persona a un ítem, fijar `billTotal`, fijar propina, registrar pago, borrar pago, archivar, borrar salida.
   - Cada acción persiste la salida modificada.
   - Getters que delegan en `src/domain/` (nunca recalculan por su cuenta).
   - IDs con `crypto.randomUUID()`.
   - Al crear una salida, se agrega automáticamente la persona "Yo" como pagador por defecto.
   - El color de cada persona nueva es el siguiente libre de los 8 (vuelve a empezar si se agotan).
3. Pruebas del store para las acciones que tienen lógica (por ejemplo, quitar una persona elimina sus asignaciones y sus pagos).

**Listo cuando:** recargar la página no pierde datos y `npm run check` pasa.

---

## Fase 3 — Pantallas

Diseño mobile-first (ancho base 360 px), una columna centrada con ancho máximo cómodo en PC. Textos en español colombiano neutro: cercano, sin regionalismos difíciles de traducir.

| Ruta | Pantalla | Contenido |
| --- | --- | --- |
| `/` | Inicio | Botón principal grande "¿Quién me debe?" con el total pendiente; lista de salidas (activas primero, luego saldadas; archivadas ocultas) con nombre, fecha, total y estado; botón flotante "Nueva cuenta". |
| `/debts` | ¿Quién me debe? | Lista de personas con su total pendiente, de mayor a menor; al tocar una persona se despliega el detalle por salida. Estado vacío: "Nadie te debe nada". |
| `/outings/new` | Nueva cuenta | Nombre (ej. "Almuerzo en El Corral"), fecha (hoy por defecto), personas como etiquetas de color (agregar con Enter, quitar con ×), quién pagó, modo: "Miti miti (partes iguales)" o "Cada quien lo suyo (por plato)". |
| `/outings/:id/items` | Ítems | Modo por plato: agregar ítem (nombre, precio unitario, cantidad); cada ítem muestra las etiquetas asignadas; al tocarlo se abre una hoja inferior con las personas para marcar (varias = compartido) y un atajo "Todos". Ítems sin asignar resaltados. Campo opcional "Total de la factura". Modo partes iguales: solo el total de la factura. |
| `/outings/:id/summary` | Resumen | Selector de propina (sin propina, 10 %, otro %, monto fijo), subtotal, propina, total, y lo que paga cada persona con su color. Avisos de `validate`. |
| `/outings/:id/payments` | Cobros | Lista de quienes deben con su parte, lo pagado y lo pendiente. Botón "Pagó todo" (registra el pendiente completo) y "Abono" (monto + medio de pago). Quien queda en cero se ve **tachado** con check verde pero sigue en la lista. Se pueden ver y borrar los pagos registrados. Cuando todos pagan: mensaje "Cuenta saldada" y opción de archivar. |

Detalles:

- Navegación dentro de una salida con una barra de tres pasos (Ítems, Resumen, Cobros).
- Inputs de dinero con teclado numérico (`inputmode="numeric"`) y formato de miles al salir del campo, usando `parseCOP` y `formatCOP`.
- Confirmación antes de borrar una salida, una persona con pagos o un pago.
- El color nunca es la única señal: pagado = tachado + ícono de check + color.

**Listo cuando:** se puede completar de principio a fin el caso de prueba manual de abajo en un celular real (o la vista móvil del navegador) y `npm run check` pasa.

### Caso de prueba manual

1. Crear "Almuerzo prueba" con Yo, Miguel y Laura; pagó Yo; modo por plato.
2. Agregar: Cazuela 20.000 → Miguel; Bandeja 18.000 → Laura; Jarra limonada 12.000 → Miguel y Laura; Café 4.000 → Yo.
3. Propina 10 %. Verificar que las partes sumen el total exacto.
4. Miguel paga todo por Nequi: aparece tachado. Laura abona 10.000 en efectivo: queda pendiente el resto.
5. Recargar la página: todo sigue igual.
6. En "¿Quién me debe?" aparece solo Laura con su saldo.
7. Laura paga el resto: la salida queda saldada y se puede archivar.

---

## Fase 4 — PWA

1. Configurar `vite-plugin-pwa`: nombre "Miti Miti", `display: standalone`, colores de tema desde la paleta provisional, `start_url: '/'`.
2. Ícono provisional: un cuadrado del color de marca con las letras "MM", en los tamaños que exige el manifest. Se reemplazará cuando exista el logo.
3. Precarga de la app para que abra sin conexión.
4. Probar la instalación en Android (Chrome) y la opción "Agregar a pantalla de inicio" en iPhone (Safari).

**Listo cuando:** la app se instala, abre sin internet y Lighthouse no reporta errores de PWA.

---

## Siguientes planes (no implementar ahora)

1. Identidad visual: logo, paleta e ícono definitivos.
2. Supabase y link compartido por WhatsApp para quien debe.
3. Escaneo de factura con modelo de visión (Edge Function).
4. Login, sincronización entre celular y PC, grupos recurrentes.
