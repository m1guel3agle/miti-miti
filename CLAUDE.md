# Miti Miti

PWA mobile-first (también usable en PC) para dividir cuentas de restaurante y llevar el control de quién debe y cuánto. El plan de trabajo está en `PLAN.md`: síguelo fase por fase.

## Stack

- Vue 3 (Composition API, `<script setup lang="ts">`) + Vite + TypeScript
- Tailwind CSS v4 (plugin `@tailwindcss/vite`)
- Pinia (estado) y Vue Router
- `idb` para persistir en IndexedDB
- `vite-plugin-pwa` para manifest, instalación y modo offline
- Vitest para pruebas
- ESLint (flat config) + `typescript-eslint` + `eslint-plugin-vue`, Prettier y Knip

Usa la última versión estable de cada paquete. No agregues dependencias fuera de esta lista sin preguntar primero.

## Reglas de código (obligatorias)

1. **Nada de código muerto.** Ninguna variable, función, prop, emit, tipo, export, archivo o dependencia que no se use. Si algo deja de usarse en un cambio, se borra en ese mismo cambio.
2. **Nada especulativo.** No crees abstracciones, parámetros, opciones de configuración o archivos "por si acaso" o "para más adelante". Construye solo lo que pide la tarea actual.
3. **Sin código comentado, sin `console.log`, sin `TODO`** en lo que entregas.
4. **Dinero siempre en pesos enteros** (`number` entero, nunca decimales ni `float`). Toda la aritmética de dinero vive en `src/domain/` y está probada.
5. **Lógica de negocio pura y separada de la UI.** `src/domain/` no importa Vue, Pinia ni nada del navegador. Los componentes no calculan montos: llaman funciones del dominio o getters del store.
6. **Tipos estrictos.** `strict: true`, `noUnusedLocals`, `noUnusedParameters`, `noFallthroughCasesInSwitch`, `noUncheckedIndexedAccess`. Prohibido `any` y `@ts-ignore`.
7. **Componentes pequeños.** Si un componente pasa de ~150 líneas o hace dos cosas, divídelo. Un componente solo se extrae si se usa o si reduce claramente la complejidad, no por adelantado.
8. **Nombres:** código (variables, funciones, archivos, tipos) en inglés; textos visibles para el usuario en español.
9. **Accesibilidad básica:** botones con texto o `aria-label`, inputs con `label`, contraste suficiente, áreas táctiles de al menos 44 px.

## Verificación antes de dar una tarea por terminada

Ejecuta `npm run check` y debe pasar sin errores ni advertencias. Ese script corre, en orden:

- `vue-tsc --noEmit` (tipos)
- `eslint .` con `--max-warnings 0`
- `knip` (exports, archivos y dependencias sin usar)
- `vitest run` (pruebas)

Si `knip` reporta algo, se elimina; no se silencia con configuración salvo los puntos de entrada reales del proyecto.

## Forma de trabajar

- Trabaja una fase de `PLAN.md` a la vez. Al terminar cada fase, detente, resume qué hiciste y espera revisión antes de seguir.
- Un commit por tarea, con Conventional Commits en inglés (`feat: add split calculation`).
- Sin logo ni identidad visual definitiva todavía: usa la paleta provisional de `src/style.css` mediante tokens, para poder cambiarla en un solo lugar.
