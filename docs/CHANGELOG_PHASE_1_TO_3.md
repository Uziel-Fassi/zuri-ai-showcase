# CHANGELOG — Phases 1 to 3
## GENIE.AI / ZURI AI MVP — Security, Resiliency & Quality Hardening

**Release Date:** 30 March 2026  
**Engineer:** GitHub Copilot (AI-assisted sessions)  
**Reviewed by:** Tech Lead  
**Scope:** Full-stack hardening across Node.js backend, Vue.js frontend, Python OPEA microservices, and Docker orchestration

---

## Resumen Ejecutivo

El sistema GENIE.AI/ZURI AI partió de un estado funcional pero frágil: secretos por defecto en producción, ausencia de rate limiting, workers que spawneaban sin control, timeouts sin definir, y microservicios internos sin autenticación. En tres fases de trabajo se transformó en un sistema seguro, resiliente ante OOM/timeouts, y preparado para pruebas reales con usuarios.

Las intervenciones más impactantes fueron: (1) la validación fatal del JWT Secret que impide arrancar el backend con credenciales por defecto, (2) el timeout global de 90 s en el pipeline ChatQnA que convierte stalls infinitos en errores HTTP 504 controlados, (3) la migración de workers ad-hoc a un pool persistente con Piscina que elimina OOM por thread-spawning, y (4) la autenticación interna HMAC entre todos los microservicios de la red Docker. Como extras no planificados, se integró un servicio STT (Speech-to-Text) con Whisper y se construyó la infraestructura base de evaluación RAG con RAGAS.

---

## Phase 1 — Security Hardening

### 1.1 Centralizar Secretos y Eliminar Credenciales Hardcodeadas

| Detalle | Valor |
|---|---|
| **Archivo principal** | `.env.example` (nuevo, 250+ líneas) |
| **Secretos centralizados** | `JWT_SECRET`, `SESSION_SECRET`, `INTERNAL_API_KEY`, `ARANGO_PASSWORD`, `TRANSLATION_CACHE_PASSWORD`, `POSTGRES_PASSWORD`, `TOKEN_SECRET`, `KC_BOOTSTRAP_ADMIN_PASSWORD`, `KC_DB_PASSWORD`, `HUGGINGFACEHUB_API_TOKEN`, `GEMINI_API_KEY`, `VLLM_API_KEY`, `EMAIL_PASSWORD` |
| **Método de generación** | Documentado inline con comandos `node -e "require('crypto').randomBytes(64).toString('base64')"` |
| **Credenciales hardcodeadas eliminadas** | Se removieron valores por defecto inseguros de `auth-service.js`, `index.js` y configs de Docker |

### 1.2 Fix Fatal en JWT Secret Inseguro

**Archivo:** `components/gov-chat-backend/services/auth-service.js` L13-18

El backend ahora ejecuta `process.exit(1)` si `JWT_SECRET` no está definido o si coincide con el valor por defecto `'your-secret-key-here-change-in-production'`. El servicio **no arranca** con secretos inseguros — fail-fast por diseño.

```js
if (!process.env.JWT_SECRET || process.env.JWT_SECRET === 'your-secret-key-here-change-in-production') {
  logger.error('[SECURITY] FATAL: JWT_SECRET is not set or uses an insecure default value.');
  process.exit(1);
}
```

### 1.3 Activar Rate Limiting

**Archivo:** `components/gov-chat-backend/middleware/rate-limit-middleware.js` (nuevo)

`express-rate-limit@^7.5.0` estaba instalado pero no se usaba. Se creó middleware dedicado y se aplicó en `index.js`:

| Limiter | Ventana | Máximo | Aplica a |
|---|---|---|---|
| `globalLimiter` | 1 min | 200 req/IP | Todas las rutas (excepto `/health`) |
| `authLimiter` | 15 min | 10 req/IP | `/api/auth/login` |
| `registerLimiter` | 1 hora | 5 req/IP | `/api/auth/register` |

Headers: `draft-7` standard. Skip automático en health checks.

### 1.4 Corregir Path Traversal en Dataprep

**Archivo:** `genie-ai-overlay/dataprep/genieai_dataprep_microservice.py` L107-136

Se implementó `sanitize_filename()`:
- `os.path.basename()` para eliminar componentes de directorio (`../../../etc/passwd` → `passwd`)
- Regex `^[a-zA-Z0-9._-]+$` para whitelist estricta de caracteres
- Validación de extensión contra `ALLOWED_EXTENSIONS = {'.pdf', '.docx', '.txt', '.md', '.html', '.xlsx', '.pptx'}`
- Rechazo con HTTP 400 y mensaje explícito

### 1.5 Autenticación Interna a Microservicios Expuestos

**Archivos:** `genieai_retriever_microservice.py`, `genieai_dataprep_microservice.py`, `genieai_chatqna.py`

Se implementó `InternalAPIKeyMiddleware` (Starlette `BaseHTTPMiddleware`) en retriever y dataprep:
- Valida header `X-Internal-API-Key` en todas las rutas excepto health checks
- Comparación en tiempo constante con `hmac.compare_digest()` para prevenir timing attacks
- El key se inyecta automáticamente en todas las llamadas salientes desde ChatQnA mediante monkey-patch de `aiohttp.ClientSession` al arranque

---

## Phase 2 — Resiliency & Stability

### 2.1 Timeout Global del Pipeline (90 s)

**Archivo:** `genie-ai-overlay/chatqna/genieai_chatqna.py` L95, L887-901

El método `handle_request()` envuelve toda la ejecución del pipeline (embedding → retrieval → reranking → LLM) con `asyncio.wait_for(timeout=PIPELINE_TIMEOUT_SEC)`. Configurable vía env var, default 90 s. Al expirar, devuelve HTTP 504 con mensaje localizado al usuario en lugar de colgar indefinidamente.

Adicionalmente, se configuró LiteLLM (proxy de Gemini) con:
- `LITELLM_NUM_RETRIES=0` — evita reintentos internos que consumían el timeout completo
- `LITELLM_REQUEST_TIMEOUT=30` — fail-fast individual por llamada a la API de Google

### 2.2 Timeout del Labeler + Warm-up del Modelo

**Archivo:** `genie-ai-overlay/chatqna/intent_classifier.py`

- `LABELER_TIMEOUT_MS`: elevado de 500 ms a **2000 ms** vía override en `docker-compose.yaml`
- Clasificación de intenciones envuelta en `asyncio.wait_for()` con fallback graceful a `IntentLabel.ACADEMIC` (confidence 0.0) si expira
- `get_classifier()` implementa lazy singleton con double-checked locking para cargar DistilBERT con device auto-detection (CPU/CUDA)

### 2.3 Timeout en Frontend + UX de Error Localizada

**Archivos:** `chatbotService.js`, `ChatBotComponent.vue`, 11 archivos de locale

- Timeout de axios configurado a **95 000 ms** (5 s de gracia sobre el pipeline de 90 s)
- Detección de `error.code === 'ECONNABORTED'` para diferenciar timeouts de errores genéricos
- Clave de traducción `chatbot.timeoutError` implementada en **11 idiomas**: en, es, fr, de, ar, pt, ru, zh, sw, th, id

### 2.4 Resource Limits en Docker Compose (Anti-OOM)

**Archivos:** `docker-compose.yaml`, `docker-compose-t4.yaml`, `docker-compose-RTX6000-ADA.yaml`

Se aplicaron `deploy.resources.limits` a **todos** los servicios en los 3 archivos de compose. Selección representativa:

| Servicio | Memoria | CPUs |
|---|---|---|
| `chatqna-backend-server` | 2 GB | 2.0 |
| `arango-vector-db` | 5 GB | 1.5 |
| `tei-reranker` | 4 GB | 1.5 |
| `stt-service` | 4 GB | 2.0 |
| `backend` | 512 MB | 0.5 |
| `embedding` | 512 MB | 0.5 |
| `redis-cache` | 256 MB | 0.25 |
| `frontend` | 256 MB | 0.25 |

Se eliminó la directiva obsoleta `version: '3.8'` de `docker-compose.yaml`.

### 2.5 Health Checks Completados

**Archivos:** `docker-compose.yaml`, `docker-compose-t4.yaml`, `docker-compose-RTX6000-ADA.yaml`

Se agregaron bloques `healthcheck` a todos los servicios con los endpoints correctos:

| Servicio | Endpoint | intervalo | start_period |
|---|---|---|---|
| `backend` | `/api/health` | 10 s | 15 s |
| `retriever` | `/v1/health_check` | 10 s | 15 s |
| `dataprep` | `/v1/health_check` | 15 s | 30 s |
| `chatqna` | `/v1/health_check` | 10 s | 30 s |
| `stt-service` | `/health` | 15 s | 30 s |

> **Nota sobre Image Pinning:** Se intentó fijar versiones de imágenes Docker (e.g., `opea/embedding:v1.3`, `litellm:main-v1.82.1`). Las imágenes con tags específicos **no existían** en los registros públicos. Se revirtieron todas las imágenes a `:latest` para restaurar la funcionalidad. Ver sección de Deuda Técnica.

### 2.6 Worker Thread Pool (Piscina)

**Archivos:** `services/query-service.js`, `services/opea-worker.js`, `package.json`

**Antes:** Cada query a OPEA ejecutaba `new Worker(workerPath)` (dead code que nunca se invocaba) o llamaba a `axios.post` directamente en el event loop principal — bloqueando el thread principal bajo carga.

**Después:** Se instaló `piscina@^5.1.4` y se reescribió la arquitectura:

- **Pool singleton** en `query-service.js`: `new Piscina({ maxThreads: WORKER_POOL_SIZE || 4, minThreads: 1 })`
- **Worker module** en `opea-worker.js`: `module.exports = async ({ url, payload }) => { ... }` — función pura con axios keepAlive agents (maxSockets: 100) y timeout de 120 s
- Se eliminó el método dead `runOPEAWorker()` (~25 líneas de código muerto)
- `WORKER_POOL_SIZE` agregado a las variables de entorno del servicio backend en los 3 archivos de compose (default: 4)

---

## Phase 3 — Code Quality & Cleanup

### 3.1 Eliminar Debug Logging de Producción

**Archivos:** `vue.config.js`, `ChatBotComponent.vue`, `index.js` (backend)

**Estrategia:** En lugar de envolver 122 `console.log` individualmente en checks de `NODE_ENV`, se configuró `terser-webpack-plugin` con `pure_funcs: ['console.log']` en `vue.config.js`. En builds de producción, todas las llamadas a `console.log` se eliminan automáticamente del bundle. `console.error` y `console.warn` de manejo de errores se conservan intactos.

Cambios específicos:
- **`vue.config.js`**: Eliminado bloque de 9 `console.log` de debug (`"!!! VUE.CONFIG.JS IS LOADING..."`, env vars, CSP string). Agregado bloque `optimization.minimizer` con terser
- **`ChatBotComponent.vue`**: 1 `console.warn` con prefijo `[DEBUG]` convertido a `console.log` para que terser lo elimine
- **`index.js` (backend)**: Eliminado `console.log("allowlist:" + allowlist)` que exponía la configuración CORS

### 3.2 Centralizar AuthTokenManager (DRY Python)

**Archivos:** Nuevo `genie-ai-overlay/core/auth_token_manager.py`, modificados `genieai_chatqna.py` y `genieai_dataprep_arangodb.py`

El patrón `_get_auth_token()` con double-checked locking via `asyncio.Lock()` y caché de 50 minutos estaba duplicado textualmente en dos archivos (35 líneas cada uno). Se extrajo a una clase reutilizable:

```python
class AuthTokenManager:
    def __init__(self, token_url: str): ...
    async def get_token(self): ...  # Fast path sin lock → slow path con double-checked locking
```

Ambos consumidores ahora delegan con `self._auth = AuthTokenManager(GET_AUTH_TOKEN_URL)` y `_get_auth_token()` es una línea: `return await self._auth.get_token()`. ~70 líneas de código duplicado eliminadas.

### 3.3 CORS Fix + Validación de Inputs en Node.js

Tres correcciones de seguridad en el backend:

**(1) CORS — Bloqueo de requests sin Origin en producción**

**Archivo:** `components/gov-chat-backend/index.js`

El callback CORS tenía `if (!origin) return callback(null, true)` — permitía cualquier request sin header `Origin` (curl, scripts, etc.). Ahora se evalúa `NODE_ENV`:
- **Producción:** Rechaza con error explícito `"Origin header is required in production"`
- **Desarrollo:** Permite (para Postman, curl, test tools)

**(2) Validación de `queryId` en rutas PATCH/GET/POST**

**Archivo:** `components/gov-chat-backend/routes/query-routes.js`

Se agregó validación regex `/^[a-zA-Z0-9_-]{1,64}$/` en **4 rutas** que reciben `:queryId` como parámetro de ruta:
- `PATCH /:queryId/responsetime`
- `PATCH /:queryId/answered`
- `GET /:queryId`
- `POST /:queryId/feedback`

Rechazo con HTTP 400 si el formato no cumple.

**(3) File upload MIME type validation**

**Archivo:** `components/gov-chat-backend/routes/user-routes.js`

Se agregó `fileFilter` a la configuración de `multer`:
- MIME types permitidos: `application/pdf`, `image/jpeg`, `image/png`
- Prefijo permitido: `application/vnd.openxmlformats-officedocument.*` (docx, xlsx, pptx)
- Cualquier otro tipo es rechazado con error

### 3.4 Refactorizar AdminDashboard.vue — NO IMPLEMENTADO

Pospuesto por decisión explícita. El componente tiene ~3300 LOC y 3 instancias de `$forceUpdate`. Refactorizarlo implica riesgo alto de regresión sin test suite. Se documenta en la sección de Deuda Técnica.

### 3.5 Plan de Migración Vue 2 → Vue 3 (Solo Análisis)

**Archivo generado:** `components/gov-chat-frontend/MIGRATION-VUE3-PLAN.md`

Análisis completo de migración que incluye:
- Inventario de 52 componentes con clasificación de severidad
- 3 bugs activos en producción identificados (render functions Vue 2, `$once` eliminado, CSS de transición)
- 7 instancias de `beforeDestroy` sin renombrar (memory leaks)
- Mapa de 25 usos de event bus global
- Roadmap de 4 sprints con criterios de éxito
- Checklist de Sprint 1 ejecutable en 1 día

**Estado:** Documento de diseño completado. Ejecución pendiente.

---

## Implementaciones Extra (Bonus)

### STT — Servicio de Speech-to-Text

**Archivos:** `stt_service.py`, `Dockerfile.stt`, `docker-compose.yaml` (servicio `stt-service`), `ChatBotComponent.vue`

Se integró un servicio de transcripción de voz completo:
- **Backend:** Microservicio FastAPI con `faster-whisper`, modelo `base`, optimizado para CPU (`int8`)
- **Docker:** Contenedor dedicado en puerto 8005, 4 GB RAM, 2 CPUs, con health check en `/health`
- **Frontend:** Botón de micrófono en `ChatBotComponent.vue` con grabación via `MediaRecorder API`, envío como `FormData`, y transcripción automática al `textarea` del chat
- **Configuración:** Idioma configurable via `WHISPER_LANGUAGE` (default: `es`), modelo via `WHISPER_MODEL_SIZE`

### RAGAS — Infraestructura de Evaluación RAG

**Archivos:** `evaluate_rag.py`, `test_dataset.csv`

Se construyó la infraestructura base para evaluación automatizada del pipeline RAG:
- Script de ~340 líneas usando `ragas 0.4.x` con métricas: `Faithfulness`, `AnswerRelevancy`, `ContextPrecision`
- Google Gemini configurado como LLM juez via `InstructorLLM`
- Dataset de prueba con preguntas, respuestas esperadas y contexto de referencia
- Output a `rag_evaluation_results.csv` para análisis iterativo

**Estado:** Infraestructura creada. La primera ejecución requiere refinamiento de configuración (API keys, tuning de prompts del juez). Ver Deuda Técnica.

---

## Deuda Técnica y Próximos Pasos

### Prioridad Alta

| Item | Contexto | Acción requerida |
|---|---|---|
| **Image Pinning (2.5)** | Se intentó anclar imágenes Docker a versiones específicas. Los tags `opea/embedding:v1.3`, `litellm:main-v1.82.1`, entre otros, **no existen** en Docker Hub ni en los registros de OPEA/BerriAI. Se revirtieron todos a `:latest` para restaurar `docker compose up`. | Contactar a los maintainers de OPEA y BerriAI/LiteLLM para obtener los tags correctos publicados. Alternativamente, hacer build local de las imágenes y pushear a un registry privado con version tags propios. |
| **Migración Vue 3 — Sprint 1** | El análisis (`MIGRATION-VUE3-PLAN.md`) identificó 3 bugs activos: render functions de Vue 2 que no renderizan, `$once` que causa memory leak en `AnalyticsDashboard.vue`, y CSS de transición `.fade-enter` que no aplica. Además, 7 instancias de `beforeDestroy` que no se ejecutan (memory leaks). | Ejecutar Sprint 1 del plan de migración (estimado: 1 día). Es el sprint con mayor ROI: corrige todos los bugs activos y memory leaks con cambios mínimos. |
| **RAGAS Tuning** | El script `evaluate_rag.py` está funcional pero la primera ejecución falló por configuración de API keys y formato de respuesta del juez LLM. | Configurar `GEMINI_API_KEY` válido, verificar formato de `test_dataset.csv`, y ejecutar iteraciones hasta obtener scores baseline reproducibles. |

### Prioridad Media

| Item | Contexto | Acción requerida |
|---|---|---|
| **AdminDashboard.vue (3.4)** | Componente de ~3300 LOC con 3 `$forceUpdate()` anti-pattern. No se refactorizó por riesgo de regresión sin tests. | Escribir tests E2E para el dashboard antes de refactorizar. Luego migrar a Composition API con estado reactivo (`ref()`/`reactive()`) que elimine la necesidad de `$forceUpdate`. |
| **Vuex → Pinia (Sprint 3)** | `vuex@^4.0.2` funciona con Vue 3 pero Pinia es el gestor de estado oficial recomendado. | Seguir Sprint 3 del plan de migración. |
| **axios v0 → v1** | `axios@^0.27.2` en el frontend es pre-v1 con breaking changes en manejo de errores. No es un riesgo de seguridad directo pero es deuda técnica. | Actualizar a `axios@^1.x` y ajustar interceptors en `httpService.js`. |

### Prioridad Baja

| Item | Contexto | Acción requerida |
|---|---|---|
| **Test Suite** | `@vue/test-utils` no está instalado. `tests/` solo contiene `.gitkeep`. | Instalar `@vue/test-utils@^2`, configurar Vitest, priorizar tests de `ChatBotComponent.vue` y `AdminDashboard.vue`. |
| **Options API → Composition API (Sprint 4)** | Los 52 componentes usan Options API. Vue 3 lo soporta indefinidamente — no es bloqueador. | Migrar los 5 componentes críticos a `<script setup>` según Sprint 4 del plan. |
| **Router legacy** | `src/router/index.js` (no importado) referencia views que no existen. | Eliminar el archivo. |

---

## Mapa de Archivos Modificados

```
genie-ai-replica-MVP-ready/
├── .env.example                                    [1.1] Secretos centralizados
├── docker-compose.yaml                             [2.4][2.5][2.6] Limits, health checks, WORKER_POOL_SIZE
├── docker-compose-t4.yaml                          [2.4][2.5][2.6] Limits, health checks, WORKER_POOL_SIZE
├── docker-compose-RTX6000-ADA.yaml                 [2.4][2.5][2.6] Limits, health checks, WORKER_POOL_SIZE
├── stt_service.py                                  [EXTRA] Servicio STT
├── Dockerfile.stt                                  [EXTRA] Build del servicio STT
├── evaluate_rag.py                                 [EXTRA] Framework de evaluación RAGAS
├── test_dataset.csv                                [EXTRA] Dataset de prueba RAGAS
│
├── components/gov-chat-backend/
│   ├── index.js                                    [1.3][3.1][3.3] Rate limiting, CORS fix, debug log removido
│   ├── package.json                                [2.6] piscina@^5.1.4
│   ├── middleware/
│   │   └── rate-limit-middleware.js                [1.3] NUEVO — Rate limiters
│   ├── services/
│   │   ├── auth-service.js                         [1.2] JWT fatal validation
│   │   ├── query-service.js                        [2.6] Piscina pool singleton
│   │   └── opea-worker.js                          [2.6] Reescrito para Piscina (module.exports)
│   └── routes/
│       ├── query-routes.js                         [3.3] queryId regex validation
│       └── user-routes.js                          [3.3] multer fileFilter MIME validation
│
├── components/gov-chat-frontend/
│   ├── vue.config.js                               [3.1] Debug logs eliminados, terser pure_funcs
│   ├── MIGRATION-VUE3-PLAN.md                      [3.5] NUEVO — Plan de migración Vue 3
│   └── src/
│       ├── components/ChatBotComponent.vue          [2.3][EXTRA] Timeout UX, STT integration
│       ├── services/chatbotService.js               [2.3] Timeout 95s
│       └── i18n/locales/{en,es,fr,de,ar,pt,ru,zh,sw,th,id}.js  [2.3] timeoutError key
│
├── genie-ai-overlay/
│   ├── core/
│   │   └── auth_token_manager.py                   [3.2] NUEVO — AuthTokenManager centralizado
│   ├── chatqna/
│   │   ├── genieai_chatqna.py                      [1.5][2.1][3.2] Internal auth, pipeline timeout, AuthTokenManager
│   │   └── intent_classifier.py                    [2.2] Labeler timeout 2000ms, warm-up singleton
│   ├── dataprep/
│   │   ├── genieai_dataprep_microservice.py         [1.4][1.5] Path traversal fix, InternalAPIKeyMiddleware
│   │   └── genieai_dataprep_arangodb.py             [3.2] AuthTokenManager delegación
│   └── retriever/
│       └── genieai_retriever_microservice.py        [1.5] InternalAPIKeyMiddleware
```

---

## Resumen de Cobertura vs. Plan Original

| Tarea | Estado | Notas |
|---|---|---|
| 1.1 Centralizar Secretos | ✅ Completado | `.env.example` con 13 secretos + comandos de generación |
| 1.2 JWT Fatal Check | ✅ Completado | `process.exit(1)` si secret inseguro |
| 1.3 Rate Limiting | ✅ Completado | Global (200/min) + Auth (10/15min) + Register (5/h) |
| 1.4 Path Traversal | ✅ Completado | `sanitize_filename()` con regex whitelist |
| 1.5 Auth Interna | ✅ Completado | HMAC middleware + injection automática via aiohttp |
| 2.1 Pipeline Timeout | ✅ Completado | 90 s con HTTP 504 + LiteLLM tuning |
| 2.2 Labeler Timeout | ✅ Completado | 2000 ms + fallback graceful + warm-up singleton |
| 2.3 Frontend Timeout UX | ✅ Completado | 95 s + mensajes en 11 idiomas |
| 2.4 Docker Limits | ✅ Completado | Todos los servicios en 3 archivos compose |
| 2.5 Health Checks | ✅ Parcial | Health checks implementados; image pinning revertido |
| 2.6 Worker Pool | ✅ Completado | Piscina con pool persistente (4 threads default) |
| 3.1 Debug Logging | ✅ Completado | Terser `pure_funcs` + remoción manual de logs backend |
| 3.2 AuthTokenManager | ✅ Completado | Clase DRY con double-checked locking |
| 3.3 CORS + Validation | ✅ Completado | Production CORS + queryId regex + MIME filter |
| 3.4 AdminDashboard | ⏸️ Pospuesto | Riesgo de regresión sin tests |
| 3.5 Vue 3 Migration Plan | ✅ Diseño completo | Ejecución pendiente (4 sprints planificados) |
| STT Integration | ✅ Bonus | faster-whisper + frontend UI completa |
| RAGAS Setup | ✅ Parcial (Bonus) | Infraestructura lista, tuning pendiente |

---


Archivos modificados (7 archivos)
Archivo	Cambio
.env	DOCUMENT_INGESTION_LANGUAGE=en,es + EMBEDDING_MODEL_ID=BAAI/bge-m3
.env.example	DOCUMENT_INGESTION_LANGUAGE=en,es
document-repository/env	DOCUMENT_INGESTION_LANGUAGE=en,es
fileService.js	Validación ahora acepta lista de idiomas separados por coma
genieai_dataprep_utils.py	OCR configurable via DATAPREP_OCR_LANGS=en,es
retriever/config.py	Defaults actualizados a BAAI/bge-m3
docker-compose.yaml	Volumen TEI apunta a ./models/bge-m3

*Documento generado como parte del proceso de hardening MVP → producción. Última actualización: 30 de marzo de 2026.*
