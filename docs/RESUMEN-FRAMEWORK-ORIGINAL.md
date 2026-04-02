# Resumen del Framework Original GENIE.AI

## 1. Que es GENIE.AI (framework original)
GENIE.AI se describe como un framework open source para crear soluciones de IA generativa orientadas al sector publico (por ejemplo: chatbots, asistentes digitales y flujos de generacion de contenido).

Su enfoque original es:
- Stack abierto e interoperable, alineado con principios de GovStack.
- Arquitectura modular y adaptable.
- Despliegue dockerizado y orientado a entornos escalables (incluyendo Kubernetes).
- Base tecnologica construida sobre OPEA (Open Platform for Enterprise AI).

## 2. Objetivo original del framework
El objetivo del framework es habilitar a gobiernos e instituciones para construir aplicaciones GenAI/RAG:
- Con menor costo que alternativas comerciales.
- Ajustadas a necesidades y datos del sector publico.
- Con control de despliegue, configuracion y mantenimiento.
- Con capacidad de integrarse con infraestructura digital publica mas amplia.

## 3. Arquitectura original (alto nivel)
A partir de los README, la arquitectura original se organiza como una plataforma multi-servicio con capas bien separadas.

### 3.1 Capa de interfaz
- Frontend web en Vue 3.
- Aplicacion movil en Flutter (Android/iOS/Web) como cliente adicional del framework.

### 3.2 Capa de API y negocio
- Backend principal en Node.js + Express.
- Modelo de servicios con enfoque microservicios para funciones como:
  - autenticacion y sesiones,
  - analitica,
  - historial de chat,
  - gestion de perfiles,
  - gestion de categorias/servicios de conocimiento.

### 3.3 Capa de RAG
- Integracion con OPEA para acceso a modelos y pipeline de generacion.
- Flujo de retrieval con enfoque hibrido (vector + grafo + filtros por metadatos) en el microservicio retriever.
- Posibilidad de post-procesamiento/sumarizacion para mejorar el contexto enviado al modelo.

### 3.4 Capa de datos e ingestion
- ArangoDB como base multimodelo (documentos, grafo y vector index experimental).
- Servicio document-repository para:
  - carga y gestion de archivos,
  - escaneo antivirus (ClamAV),
  - extraccion/edicion de metadatos,
  - ingest/retract hacia el flujo de dataprep/RAG.
- Redis como cache (en particular para traducciones, segun README de componentes).

### 3.5 Capa de seguridad y gateway
- API Gateway con Kong.
- Reverse proxy con NGINX.
- IAM con Keycloak en el entorno descrito del gateway.
- Librerias compartidas para:
  - headers de seguridad,
  - middleware de deteccion de amenazas,
  - rate limiting,
  - logging y observabilidad.

## 4. Estructura funcional del framework (segun README)
En terminos de organizacion, el framework original aparece distribuido por dominios:

- `components/`
  - `gov-chat-frontend/` (UI web Vue 3)
  - `gov-chat-backend/` (API y logica de negocio)
  - `document-repository/` (gestion documental e ingestion)
  - `arangodb/` (operacion de base de datos y scripts)
  - `shared/lib/` (librerias compartidas)

- `genie-ai-overlay/`
  - microservicios de la capa OPEA/RAG (incluyendo retriever).

- `api-gateway-solution/`
  - configuracion de Kong + NGINX + servicios auxiliares de seguridad/autenticacion.

- `mobile/genie_ai_mobile/`
  - cliente movil Flutter conectado al framework RAG.

## 5. Tech stack original identificado
### Frontend y clientes
- Vue.js 3 (web)
- Flutter/Dart (mobile)

### Backend y APIs
- Node.js
- Express

### IA/RAG
- OPEA como plataforma base
- Microservicios de retrieval con FastAPI (segun README del retriever)
- Integracion con modelos/servicios LLM y embeddings (OpenAI/vLLM mencionados en retriever)

### Datos e infraestructura
- ArangoDB 3.12.x (document + graph + vector)
- Redis 7 (cache)
- ClamAV (escaneo de archivos)
- Docker / Docker Compose
- Kubernetes-ready (segun README principal)

### Gateway, seguridad y operacion
- Kong (API Gateway)
- NGINX (reverse proxy)
- Keycloak (IAM en stack de gateway)
- OWASP ZAP (testing de seguridad en stack gateway)
- Libreria de logging basada en Winston

## 6. Capacidades originales del framework (resumen)
- Chat conversacional con soporte RAG.
- Configuracion dinamica del frontend via JSON (enfoque configuration-driven).
- Soporte multilenguaje/i18n.
- Gestion de historial de conversaciones.
- Gestion de conocimiento documental con metadata.
- Retrieval hibrido para mejorar relevancia y contexto.
- Enfoque enterprise: autenticacion, seguridad, administracion y observabilidad.

## 7. Limites de este resumen
Este documento resume exclusivamente lo que se declara en los README revisados del framework base y sus componentes. No incluye analisis de cambios locales ni personalizaciones del repositorio actual.

## 8. Fuentes internas revisadas
- README principal del proyecto.
- README de `components/`.
- README de `components/gov-chat-backend/`.
- README de `components/gov-chat-frontend/`.
- README de `components/document-repository/`.
- README de `components/arangodb/`.
- README de `components/shared/lib/`.
- README de `genie-ai-overlay/retriever/`.
- README de `api-gateway-solution/`.
- README de `mobile/genie_ai_mobile/`.
