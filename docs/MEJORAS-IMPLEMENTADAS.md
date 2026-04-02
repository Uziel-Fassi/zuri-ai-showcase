# Reporte de Mejoras Implementadas

## Introducción

Este documento describe las mejoras de seguridad, estabilidad y capacidades multilingüe implementadas en el sistema Genie AI / ZURI, basadas en el reporte de auditoría técnica.

---

## Fase A — Correcciones de Seguridad Crítica

### Fix 1 — Secreto JWT hardcodeado eliminado
**Archivo:** `genie-ai-overlay/http-service/app.py`

El secreto JWT estaba definido directamente en el código fuente con un valor base64 de 88 caracteres. Cualquier persona con acceso al repositorio podría haberlo usado para falsificar tokens de autenticación.

**Cambio aplicado:** El secreto ahora se lee exclusivamente de la variable de entorno `JWT_SECRET`. Si la variable no está definida al arrancar el servicio, la aplicación lanza un `RuntimeError` inmediatamente, impidiendo que corra en estado inseguro.

---

### Fix 2 — Regex de CORS sin anclas (bypass de subdominio)
**Archivo:** `components/gov-chat-backend/index.js`

El patrón regex para validar orígenes CORS no tenía anclas `^` y `$`. Esto permitía que un atacante usara un dominio como `evil.localhost.attacker.io` y pasara la validación porque el string `localhost` aparece en algún lugar del dominio.

**Cambio aplicado:** El código ahora construye el regex con `^` al inicio y `$` al final automáticamente, de forma que solo coincide el dominio exacto.

---

### Fix 3 — `jwt.verify()` sin restricción de algoritmo
**Archivo:** `components/gov-chat-backend/services/auth-service.js`

Las tres llamadas a `jwt.verify()` en el servicio de autenticación no especificaban el algoritmo permitido. Esto habilita el ataque "alg:none" (tokens sin firma aceptados) y la confusión HS256/RS256.

**Cambio aplicado:** Las tres llamadas ahora incluyen `{ algorithms: ['HS256'] }` como segundo argumento, rechazando cualquier token firmado con otro algoritmo.

---

## Fase B — Protección contra DoS y Estabilidad

### Fix 4 — Rate limiter aplicado después del body parser
**Archivo:** `components/gov-chat-backend/index.js`

El middleware de rate limiting estaba registrado *después* del body parser. Esto significa que una solicitud con un cuerpo de 50 MB se leía completamente en memoria antes de que el sistema decidiera si el cliente había superado su límite de peticiones. Un atacante podía mandar cientos de requests grandes y agotar la RAM del servidor.

**Cambio aplicado:** El rate limiter se movió antes del body parser en la cadena de middleware.

---

### Fix 5 — Límite global del body parser reducido
**Archivo:** `components/gov-chat-backend/index.js`

El parser global aceptaba cuerpos de hasta 50 MB para todas las rutas. El límite quirúrgico de 50 MB se mantiene únicamente para `/api/database` que lo necesita; el resto del sistema opera con 1 MB por defecto.

---

### Fix 6 — Validación de archivos solo por MIME type (sin magic bytes)
**Archivo:** `components/gov-chat-backend/routes/user-routes.js`

La validación de uploads verificaba únicamente el `Content-Type` declarado por el cliente, que puede falsificarse trivialmente. Un archivo ejecutable con extensión `.pdf` pasaba la validación.

**Cambio aplicado:** Se implementó validación por **magic bytes** — los primeros bytes del archivo determinan su tipo real:
- PDF: `%PDF`
- JPEG: `FF D8 FF`
- PNG: firma de 8 bytes estándar
- ZIP/OOXML (docx, xlsx): `PK\x03\x04`

También se agregó sanitización de nombre de archivo para prevenir path traversal (`../../../etc/passwd`).

---

### Fix 7 — Input de autenticación sin validación de esquema
**Archivo:** `components/gov-chat-backend/routes/auth-routes.js`

Los endpoints `/register`, `/login` y `/refresh-token` aceptaban cualquier payload sin validar estructura ni tipos.

**Cambio aplicado:** Se implementaron esquemas Joi para cada endpoint:
- `loginName`: alfanumérico, 3–30 caracteres
- `email`: formato RFC válido
- `encPassword`: mínimo 6 caracteres
- Campos desconocidos rechazados con `unknown(false)`

---

## Fase C — Correcciones de Migración a Vue 3

### Fix 8 — Lifecycle hooks de Vue 2 en componentes Vue 3
**Archivos afectados:** `ChatFolders.vue`, `ChatResponseFeedbackDialog.vue`, `ModalComponent.vue`, `PersonalIdentificationTab.vue`, `SearchableCountryDropdown.vue`, `UserProfileComponent.vue`

En Vue 3, el hook `beforeDestroy()` fue renombrado a `beforeUnmount()`. Los componentes que usaban `beforeDestroy` nunca ejecutaban su código de limpieza (cancelar timers, remover listeners), causando memory leaks silenciosos.

**Cambio aplicado:** Todos los usos de `beforeDestroy()` reemplazados por `beforeUnmount()`.

---

### Fix 9 — Patrón `$once("hook:beforeDestroy")` de Vue 2
**Archivo:** `components/gov-chat-frontend/src/views/AnalyticsDashboard.vue`

El patrón `this.$once("hook:beforeDestroy", callback)` es exclusivo de Vue 2 y no existe en Vue 3, por lo que el interval del dashboard de analíticas nunca se limpiaba.

**Cambio aplicado:** Reemplazado por `beforeUnmount()` estándar de Vue 3, guardando la referencia del interval en la instancia del componente.

---

### Fix 10 — Clases CSS de animación con nombre de Vue 2
**Archivo:** `components/gov-chat-frontend/src/App.vue`

La clase de animación `.fade-enter` es sintaxis de Vue 2. En Vue 3 se renombró a `.fade-enter-from`.

**Cambio aplicado:** `.fade-enter` → `.fade-enter-from`.

---

## Soporte Multilingüe — Migración a bge-m3

### Contexto

El sistema usaba `BAAI/bge-base-en-v1.5` como modelo de embeddings, que genera vectores de 768 dimensiones y fue entrenado solo en inglés. Documentos en español producían embeddings de baja calidad, afectando directamente la relevancia de las respuestas del chatbot.

### Cambios implementados

#### Modelo de embeddings
**Archivos:** `docker-compose.yaml`, `genie-ai-overlay/retriever/config.py`, `.env`, `.env.example`

Se migró al modelo `BAAI/bge-m3`, que:
- Soporta más de 100 idiomas incluyendo español e inglés
- Genera vectores de **1024 dimensiones** (vs 768 del modelo anterior)
- Mantiene mejor rendimiento en búsqueda semántica cross-language

El volumen de Docker apunta a `./models/bge-m3` en lugar de `./models/bge-base-en-v1.5`.

#### OCR multilingüe
**Archivo:** `genie-ai-overlay/dataprep/genieai_dataprep_utils.py`

El OCR (EasyOCR) estaba hardcodeado a `['en']`. Ahora lee la variable de entorno `DATAPREP_OCR_LANGS` (valor por defecto: `"en,es"`), que acepta cualquier lista de idiomas separados por coma.

#### Validación de idioma de documentos
**Archivo:** `components/document-repository/src/services/fileService.js`

La validación comparaba el idioma detectado contra un único idioma requerido. Ahora parsea `DOCUMENT_INGESTION_LANGUAGE` como lista separada por comas (`"en,es"`) y acepta documentos en cualquiera de los idiomas listados.

#### Variables de entorno actualizadas
**Archivos:** `.env`, `.env.example`, `components/document-repository/env`

| Variable | Antes | Después |
|---|---|---|
| `EMBEDDING_MODEL_ID` | `BAAI/bge-base-en-v1.5` | `BAAI/bge-m3` |
| `DOCUMENT_INGESTION_LANGUAGE` | `es` | `en,es` |

### Nota importante sobre re-ingesta

El cambio de dimensiones de vectores (768 → 1024) hace que los embeddings existentes en ArangoDB sean incompatibles con el nuevo modelo. **Todos los documentos deben re-ingresarse** después del primer arranque exitoso con bge-m3 para que la búsqueda semántica funcione correctamente.

---

## Resumen de archivos modificados

| Archivo | Fase | Cambio |
|---|---|---|
| `genie-ai-overlay/http-service/app.py` | A | JWT_SECRET obligatorio por envvar |
| `components/gov-chat-backend/index.js` | A, B | CORS anchored regex; rate limiter reordenado; body parser 1mb |
| `components/gov-chat-backend/services/auth-service.js` | A | algorithms: ['HS256'] en 3 llamadas |
| `components/gov-chat-backend/routes/user-routes.js` | B | Magic bytes + sanitización de filename |
| `components/gov-chat-backend/routes/auth-routes.js` | B | Joi schemas en 3 endpoints |
| `ChatFolders.vue` | C | beforeDestroy → beforeUnmount |
| `ChatResponseFeedbackDialog.vue` | C | beforeDestroy → beforeUnmount |
| `ModalComponent.vue` | C | beforeDestroy → beforeUnmount |
| `PersonalIdentificationTab.vue` | C | beforeDestroy → beforeUnmount |
| `SearchableCountryDropdown.vue` | C | beforeDestroy → beforeUnmount |
| `UserProfileComponent.vue` | C | beforeDestroy → beforeUnmount |
| `AnalyticsDashboard.vue` | C | $once hook → beforeUnmount |
| `App.vue` | C | .fade-enter → .fade-enter-from |
| `genie-ai-overlay/dataprep/genieai_dataprep_utils.py` | Multilingüe | OCR configurable por envvar |
| `genie-ai-overlay/retriever/config.py` | Multilingüe | Default model → bge-m3 |
| `docker-compose.yaml` | Multilingüe | Volumen TEI → bge-m3 |
| `components/document-repository/src/services/fileService.js` | Multilingüe | Validación multi-idioma |
| `.env` / `.env.example` / `document-repository/env` | Multilingüe | Variables actualizadas |
