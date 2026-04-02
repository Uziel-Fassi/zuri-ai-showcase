# MIGRATION-VUE3-PLAN.md
## GENIE.AI / ZURI AI Frontend — Plan de Migración Vue 2 → Vue 3

**Fecha de análisis:** 30 de marzo de 2026  
**Analista:** GitHub Copilot  
**Vue 2 EOL:** 31 de diciembre de 2023 (sin más parches de seguridad)

---

## Resumen Ejecutivo

**Buenas noticias:** El proyecto ya tiene su `package.json` completamente actualizado a Vue 3.
Las dependencias principales (`vue@^3.2`, `vue-router@^4`, `vuex@^4`, `vue-i18n@^9`) ya son las versiones correctas para Vue 3. `main.js` también usa el patrón correcto `createApp()`.

**El problema:** El código fuente de los componentes (.vue) no fue migrado cuando se actualizaron las dependencias — la mayoría todavía usa sintaxis de Vue 2. Actualmente el proyecto funciona *parcialmente* porque Vue 3 tolera algo de sintaxis heredada, pero hay fallos silenciosos confirmados (render functions, `$once`, `beforeDestroy` sin renombrar) que causan memory leaks y componentes que no renderizan correctamente.

**Esfuerzo estimado:** 3–4 sprints de 2 semanas.

---

## 1. Inventario de Dependencias

### 1.1 Estado actual de dependencias (package.json)

| Paquete | Versión actual | Estado Vue 3 |
|---|---|---|
| `vue` | `^3.2.0` | ✅ Correcto |
| `vue-router` | `^4.0.0` | ✅ Correcto |
| `vuex` | `^4.0.2` | ✅ Correcto (compatible con Vue 3) |
| `vue-i18n` | `^9.14.2` | ✅ Correcto (v9 es para Vue 3) |
| `vue3-apexcharts` | `^1.8.0` | ✅ Correcto |
| `@vue/cli-service` | `^5.0.0` | ✅ Correcto (soporta Vue 3) |

**No hay dependencias Vue-2-específicas que eliminar** (no hay `vue-template-compiler`, `@vue/composition-api`, `vue-class-component`, etc.).

### 1.2 Dependencias candidatas para revisión futura

| Paquete | Versión | Nota |
|---|---|---|
| `vuex` | `^4.0.2` | Funciona, pero considerar migrar a **Pinia** en sprint futuro. Pinia es el gestor de estado oficial recomendado para Vue 3. |
| `axios` | `^0.27.2` | Versión v0.x antigua. Actualizar a `^1.x` (breaking change: manejo de errores diferente). No es Vue-específico pero es deuda técnica. |
| `@vue/test-utils` | No instalado | Si se añaden tests unitarios, instalar `@vue/test-utils@^2.x`. |

---

## 2. Inventario de Problemas por Archivo

### 2.1 🔴 CRÍTICO — Fallos activos en producción

Estos problemas causan silently broken behavior *ahora mismo*.

#### `NotificationSystem.vue` (raíz del proyecto, posiblemente no importado)
- **Problema:** Render functions de Vue 2 en 4 sub-componentes inline.
  - `render(h) { return h('svg', { attrs: {...} }, [...]) }` — Vue 3 eliminó el argumento `h`; debe importarse: `import { h } from 'vue'`.
  - La sintaxis `{ attrs: {}, class: '' }` fue aplanada en Vue 3: `{ class: '', ...attrs }`.
  - **Resultado actual:** Los 4 iconos SVG (IconSuccess, IconError, IconInfo, IconWarning) no renderizan nada.
- **Fix:** Reescribir los 4 componentes inline como templates `<template>` simples, o usar `h` correctamente importado.
- **⚠️ Nota:** Este archivo no parece importado en ningún componente de `src/`. Verificar si está en uso antes de invertir esfuerzo.

#### `AnalyticsDashboard.vue` — `$once` eliminado
- **Problema:** `this.$once("hook:beforeDestroy", () => { clearInterval(intervalId); })` — `$once` fue eliminado de la instancia del componente en Vue 3. Este código **no hace nada** actualmente — el intervalo nunca se limpia.
- **Resultado actual:** Memory leak — intervalos activos después de que el componente se desmonta.
- **Fix:** Añadir un lifecycle hook `beforeUnmount()` que llame a `clearInterval(intervalId)`.

#### `App.vue` — Clase CSS de transición Vue 2
- **Problema:** La clase `.fade-enter` debe ser `.fade-enter-from` en Vue 3. Vue 2 usaba `.v-enter`; Vue 3 lo renombró.
  ```css
  /* Vue 2 (actual — roto) */
  .fade-enter,
  .fade-leave-to { opacity: 0; }

  /* Vue 3 (correcto) */
  .fade-enter-from,
  .fade-leave-to { opacity: 0; }
  ```
- **Resultado actual:** La animación fade del router no se aplica al entrar. La transición de salida (`.fade-leave-to`) sí funciona.
- **Fix:** 1 línea de CSS.

---

### 2.2 🟠 ALTO — `beforeDestroy` sin renombrar (memory leaks potenciales)

`beforeDestroy` en Vue 3 simplemente no se ejecuta — equivale a no tener cleanup hook. Cada uno de estos causa que listeners DOM y event bus queden registrados indefinidamente.

| Archivo | Línea | Contenido que limpia |
|---|---|---|
| `ChatFolders.vue` | 711 | Event listener DOM (`input.search-box`), `eventBus.$off("conversation-saved")` |
| `ModalComponent.vue` | 80 | `document.removeEventListener('keydown', ...)`, restaura `body.style.overflow` |
| `ChatResponseFeedbackDialog.vue` | 225 | `document.removeEventListener("keydown", this.escHandler)` |
| `UserProfileComponent.vue` | 1539 | `window.removeEventListener("themeChange", ...)` |
| `SearchableCountryDropdown.vue` | 132 | Click-outside listener (`document.addEventListener`) |
| `PersonalIdentificationTab.vue` | 273 | debounce timer cleanup |

**`dialogThemeUtils.js` líneas 143–146** — utilidad que **inyecta** `beforeDestroy` dinámicamente en componentes pasados como argumento:
```js
const originalBeforeDestroy = component.beforeDestroy || (() => {});
component.beforeDestroy = function() { ... }
```
Este patrón no funcionará en Vue 3. Requiere ser actualizado a `beforeUnmount`.

**Fix para todos:** Renombrar `beforeDestroy()` → `beforeUnmount()` en cada archivo.

---

### 2.3 🟠 ALTO — Uso de `$root` para acceder a `$i18n`

#### `ChatBotComponent.vue` (3 instancias)
```js
// Actual — frágil en Vue 3
if (this.$root.$i18n) {
  this.currentLocale = this.$root.$i18n.locale;
  this.$watch(() => this.$root.$i18n.locale, ...)
}
```
`$root` en Vue 3 es la instancia del componente raíz (App.vue), no el objeto `app` de Vue. `$i18n` no está en `$root` — está en la instancia actual a través del plugin.

**Fix:** Cambiar a acceso directo:
```js
// Vue 3 correcto
this.currentLocale = this.$i18n.locale;
this.$watch(() => this.$i18n.locale, ...)
```

#### `SettingsComponent.vue` (2 instancias)
```js
// Actual — anti-patrón
if (this.$root) {
  this.$root.$forceUpdate();
}
```
`$root.$forceUpdate()` fuerza un re-render del componente raíz, no del árbol completo. En Vue 3, los cambios de estado reactivo deberían propagarse sin `$forceUpdate`. Esto es una señal de estado no reactivo.

**Fix:** Revisar por qué se necesita `$forceUpdate` y hacerlo reactivo en su lugar.

---

### 2.4 🟡 MEDIO — `$forceUpdate` como anti-patrón (3 instancias en `AdminDashboard.vue`)

`AdminDashboard.vue` llama a `this.$forceUpdate()` en tres lugares (líneas ~2735, ~2914, ~3216). Si bien `$forceUpdate()` existe en Vue 3, su necesidad indica estado no reactivo. Al migrar a Composition API, estos deberían resolverse con `ref()` o `reactive()` correctamente.

---

### 2.5 🟡 MEDIO — Event Bus (`eventBus.js`)

El event bus actual es un objeto POJO personalizado (`src/eventBus.js`) — no usa `new Vue()`, por lo que técnicamente ya es "Vue 3 compatible". Sin embargo, el patrón de event bus global está desaconsejado en Vue 3.

**Componentes que usan `eventBus`:**

| Componente | Eventos subscribe (`$on`) | Eventos emite (`$emit`) |
|---|---|---|
| `App.vue` | `notification:show` | — |
| `ChatBotComponent.vue` | `chat-deleted`, `load-conversation`, `treeNodeSelected`, `open-chat` | `contextItemRemoved`, `conversation-saved` |
| `ChatFolders.vue` | `conversation-saved` | `load-conversation`, `chat-deleted` |
| `ServiceTreePanelComponent.vue` | `contextItemRemoved` | `treeNodeSelected` |
| `ServiceTreeContainer.vue` | `reload-service-tree` | — |
| `AdminDashboard.vue` | — | `notification:show`, `knowledge-hierarchy-updated` |
| `LoginScreen.vue` | — | `notification:show` |
| `AddFromLinkDialog.vue` | — | `notification:show` |
| `FileDetailsDialog.vue` | — | `notification:show` |
| `UploadFilesDialog.vue` | — | `notification:show` |
| `notificationService.js` (service) | — | `notification:show` |

**Recomendación a futuro:** Reemplazar el event bus de `notification:show` con el sistema de notificaciones del Plugin ([`app.config.globalProperties.$notify`]) o una librería como `vue-toastification`. Los eventos de coordinación de componentes (`chat-deleted`, `load-conversation`) deberían moverse al store Vuex/Pinia.

---

### 2.6 🟢 BAJO — Options API (requiere reescritura eventual a Composition API)

Todos los componentes usan Options API. Vue 3 soporta Options API completamente, por lo que **esto no es un bloqueador**. Sin embargo, para aprovechar los beneficios de Vue 3 (mejor TypeScript, mejor tree-shaking, `<script setup>`), eventualmente se reescribirán.

**Componentes con lógica compleja (priorizados para Composition API):**

| Componente | Razón de complejidad | LOC aprox. |
|---|---|---|
| `ChatBotComponent.vue` | Hub central: i18n, Vuex, event bus, múltiples watchers, query pipeline | ~1800 |
| `AdminDashboard.vue` | Dashboard completo: CRUD, analytics, 3x $forceUpdate | ~3300 |
| `UserProfileComponent.vue` | Múltiples tabs, watchers, validaciones, `beforeDestroy` | ~1700 |
| `ChatFolders.vue` | Drag-and-drop, Vuex, event bus, `beforeDestroy` | ~1500 |
| `AnalyticsDashboard.vue` | Charts, intervals, `$once` bug | ~600 |

**Componentes simples (candidatos para migración rápida con `<script setup>`):**
`ConfirmDialog.vue`, `ModalDialog.vue`, `FloatingChatWidget.vue`, `LanguageSelector.vue`, `SplashScreen.vue`, todos los `charts/` individuales.

---

### 2.7 🟢 BAJO — Router duplicado

Existe `src/router.js` (activo, importado por `main.js`) y `src/router/index.js` (legacy, no importado). El archivo legacy referencia `'../views/Dashboard.vue'` que no existe. Eliminar `src/router/index.js`.

---

## 3. Resumen de Inventario de Problemas

| Severidad | Categoría | Archivos afectados | Cuenta |
|---|---|---|---|
| 🔴 Crítico | Render functions Vue 2 | `NotificationSystem.vue` | 4 funciones |
| 🔴 Crítico | `$once("hook:beforeDestroy")` eliminado | `AnalyticsDashboard.vue` | 1 |
| 🔴 Crítico | CSS de transición Vue 2 | `App.vue` | 1 clase |
| 🟠 Alto | `beforeDestroy` sin renombrar | 6 componentes + `dialogThemeUtils.js` | 7 |
| 🟠 Alto | `this.$root.$i18n` / `this.$root.$forceUpdate` | `ChatBotComponent.vue`, `SettingsComponent.vue` | 5 instancias |
| 🟡 Medio | `$forceUpdate` anti-patrón | `AdminDashboard.vue` | 3 |
| 🟡 Medio | Event bus (funciona, mejor reemplazar) | 10 componentes | ~25 usos |
| 🟢 Bajo | Options API sin convertir | 52 componentes | 52 |
| 🟢 Bajo | Router legacy no usado | `src/router/index.js` | 1 archivo |

---

## 4. Roadmap Sprint a Sprint

### Sprint 1 — Semana 1–2: "Apagar Incendios" (Bugs Activos)
**Objetivo:** Eliminar todos los fallos silenciosos que ya están rompiendo la app en producción.

| # | Tarea | Archivo(s) | Esfuerzo |
|---|---|---|---|
| S1.1 | Renombrar `.fade-enter` → `.fade-enter-from` en CSS | `App.vue` | 15 min |
| S1.2 | Reemplazar `$once("hook:beforeDestroy")` por `beforeUnmount()` | `AnalyticsDashboard.vue` | 30 min |
| S1.3 | Renombrar `beforeDestroy` → `beforeUnmount` en 6 componentes | `ChatFolders.vue`, `ModalComponent.vue`, `ChatResponseFeedbackDialog.vue`, `UserProfileComponent.vue`, `SearchableCountryDropdown.vue`, `PersonalIdentificationTab.vue` | 1–2h |
| S1.4 | Actualizar `dialogThemeUtils.js` para usar `beforeUnmount` | `src/utils/dialogThemeUtils.js` | 30 min |
| S1.5 | Corregir acceso `this.$root.$i18n` → `this.$i18n` | `ChatBotComponent.vue` | 30 min |
| S1.6 | Decidir si `NotificationSystem.vue` está en uso; si sí, reescribir render functions | `NotificationSystem.vue` | 1–3h |
| S1.7 | Eliminar `src/router/index.js` (legacy no usado) | `src/router/index.js` | 5 min |

**Criterio de éxito:** 0 memory leaks en DevTools al navegar entre todas las rutas. Animación fade del router funciona al entrar y salir.

---

### Sprint 2 — Semana 3–4: "Limpiar Anti-patrones"
**Objetivo:** Eliminar `$root`, `$forceUpdate`, y modernizar el event bus de notificaciones.

| # | Tarea | Archivo(s) | Esfuerzo |
|---|---|---|---|
| S2.1 | Reemplazar `this.$root.$forceUpdate()` con estado reactivo | `SettingsComponent.vue` | 2–3h |
| S2.2 | Refactorizar los 3 `$forceUpdate` en AdminDashboard | `AdminDashboard.vue` | 3–4h |
| S2.3 | Instalar `vue-toastification` o similar; reemplazar `eventBus.$emit('notification:show')` en todos los componentes | 10 componentes | 2–4h |
| S2.4 | Actualizar `axios` de v0.27 a v1.x; ajustar manejo de errores | todos los `services/` | 3–4h |
| S2.5 | Añadir `@vue/test-utils@^2` + configurar Jest/Vitest | `package.json`, nueva config | 4–6h |

**Criterio de éxito:** No quedan `$root` ni `$forceUpdate` en el codebase. Las notificaciones usan una librería estándar.

---

### Sprint 3 — Semana 5–6: "Migrar Vuex a Pinia" *(Opcional pero recomendado)*
**Objetivo:** Reemplazar Vuex 4 con Pinia para mejor DevX en Vue 3.

| # | Tarea | Esfuerzo |
|---|---|---|
| S3.1 | Instalar Pinia; crear `stores/chatHistory.ts` equivalente a `chatHistoryStore.js` | 4–6h |
| S3.2 | Migrar `stores/auth.ts` (actualmente módulo Vuex sin namespace) | 2–3h |
| S3.3 | Actualizar `App.vue`, `ChatBotComponent.vue`, `ChatFolders.vue` para usar `useChatHistoryStore()` | 4–6h |
| S3.4 | Actualizar todos los demás componentes que acceden a `this.$store` directamente | 3–4h |
| S3.5 | Eliminar Vuex del proyecto | 30 min |

**Criterio de éxito:** `vuex` removido de `package.json`. Estado de la app funcionando con Pinia. DevTools de Pinia operativas.

---

### Sprint 4 — Semana 7–8+: "Composition API para Componentes Críticos"
**Objetivo:** Reescribir los 5 componentes más complejos a `<script setup>` + Composition API. Los 47 componentes simples pueden quedarse en Options API — Vue 3 los soporta indefinidamente.

| # | Tarea | Esfuerzo |
|---|---|---|
| S4.1 | `AnalyticsDashboard.vue` → `<script setup>` (el más pequeño de los críticos) | 1 día |
| S4.2 | `UserProfileComponent.vue` → `<script setup>` | 1.5 días |
| S4.3 | `ChatFolders.vue` → `<script setup>` | 1.5 días |
| S4.4 | `AdminDashboard.vue` → `<script setup>` (el más grande, ~3300 LOC) | 3–4 días |
| S4.5 | `ChatBotComponent.vue` → `<script setup>` (hub central, ~1800 LOC) | 2–3 días |
| S4.6 | Migrar todos los `charts/` → `<script setup>` simple | 1 día |

**Patrón de referencia para la migración:**
```html
<!-- Antes (Options API) -->
<script>
import { mapGetters } from 'vuex'
export default {
  data() { return { items: [] } },
  computed: { ...mapGetters(['currentUser']) },
  mounted() { this.loadItems() },
  methods: { async loadItems() { ... } }
}
</script>

<!-- Después (Composition API con <script setup>) -->
<script setup>
import { ref, onMounted } from 'vue'
import { useStore } from 'pinia' // o useAuthStore() con Pinia
const store = useAuthStore()
const items = ref([])
const loadItems = async () => { ... }
onMounted(loadItems)
</script>
```

**Criterio de éxito:** Los 5 componentes críticos operativos con `<script setup>`. TypeScript puede adoptarse gradualmente en este punto.

---

## 5. Consideraciones de Testing

- Antes de cada sprint, tomar un snapshot del comportamiento con E2E (Cypress o Playwright) para validar que los refactors no rompan funcionalidad.
- Vue Test Utils v2 (`@vue/test-utils@^2`) para unit tests de componentes.
- Priorizar tests de `ChatBotComponent.vue` (query pipeline) y `AdminDashboard.vue` antes de refactorizar.

---

## 6. Checklist de Pre-Sprint Rápido (Sprint 1 se puede hacer en 1 día)

```
[ ] App.vue: .fade-enter → .fade-enter-from
[ ] AnalyticsDashboard.vue: $once → beforeUnmount()
[ ] ChatFolders.vue: beforeDestroy → beforeUnmount
[ ] ModalComponent.vue: beforeDestroy → beforeUnmount
[ ] ChatResponseFeedbackDialog.vue: beforeDestroy → beforeUnmount
[ ] UserProfileComponent.vue: beforeDestroy → beforeUnmount
[ ] SearchableCountryDropdown.vue: beforeDestroy → beforeUnmount
[ ] PersonalIdentificationTab.vue: beforeDestroy → beforeUnmount
[ ] dialogThemeUtils.js: beforeDestroy → beforeUnmount (x2)
[ ] ChatBotComponent.vue: this.$root.$i18n → this.$i18n (x3)
[ ] Eliminar src/router/index.js (legacy)
[ ] Verificar NotificationSystem.vue (¿está importado en algún lugar?)
```

---

*Documento generado en base a análisis estático del código fuente. Última revisión: 30 de marzo de 2026.*
