# Informe de Cambios y Decisiones de Arquitectura
## Proyecto ZURI AI — Adaptación de GENIE.AI para UPB

**Autor:** Uziel Fassi
**Stack base:** GENIE.AI / OPEA (Open Platform for Enterprise AI) — Intel  
**Stack destino:** ZURI AI — Universidad Privada Boliviana (UPB)  
**Fecha:** Marzo 2026

---

## Resumen Ejecutivo

GENIE.AI es un framework empresarial de código abierto desarrollado por Intel/UNICC para desplegar chatbots gubernamentales basados en RAG (*Retrieval-Augmented Generation*). Este proyecto adapta dicho framework en **ZURI AI**, un asistente académico para la UPB, implementando cambios profundos en todas las capas del stack: infraestructura, backend, base de datos, pipeline de IA y frontend.

---

## Arquitectura de Referencia (Original GENIE.AI)

```
Usuario → Kong API Gateway
              ↓
    gov-chat-frontend (Vue 2)
              ↓
    gov-chat-backend (Node.js/Express)
              ↓
    ChatQnA Pipeline (OPEA / Python)
     ├── TEI Embedding (GPU, nvidia runtime)
     ├── TEI Reranker  (GPU, nvidia runtime)
     ├── Retriever     (ArangoDB graph query)
     └── LLM (opea/chatqna:latest → Megaservice)
              ↓
    ArangoDB (graph database)
```

---

## 1. Infraestructura y DevOps

### 1.1 Eliminación de dependencia GPU

**Problema:** El despliegue original requería `runtime: nvidia` y GPUs físicas para dos servicios TEI (embeddings y reranker), lo que bloqueaba el desarrollo y CI en máquinas sin GPU.

**Decisión:** Migrar ambos servicios a imágenes CPU-only.

**Cambios en `docker-compose.yaml`:**
```yaml
# Antes
image: ghcr.io/huggingface/text-embeddings-inference:1.5
runtime: nvidia
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]

# Después
image: ghcr.io/huggingface/text-embeddings-inference:cpu-1.5
# (sin runtime: nvidia, sin deploy.resources)
```

**Impacto:** El stack completo corre en cualquier máquina x86 sin hardware especializado. Rendimiento de inferencia reducido, pero aceptable para un MVP académico con carga moderada.

---

### 1.2 Builds personalizados vs imágenes upstream

**Problema:** Las imágenes `opea/chatqna:latest` y `opea/reranking:latest` no soportaban las customizaciones requeridas (Gemini como LLM, lógica propia de reranking, correcciones de protocolo).

**Decisión:** Construir imágenes propias a partir de Dockerfiles locales.

**Cambios en `docker-compose.yaml`:**
```yaml
# ChatQnA — antes
image: opea/chatqna:latest

# ChatQnA — después
build:
  context: .
  dockerfile: genie-ai-overlay/chatqna/Dockerfile-chatqna_genie-ai
image: genie-ai-chatqna

# Reranker — antes
image: opea/reranking:latest

# Reranker — después
build:
  context: .
  dockerfile: genie-ai-overlay/reranker/Dockerfile-reranker_genie-ai
image: genie-ai-reranker
restart: on-failure:5
environment:
  RERANKING_STRATEGY: slice
```

---

### 1.3 Nuevo servicio: Gemini Adapter Wrapper

**Problema:** El pipeline ChatQnA de OPEA espera un backend LLM compatible con la API de OpenAI. Google Gemini expone una API propia incompatible con este contrato.

**Decisión:** Crear un microservicio proxy (`gemini-adapter-wrapper`) que actúa como intermediario: recibe peticiones en formato OpenAI y las retransmite a Gemini vía LiteLLM.

**Archivos nuevos:**
- `genie-ai-overlay/gemini-adapter-wrapper/openai_compatible_server.py` — Servidor FastAPI/uvicorn
- `genie-ai-overlay/gemini-adapter-wrapper/Dockerfile`
- `genie-ai-overlay/gemini-adapter-wrapper/requirements.txt`

**Flujo:**
```
ChatQnA Pipeline → [OpenAI format POST] → gemini-adapter-wrapper
                                              ↓
                                   LiteLLM → Google Gemini API
                                              ↓
                              [OpenAI format response] → ChatQnA
```

**Variables de entorno clave:** `GEMINI_API_KEY`, `GEMINI_MODEL` (default: `gemini/gemini-flash-latest`), `LITELLM_HOST`.

---

### 1.4 Volume mount para configuración frontend

**Problema:** Cualquier cambio en logos, colores o textos del frontend (`public/config/`) requería un rebuild completo del contenedor (20+ minutos).

**Decisión:** Agregar un bind mount para ese directorio.

**Cambio en `docker-compose.yaml`:**
```yaml
gov-chat-frontend:
  volumes:
    - ./components/gov-chat-frontend/public/config:/app/dist/config
```

**Impacto:** Cambios de branding son instantáneos sin rebuild. El contenedor siempre sirve los archivos del host en tiempo real.

---

### 1.5 Corrección de nombre de grafo en Retriever

**Problema:** El servicio Retriever no encontraba el grafo ArangoDB porque el nombre configurado no coincidía con el grafo real.

**Cambio:**
```yaml
# Antes
RETRIEVER_ARANGO_GRAPH_NAME: GRAPH_SOURCE

# Después
RETRIEVER_ARANGO_GRAPH_NAME: GRAPH
```

---

### 1.6 Eliminación de dependencia ClamAV

El servicio `document-repository` dependía de ClamAV para escaneo antivirus de documentos subidos. En el contexto MVP esta dependencia causaba fallos de arranque y no era requerida.

**Cambio:** Removida la dependencia `clamav` del `docker-compose.yaml`. Agregada variable `NODE_PATH=/app/node_modules` para resolución correcta de módulos.

---

## 2. Pipeline de IA (ChatQnA / OPEA)

### 2.1 Separación de `LLM_MODEL` y `TOKENIZER_MODEL_ID`

**Problema:** El servicio ChatQnA llamaba `AutoTokenizer.from_pretrained(LLM_MODEL)` para tokenizar. Los IDs de modelos Gemini (ej. `gemini/gemini-flash-latest`) no son repositorios de HuggingFace — la llamada lanzaba una excepción fatal al arrancar el servicio.

**Decisión:** Introducir la variable de entorno `TOKENIZER_MODEL_ID`, independiente de `LLM_MODEL`, que apunta a un modelo HuggingFace válido para tokenización local.

**Cambio en `genieai_chatqna.py`:**
```python
# Antes
TOKENIZER_MODEL_ID = os.getenv("LLM_MODEL", "Intel/neural-chat-7b-v3-3")

# Después
LLM_MODEL = os.getenv("LLM_MODEL", "gemini/gemini-flash-latest")
TOKENIZER_MODEL_ID = os.getenv("TOKENIZER_MODEL_ID", "bert-base-uncased")
```

---

### 2.2 Corrección del pipeline de Reranking

**Problema:** El módulo de reranking enviaba los campos con nombres incorrectos y apuntaba al endpoint equivocado, causando que el reranker rechazara todas las peticiones con error 422.

**Cambios en `genieai_chatqna.py`:**

| Campo / Ruta | Original | Corregido |
|---|---|---|
| Endpoint HTTP | `/rerank` | `/v1/reranking` |
| Campo query | `query` | `initial_query` |
| Campo documentos | `documents` | `retrieved_docs` |
| Respuesta | Asumía `list` raw | Maneja `RerankingResponse` dict + fallback a `list` |

---

### 2.3 Endurecimiento del parsing de respuesta LLM

**Problema:** Cuando Gemini devolvía una respuesta de error (ej. rate limit, safety filter), el payload no contenía el campo `choices` que espera el parser de OPEA, causando un `KeyError` que cortaba la conversación sin mensaje de error útil.

**Cambio:** Añadida lógica de fallback en el parser:
```python
# Extrae texto con manejo de error shape
if "choices" in response_data:
    answer = response_data["choices"][0]["message"]["content"]
elif "error" in response_data:
    answer = f"[LLM Error] {response_data['error'].get('message', 'unknown')}"
else:
    answer = str(response_data)
```

---

### 2.4 Re-implementación del módulo Labeler (Clasificador de Intenciones)

El pipeline original de GENIE.AI incluía un paso de clasificación/etiquetado automático de consultas que fue eliminado inicialmente por latencia y dependencias GPU. Se reimplementó con un enfoque ligero adaptado al contexto académico UPB.

**Modelo:** `typeform/distilbert-base-uncased-mnli` (~260 MB) — clasificación zero-shot usando NLI (Natural Language Inference). No requiere fine-tuning ni datos de entrenamiento propios.

**Categorías definidas:**

| Intención | Ruta | Descripción |
|---|---|---|
| `ACADEMIC` | Pipeline RAG completo (embedding → retriever → rerank → LLM) | Preguntas sobre cursos, notas, profesores, tesis, programas |
| `ADMINISTRATIVE` | Respuesta predefinida (bypass RAG) | Pagos, horarios, certificados, trámites institucionales |
| `GENERAL` | Pipeline RAG estándar (fallback seguro) | Conversación casual, saludos |

**Archivos nuevos:**
- `genie-ai-overlay/chatqna/intent_classifier.py` — Módulo standalone con API sync/async

**Decisiones de diseño:**

1. **Timeout de 500ms:** Si la clasificación excede este umbral, el sistema asume `ACADEMIC` para no bloquear al usuario.
2. **Detección automática de hardware:** `torch.cuda.is_available()` para usar GPU NVIDIA si existe, con fallback transparente a CPU.
3. **Umbral de confianza (0.45):** Si la confianza del clasificador es menor, se rutea a `ACADEMIC` como ruta segura por defecto.
4. **Pre-descarga en `entrypoint.sh`:** El modelo se descarga al iniciar el contenedor para que la primera request del usuario sea rápida.
5. **Toggle por variable de entorno:** `LABELER_ENABLED=false` desactiva completamente el clasificador sin cambiar código.

**Integración en `genieai_chatqna.py`:**
```python
# Después de extraer el último mensaje traducido, antes de megaservice.schedule()
intent_label, intent_confidence = await classify_intent(last_translated_message_content)

if intent_label == IntentLabel.ADMINISTRATIVE:
    return {"response": admin_response, "metadata": {..., "intent": "ADMINISTRATIVE"}}

# ACADEMIC y GENERAL continúan al pipeline RAG
result_dict, runtime_graph = await self.megaservice.schedule(...)
```

**Variables de entorno:**
- `LABELER_ENABLED` (default: `true`)
- `LABELER_MODEL_ID` (default: `typeform/distilbert-base-uncased-mnli`)
- `LABELER_TIMEOUT_MS` (default: `500`)
- `LABELER_CONFIDENCE_THRESHOLD` (default: `0.45`)

**Dependencias añadidas al Dockerfile:**
- `torch` (CPU-only via `--index-url https://download.pytorch.org/whl/cpu`)

---

### 2.5 Corrección de directorio de trabajo en entrypoint

**Cambio en `entrypoint.sh`:**
```bash
# Añadido al inicio
cd /app/ChatQnA || exit 1
```

Garantiza que el proceso de Python se ejecuta desde el directorio correcto, evitando imports relativos fallidos.

---

## 3. Base de Datos (ArangoDB)

### 3.1 Corrección crítica: colecciones de aristas mal tipadas

**Problema raíz:** Las colecciones `userConversations`, `userFolders` y `folderConversations` fueron creadas como **colecciones de documentos** (`type=2`) en lugar de **colecciones de aristas** (`type=3`). ArangoDB silenciosamente elimina los campos `_from` y `_to` de los documentos insertados en colecciones de tipo incorrecto. Los 206 documentos de enlace existentes no tenían referencia a usuario ni a conversación — por eso los chats guardados nunca aparecían en la interfaz.

**Diagnóstico:** Ejecutado en `arangosh`:
```javascript
db.userConversations.type() // retornó 2 (document) — debía ser 3 (edge)
db.userConversations.count() // retornó 206 documentos sin _from/_to
```

**Fix directo en base de datos:**
```javascript
db.userConversations.drop()
db._createEdgeCollection("userConversations")
// repetido para userFolders, folderConversations
```

**Fix en `arango-init.js`** (previene la regresión):
```javascript
// Antes
db._create(collectionName)

// Después — colecciones de aristas se crean explícitamente
const edgeCollections = [
  "userConversations", "userFolders", "folderConversations",
  "conversationCategories", "queryMessages", "sessionQueries"
];

if (edgeCollections.includes(collectionName)) {
  db._createEdgeCollection(collectionName);
} else {
  db._create(collectionName);
}
```

---

### 3.2 Corrección de consulta AQL — null-safety

**Problema:** La consulta que carga conversaciones del usuario filtraba `conversation.isArchived == false`. En ArangoDB, documentos sin el campo `isArchived` retornan `null`, y `null == false` evalúa a `false` — causando que conversaciones nuevas (sin ese campo) no aparecieran.

**Cambio en `chat-history-service.js`:**
```javascript
// Antes
FILTER conversation.isArchived == false

// Después
FILTER conversation != null AND conversation.isArchived != true
```

El cambio se aplicó tanto a la query principal como a la query de conteo en `getUserConversations()`.

---

## 4. Backend (Node.js / Express)

### 4.1 Soporte de esquema de respuesta nativo ChatQnA

**Problema:** El `query-service.js` esperaba `data.text` como campo de respuesta del pipeline, pero el ChatQnA customizado retorna `data.response` (esquema nativo OPEA).

**Cambio en `query-service.js`:**
```javascript
// Soporte de ambos esquemas con prioridad al nativo
const answer = data.response ?? data.text ?? data.answer ?? "No response";
```

---

### 4.2 Toggle de habilitación del servicio de traducción

**Problema:** El servicio de traducción requería cargar modelos de NLP pesados al arrancar. En entornos donde no se necesitaba traducción, esto alargaba el tiempo de inicio y consumía RAM innecesariamente.

**Cambio en `translation-service.js`:**
```javascript
async init() {
  const enabled = process.env.ENABLE_TRANSLATION_SERVICE === 'true'
               || process.env.ENABLE_TRANSLATION === 'true';
  if (!enabled) {
    logger.info("Translation service disabled — skipping model load");
    return;
  }
  // ... carga de modelos
}
```

---

### 4.3 Rutas de servicios públicas (sin autenticación)

**Problema:** El endpoint que lista las categorías de servicios disponibles tenía `authMiddleware.authenticate` aplicado. El frontend necesita este listado antes del login para construir los botones de acceso rápido — causaba un error 401 en la pantalla inicial.

**Cambio en `service-routes.js`:**
```javascript
// Antes
router.get('/services', authMiddleware.authenticate, serviceController.list)

// Después
router.get('/services', serviceController.list)
```

---

## 5. Frontend (Vue 2)

### 5.1 Re-branding completo: GENIE.AI → ZURI AI

**Archivos de configuración:**

| Archivo | Cambio |
|---|---|
| `public/config/genie-ai-config.json` | Título `"ZURI AI - UPB Academic Support"`, colores `#C9A027` (dorado) / `#0d1b2e` (azul marino), welcome message en español/inglés para UPB |
| `src/theme-variables.css` | Paleta completa redefinida: `--accent-color: #C9A027`, `--bg-navbar: linear-gradient(#0d1b2e, #1a3055)`, modo oscuro adaptado |
| `public/config/logo_zuri.png` | Logo principal ZURI (reemplaza `genie-ai.svg`) |
| `public/config/logo_zuri_short.png` | Ícono compacto para navbar y widget |
| `public/config/logo_upb.png` | Logo institucional UPB |
| `public/config/splash.png` | Imagen de splash personalizada |

SVGs del branding GENIE.AI originales fueron eliminados: `genie-ai-icon.svg`, `genie-ai-icon-dark.svg`, `genie-ai-icon-light.svg`, `genie-ai.svg`, `splash.svg`.

---

### 5.2 Nuevo componente: FloatingChatWidget.vue

**Propósito:** Widget de chat inyectable en portales externos (portales universitarios, sistemas académicos). Permite integrar ZURI AI en cualquier página sin modificar su arquitectura — un patrón de integración no-invasivo.

**Características implementadas:**
- **Botón de apertura:** Círculo blanco con borde dorado (`#C9A027`), muestra `logo_zuri_short.png`. Al hover aparece un tooltip animado ("Chat" / "Cerrar") con transición `opacity + translateY`.
- **Panel de chat:** Desplegable con header branded, área de mensajes con scroll, input de texto.
- **Efecto typewriter:** Respuestas del bot se muestran carácter a carácter usando `splice()` para respetar la reactividad de Vue 2 (la asignación directa por índice de array no es reactiva en Vue 2).
- **Animación de carga:** Burbuja con tres puntos (`.dot-pulse`) durante la espera de respuesta — reemplaza el overlay bloqueante original.
- **Layout:** `.floating-chat-widget` usa `display: flex; flex-direction: column; align-items: flex-end` para que el botón permanezca alineado a la derecha cuando el panel se expande.
- **Cleanup:** `beforeUnmount()` limpia el intervalo del typewriter.

**Ubicación:** `components/gov-chat-frontend/src/components/FloatingChatWidget.vue`

---

### 5.3 Nueva vista: StudentPortalShellView.vue

**Propósito:** Réplica de portal universitario genérico que embebe el `FloatingChatWidget` como componente inyectable. Sirve como entorno de demo para presentaciones institucionales y pruebas de integración.

**Ubicación:** `components/gov-chat-frontend/src/views/StudentPortalShellView.vue`

---

### 5.4 Mejoras en ChatBotComponent

- **Loading state:** Reemplazado el overlay `loading-spinner` (que bloqueaba la UI completa) por una burbuja de bot con animación `.dot-pulse` inline, consistente con el estilo de mensajes.
- **Efecto typewriter:** Mismo patrón `splice()` que FloatingChatWidget para reactividad Vue 2 correcta.

---

### 5.5 LoginScreen — remoción de cuentas guardadas

El componente `LoginScreen.vue` mostraba una sección de "Saved Accounts" con botones de Google y Facebook hardcodeados — artefacto del framework base que no aplica al contexto UPB.

**Cambio:** Removido el bloque template completo de `savedAccounts`. Array `savedAccounts: []` en `data()`.

---

### 5.6 ChatFolders — corrección de timing Vue 2

**Problema:** El watcher `currentUser` con `immediate: true` se disparaba antes de que `created()` terminara de inicializar el estado, causando que la primera carga de conversaciones fallara silenciosamente.

**Corrección en `ChatFolders.vue`:**
```javascript
mounted() {
  this.$nextTick(() => {
    if (this.currentUser && this.conversations.length === 0) {
      this.loadConversationsForCurrentTab();
    }
  });
}
```

---

## 6. Resumen de Decisiones de Arquitectura

| Decisión | Alternativa considerada | Razón de la elección |
|---|---|---|
| Migrar a TEI CPU-only | Mantener GPU / usar ONNX Runtime | Portabilidad en entorno académico sin GPUs dedicadas |
| Proxy Gemini → OpenAI format | Fork directo de ChatQnA para Gemini nativo | Menor superficie de cambio; reutiliza el pipeline OPEA existente |
| Builds Docker personalizados | Fork del repo upstream OPEA | Control de versiones propio; upstream puede cambiar sin previo aviso |
| ArangoDB como graph DB | PostgreSQL, MongoDB | Ya integrado en GENIE.AI; las relaciones conversación/usuario/carpeta son naturalmente grafos |
| Vue 2 (Options API) | Migrar a Vue 3 / Composition API | El framework base usa Vue 2; migración fuera del scope del MVP |
| `!= true` en lugar de `== false` | Agregar campo `isArchived: false` por defecto | Null-safe sin migración de datos existentes |
| Volume mount para `/config` | CI/CD con rebuild automático | Iteración rápida durante desarrollo; sin impacto en producción |
| Toggle env para traducción | Condicional en Dockerfile | Configuración en tiempo de ejecución, más flexible para distintos entornos |

---

## 7. Superficie de Cambios

### Archivos comprometidos en commits (vs commit inicial `541e216`)
- `docker-compose.yaml` — 25+ cambios de servicios
- `env` — variables de entorno completas para despliegue
- `genie-ai-overlay/gemini-adapter-wrapper/` — 3 archivos nuevos
- `genie-ai-overlay/retriever/Dockerfile-retriever_genie-ai`
- `components/gov-chat-backend/services/query-service.js`

### Archivos modificados no comprometidos (working tree)
- **Backend:** `query-service.js`, `chat-history-service.js`, `translation-service.js`, `service-routes.js`
- **Frontend:** `App.vue`, `ChatBotComponent.vue`, `ChatFolders.vue`, `LoginScreen.vue`, `NavBarComponent.vue`, `RightSideBarComponent.vue`, `theme-variables.css`, `global-theme.css`, `router.js`, `main.js`
- **Vistas nuevas:** `FloatingChatWidget.vue`, `StudentPortalShellView.vue`
- **Config:** `genie-ai-config.json` (re-branded), logos PNG nuevos
- **Pipeline:** `genieai_chatqna.py`, `entrypoint.sh`, `genieai_api_protocol.py`
- **Infra:** `docker-compose.yaml` (CPU images, volumes, builds), `arango-init.js`

---

## 8. Estado del Sistema

| Componente | Estado |
|---|---|
| ArangoDB | ✅ Edge collections correctas, datos de usuario funcionales |
| ChatQnA Pipeline | ✅ Conectado a Gemini vía adapter wrapper |
| Intent Classifier (Labeler) | ✅ Re-implementado con DistilBERT zero-shot (CPU/GPU auto) |
| TEI Embeddings / Reranker | ✅ CPU-only, estable |
| Frontend ZURI AI | ✅ Branding UPB, build desplegado |
| FloatingChatWidget | ✅ Funcional, inyectable |
| Autenticación / Sesiones | ✅ Funcionando |
| Traducción automática | ⚙️ Deshabilitada por toggle (configurable) |
| ClamAV (antivirus docs) | ⚙️ Removido del MVP (puede reintegrarse) |

---

*Documento generado para portfolio de ingeniería de software. Stack: Vue 2 · Node.js · Python/FastAPI · ArangoDB · Docker Compose · OPEA · Google Gemini API*
