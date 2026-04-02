# GENIE.AI - Diagnóstico y Soluciones Aplicadas

**Fecha**: Marzo 5, 2026  
**Problemas Resueltos**: 2 críticos + 1 preventivo

---

## 🔴 PROBLEMA 1: GRAPH_SOURCE Vacío (Chunks no se embedden)

### Síntomas
- ✅ Archivo ingestado con `chunk_count: 10` y status `ingested` en colección `files`
- ❌ Colección `GRAPH_SOURCE` vacía (sin vectores)
- ❌ LLM dice "no tengo información" porque no hay chunks embeddidos

### Causa Raíz
En `genieai_dataprep_arangodb.py` línea 540-575:
1. Sistema intenta usar `LLMGraphTransformer` para extraer entidades y crear grafo
2. **Cuando LLMGraphTransformer falla** (ej: Gemini overloaded), cae back a retry individual
3. **Si retry también falla**: los chunks se descartan completamente ❌
4. **No hay fallback** para insertar chunks sin transformación de grafo

### Solución Aplicada

#### A. Código: Fallback para inserción directa de chunks
**Archivo**: `genie-ai-overlay/dataprep/genieai_dataprep_arangodb.py`

**Cambio 1** (líneas 557-575): Mejorado manejo de errores en `_process_batch`
```python
# Cuando LLMGraphTransformer falla:
# 1. Intenta retry individual
# 2. Si eso también falla → llama a _insert_chunk_directly()
# 3. Los chunks se insertan en graph_source SIN transformación de grafo
await self._insert_chunk_directly(retry_doc, input.file_id, graph_name)
```

**Cambio 2** (nuevo método `_insert_chunk_directly`): Inserción directa con embeddings
```python
async def _insert_chunk_directly(self, doc: Document, file_id: str, graph_name: str):
    """
    Fallback: Inserta chunk directamente en {graph_name}_SOURCE collection
    - Calcula embedding via TEI
    - Inserta documento con metadatos
    - Asegura que el chunk esté disponible para búsqueda semántica
    """
    # 1. Obtiene colección SOURCE
    source_col_name = f"{graph_name}_SOURCE"
    
    # 2. Genera embedding para el chunk
    embedding = await asyncio.to_thread(
        self.embeddings.embed_query, 
        chunk_text
    )
    
    # 3. Crea documento con metadatos
    chunk_doc = {
        "text": chunk_text,
        "file_id": file_id,
        "chunk_index": metadata.get("chunk_index", 0),
        "chunk_labels": metadata.get("chunk_labels", []),
        "created_at": datetime.now().isoformat(),
        "embedding": embedding  # ← Campo vital para búsqueda
    }
    
    # 4. Inserta en base de datos
    source_collection.insert(chunk_doc)
```

**Beneficio**: Chunks nunca se pierden. Incluso si Gemini falla totalmente, los chunks se embedden localmente (via TEI) y se pueden recuperar.

---

## 🔴 PROBLEMA 2: Gemini API 503 + KeyError

### Síntomas
```
genie-ai-chatqna-server | KeyError: 'choices'
genie-ai-gemini-adapter | litellm.exceptions.ServiceUnavailableError: 503 Service Unavailable
```
- Chat fails inmediatamente con error de servidor
- A pesar de API key nueva, se consume rápidamente

### Causa Raíz

#### Issue 2a: KeyError en chatqna.py (línea 510)
```python
# Código original - asume estructura OpenAI estándar:
next_data["text"] = data["choices"][0]["message"]["content"]

# Pero cuando Gemini retorna error 503, estructura es:
{
    "error": {
        "code": 503,
        "message": "Service Unavailable",
        "status": "UNAVAILABLE"
    }
}
# ↑ No tiene "choices" → KeyError!
```

#### Issue 2b: Retry logic agresivo en LiteLLM
LiteLLM por defecto:
- Reintenta automáticamente en errores 5xx
- Permite hasta 4 retries por defecto
- Result: 1 request falla → 4 retries automáticos → **5 consumiciones de API key**

### Soluciones Aplicadas

#### A. Código: Manejo robusto de errores en chatqna.py
**Archivo**: `genie-ai-overlay/chatqna/genieai_chatqna.py` (línea 505-525)

```python
# ANTES (línea 510):
next_data["text"] = data["choices"][0]["message"]["content"]  # ❌ KeyError!

# DESPUÉS:
elif self.services[cur_node].service_type == ServiceType.LLM and not llm_parameters_dict["stream"]:
    if "faqgen" in self.services[cur_node].endpoint:
        next_data = data
    else:
        # FIX: Valida estructura antes de acceder
        if "choices" in data and isinstance(data["choices"], list) and len(data["choices"]) > 0:
            if "message" in data["choices"][0] and "content" in data["choices"][0]["message"]:
                next_data["text"] = data["choices"][0]["message"]["content"]
            else:
                logger.error(f"LLM response missing expected message structure: {data}")
                next_data["text"] = "Error: No response from LLM"
        else:
            # Handle error responses (e.g., 503 Service Unavailable)
            logger.error(f"LLM returned error response: {data}")
            if "error" in data:
                next_data["text"] = f"Error: {data['error'].get('message', 'Unknown error from LLM')}"
            else:
                next_data["text"] = "Error: LLM service temporarily unavailable"
```

**Beneficio**: 
- No crashes con KeyError
- Usuario ve mensaje claro de error
- Logs muestran qué estructura devolvió Gemini

#### B. Configuración: Desactivar retries en LiteLLM
**Archivo**: `.env`

```dotenv
# CRITICAL FIX: Disable aggressive retries to prevent API key depletion
LITELLM_NUM_RETRIES=0           # ← Desactiva retries automáticos
LITELLM_REQUEST_TIMEOUT=30      # ← Timeout razonable (no infinito)
LITELLM_DROP_PARAMS=True        # ← Mantiene compatibilidad
```

**Archivo**: `docker-compose.yaml` (gemini-adapter service)

```yaml
environment:
  - GEMINI_API_KEY=${GEMINI_API_KEY}
  - LITELLM_DROP_PARAMS=True
  - LITELLM_NUM_RETRIES=${LITELLM_NUM_RETRIES:-0}      # ← Nuevo
  - LITELLM_REQUEST_TIMEOUT=${LITELLM_REQUEST_TIMEOUT:-30}  # ← Nuevo
```

**Beneficio**:
- 1 error de Gemini = 1 consumición de API key (no 5)
- Con free tier (5 req/min), puedes hacer 5 chats sin agotarlo
- Evita loops infinitos de reintentos

---

## 🟡 PROBLEMA PREVENIDO: Uso incorrecto de FORCE_STANDARD_LOADER

### Estado
✅ Ya estaba configurado correctamente en `.env`:
```dotenv
FORCE_STANDARD_LOADER=true
```

Este flag asegura que si LLMGraphTransformer falla, el sistema usa un loader simple que NO requiere Gemini.

**Nota**: Con nuestro nuevo fallback `_insert_chunk_directly()`, este flag es redundante pero útil como respaldo.

---

## 📋 RESUMEN DE CAMBIOS

| Archivo | Línea(s) | Cambio | Propósito |
|---------|----------|--------|-----------|
| `genieai_dataprep_arangodb.py` | +42 líneas | Nuevo método `_insert_chunk_directly()` | Fallback para chunks |
| `genieai_dataprep_arangodb.py` | 557-578 | Mejorado error handling | Llama fallback si LLM falla |
| `genieai_chatqna.py` | 505-525 | Validación de estructura + error handling | Previene KeyError |
| `.env` | +4 líneas | `LITELLM_NUM_RETRIES=0`, timeout | Desactiva retries |
| `docker-compose.yaml` | 334-338 | Pasa variables a LiteLLM | Aplica config de retries |

---

## 🚀 CÓMO APLICAR

### Opción A: Script Automático (Recomendado)
```powershell
cd c:\Users\HP\Desktop\programacion\chatbot_UPB\genie-ai-replica-MVP-ready
.\restart-with-fixes.ps1
```

### Opción B: Manual
```bash
docker compose down -v
docker compose up -d arango-vector-db redis-cache kong
# esperar 5 segundos
docker compose up -d tei tei-reranker-serving embedding reranker
# esperar 10 segundos  
docker compose up -d  # resto de servicios
```

---

## ✅ VALIDACIÓN POST-APLICACIÓN

### 1. Verificar chunks en graph_source
```bash
docker exec arango-vector-db arangosh --server.username root --server.password arangopassword123
> db._query("FOR doc IN GRAPH_SOURCE_SOURCE RETURN doc").toArray()
```
Deberías ver ~10 documentos con campos: `text`, `file_id`, `chunk_index`, `embedding`

### 2. Verificar logs de fallback
```bash
docker logs genie-ai-dataprep-arango | grep "Direct chunk insertion"
```
Si aparece, significa que el fallback fue usado ✅

### 3. Probar chat con error handling
- En frontend, presiona "just chat" varias veces
- Deberías ver mensajes claros de error (no crashes)
- Revisa uso de API key: debería ser ~1 request per query (no 5+)

---

## 📊 IMPACTO ESPERADO

### Antes de fixes:
- ❌ Chunks perdidos (graph_source vacío)
- ❌ Chat crashes con KeyError en 503
- ❌ API key consumida en 2-3 queries (retries)
- ❌ Usuario ve "Internal Server Error"

### Después de fixes:
- ✅ Chunks siempre embeddidos (fallback local)
- ✅ Chat maneja errores gracefully
- ✅ API key dura ~5+ chats (sin retries agresivos)
- ✅ Usuario ve: "Error: LLM service temporarily unavailable"

---

## 🔧 CONFIGURACIÓN FUTURA (Opcional)

Si quieres evitar completamente Gemini (para no agotar free tier):

### Usa Ollama local
```bash
# Instalar Ollama y ejecutar modelo local
ollama run mistral

# En .env:
LLM_ENDPOINT=http://ollama:11434
VLLM_ENDPOINT=http://ollama:11434
```

### Usa modelo open-source cuantizado
```bash
# Ej: LLaMA 2 7B cuantizado
docker run -d \
  -p 8000:8000 \
  -e MODEL_NAME=meta-llama/Llama-2-7b-chat-hf \
  ghcr.io/mistralai/mistral:v0.0.1
```

---

## 📝 NOTAS

- Los cambios son **retrocompatibles** (no rompen código existente)
- El **fallback es automático** (no requiere configuración)
- Los **logs mejoran debugging** (puedes ver qué estructura devolvió Gemini)
- **LiteLLM sigue siendo proxy** válido (para Gemini pro, OpenAI, Azure, Anthropic, etc.)

---

**Estado**: ✅ Listo para uso  
**Próximo paso**: Ejecuta el script `restart-with-fixes.ps1` y prueba la ingesta de documentos
