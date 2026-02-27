# Análisis Completo e Intensivo de SimpleL7Proxy

## Tabla de Contenidos

1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [¿Qué es SimpleL7Proxy?](#qué-es-simplel7proxy)
3. [Arquitectura General](#arquitectura-general)
4. [Estructura del Repositorio](#estructura-del-repositorio)
5. [Componentes Principales (src/SimpleL7Proxy)](#componentes-principales)
   - [Punto de Entrada y Pipeline](#punto-de-entrada-y-pipeline)
   - [Servidor HTTP (server.cs)](#servidor-http)
   - [Sistema de Cola con Prioridad](#sistema-de-cola-con-prioridad)
   - [Proxy Workers](#proxy-workers)
   - [Backend y Balanceo de Carga](#backend-y-balanceo-de-carga)
   - [Circuit Breaker](#circuit-breaker)
   - [Sistema Asíncrono (Feeder)](#sistema-asíncrono-feeder)
   - [Gestión de Usuarios](#gestión-de-usuarios)
   - [Blob Storage](#blob-storage)
   - [Eventos y Telemetría](#eventos-y-telemetría)
   - [Configuración](#configuración)
   - [DTOs](#dtos)
   - [Shutdown Coordinado](#shutdown-coordinado)
6. [Proyectos de Soporte](#proyectos-de-soporte)
   - [RequestAPI (Azure Functions)](#requestapi)
   - [HealthProbe (Sidecar)](#healthprobe)
   - [Shared (Modelos Compartidos)](#shared)
   - [TestClient](#testclient)
   - [Functions (Backend Mock)](#functions)
7. [Flujo Completo de una Solicitud](#flujo-completo-de-una-solicitud)
8. [Patrones de Diseño Utilizados](#patrones-de-diseño-utilizados)
9. [Integraciones con Azure](#integraciones-con-azure)
10. [Seguridad](#seguridad)
11. [Observabilidad y Monitoreo](#observabilidad-y-monitoreo)
12. [Escenarios de Despliegue](#escenarios-de-despliegue)
13. [Configuración Detallada](#configuración-detallada)
14. [Pruebas](#pruebas)

---

## Resumen Ejecutivo

**SimpleL7Proxy** es un gateway de capa 7 (Layer 7) de alto rendimiento, diseñado específicamente para optimizar cargas de trabajo de **Modelos de Lenguaje Grande (LLMs)** en el ecosistema Azure. Funciona como una capa de orquestación inteligente que se despliega junto con **Azure API Management (APIM)** y **Azure AI Foundry**, proporcionando capacidades avanzadas de enrutamiento, priorización, resiliencia y observabilidad que los gateways comerciales no ofrecen.

**Versión actual**: `2.2.9-d2`  
**Runtime**: .NET 9  
**Despliegue principal**: Azure Container Apps (ACA)  
**Licencia**: Open Source

---

## ¿Qué es SimpleL7Proxy?

En términos simples, SimpleL7Proxy es un **proxy inverso inteligente** que se coloca entre los clientes (aplicaciones que consumen AI) y los proveedores de modelos LLM (como Azure OpenAI). A diferencia de un proxy inverso estándar como NGINX, SimpleL7Proxy entiende la semántica de las solicitudes de AI y puede:

- **Priorizar** solicitudes interactivas de chat sobre procesos batch de embeddings.
- **Balancear carga** inteligentemente entre múltiples regiones y endpoints.
- **Proteger** backends con circuit breakers y reintentos automáticos.
- **Procesar asincrónicamente** solicitudes que exceden los timeouts HTTP estándar.
- **Capturar métricas de tokens** en tiempo real, incluso en respuestas de streaming.
- **Gobernar** el uso por usuario con throttling y perfiles de acceso.

---

## Arquitectura General

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENTES (Apps de AI)                        │
└─────────────────────┬───────────────────────────────────────────────┘
                      │ HTTP/HTTPS
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Azure API Management (APIM)                         │
│              (Políticas, OAuth, Rate Limiting)                        │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      SimpleL7Proxy (ACA)                             │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────────────────┐  │
│  │ Server   │→ │ Cola de      │→ │ ProxyWorker Pool              │  │
│  │ (HTTP    │  │ Prioridad    │  │  ├─ Worker 1 (Priority 1-3)   │  │
│  │ Listener)│  │ (PriQueue)   │  │  ├─ Worker 2 (Priority 1-3)   │  │
│  │          │  │              │  │  ├─ ...                        │  │
│  │          │  │  P1 ████     │  │  └─ Worker N (Probe, P0)      │  │
│  │          │  │  P2 ██       │  │                                │  │
│  │          │  │  P3 █        │  └───────────────┬───────────────┘  │
│  └──────────┘  └──────────────┘                  │                  │
│                                                   ▼                  │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │              Backend Selection & Circuit Breaker                 │ │
│  │  Host A (Azure OpenAI East US) ✓  Latency: 45ms                │ │
│  │  Host B (Azure OpenAI West EU) ✓  Latency: 120ms               │ │
│  │  Host C (Azure OpenAI Japan)   ✗  Circuit OPEN                  │ │
│  └─────────────────────────────┬───────────────────────────────────┘ │
│                                │                                     │
│  ┌─────────────────────┐  ┌───┴────────────┐  ┌──────────────────┐  │
│  │ Telemetría          │  │ Async Feeder   │  │ Health Probe     │  │
│  │ (App Insights,      │  │ (Service Bus)  │  │ (Sidecar :9000)  │  │
│  │  Event Hub)         │  │                │  │                  │  │
│  └─────────────────────┘  └────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│           Azure OpenAI / AI Foundry / Backends LLM                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Estructura del Repositorio

```
SimpleL7Proxy/
├── src/
│   ├── SimpleL7Proxy/          # Proyecto principal del proxy
│   │   ├── Backend/            # Sondeo de backends, circuit breaker, health
│   │   ├── BlobStorage/        # Escritura/lectura en Azure Blob Storage
│   │   ├── Config/             # Esquema de configuración (BackendOptions)
│   │   ├── DTO/                # Objetos de transferencia de datos
│   │   ├── Events/             # Telemetría (App Insights, Event Hub)
│   │   ├── Feeder/             # Consumo de mensajes async (Service Bus)
│   │   ├── Proxy/              # Workers, lifecycle, async processing
│   │   ├── Queue/              # Cola de prioridad concurrente
│   │   ├── User/               # Perfiles y priorización de usuarios
│   │   ├── Program.cs          # Punto de entrada, DI, startup
│   │   ├── server.cs           # Listener HTTP principal
│   │   ├── RequestData.cs      # Wrapper de request con estado
│   │   ├── RequestType.cs      # Enum: Sync/Async/Background
│   │   ├── Constants.cs        # Constantes (versión, rutas de probes)
│   │   └── CoordinatedShutdownService.cs  # Shutdown graceful
│   │
│   ├── RequestAPI/             # Azure Functions para tracking de requests
│   ├── HealthProbe/            # Sidecar de health checks (puerto 9000)
│   └── Shared/                 # Modelos y enums compartidos
│
├── deployment/                 # YAMLs de despliegue para ACA
├── docs/                       # Documentación exhaustiva
├── functions/                  # Backend mock para pruebas
├── scripts/                    # Scripts de automatización
├── test/                       # Pruebas de integración
├── TestClient/                 # Cliente de prueba (Azure OpenAI)
├── SimpleL7Proxy.Test/         # Tests unitarios (MSTest)
├── APIM-Policy/                # Políticas de referencia para APIM
└── SimpleL7Proxy.sln           # Solución .NET
```

---

## Componentes Principales

### Punto de Entrada y Pipeline

**Archivo**: `src/SimpleL7Proxy/Program.cs`

El punto de entrada configura toda la inyección de dependencias y el pipeline de la aplicación. Los servicios registrados incluyen:

| Servicio | Tipo | Descripción |
|----------|------|-------------|
| `Server` | `BackgroundService` | Listener HTTP principal |
| `ProxyWorkerCollection` | `IHostedService` | Pool de workers de proxy |
| `BackendTokenProvider` | `IHostedService` | Proveedor de tokens OAuth |
| `ConcurrentPriQueue<RequestData>` | Singleton | Cola de prioridad compartida |
| `ServiceBusRequestService` | Condicional | Integración con Service Bus (modo async) |
| `BackupAPIService` | Condicional | API de respaldo para requests |
| `BlobWriteQueue` / `QueuedBlobWriter` | Condicional | Escritura async a Blob Storage |
| `UserPriorityService` | Singleton | Servicio de priorización por usuario |
| `UserProfile` | Singleton | Perfiles de usuario cargados dinámicamente |
| `StreamProcessorFactory` | Singleton | Fábrica para procesadores de stream |
| `SharedIteratorRegistry` | Opcional | Registro de iteradores compartidos para balanceo |

La telemetría se configura condicionalmente con **Application Insights** cuando `APPINSIGHTS_CONNECTIONSTRING` está presente.

---

### Servidor HTTP

**Archivo**: `src/SimpleL7Proxy/server.cs`  
**Clase**: `Server` (hereda de `BackgroundService`)

El servidor HTTP escucha en un puerto configurable y es responsable de:

1. **Aceptar conexiones HTTP** entrantes.
2. **Extraer la prioridad** del header `S7PPriority` (por defecto, prioridad estándar).
3. **Crear un `RequestData`** envolviendo el contexto HTTP con metadatos.
4. **Encolar** la solicitud en la `ConcurrentPriQueue` con la prioridad correspondiente.
5. **Gestionar el shutdown** en dos fases:
   - **Fase 1** (`StopListening()`): Deja de aceptar nuevas solicitudes.
   - **Fase 2** (`StopProbes()`): Detiene los health probes (permite que el orquestador detecte el drenaje).

```
Cliente HTTP → Server.Run() → RequestData → ConcurrentPriQueue → ProxyWorker
```

---

### Sistema de Cola con Prioridad

**Directorio**: `src/SimpleL7Proxy/Queue/`

El sistema de cola es uno de los componentes más sofisticados del proxy. Implementa una **cola de prioridad concurrente** con las siguientes características:

#### `ConcurrentPriQueue<T>`
- **Cola híbrida** con señalización basada en semáforos.
- **Fast-path para probes** (prioridad 0): matching directo lock-free con workers.
- **Señalización con `SemaphoreSlim`**: Los workers se despiertan cuando hay items disponibles.
- **Timeout de 40ms** en la espera de señales para balancear latencia/throughput.

#### `PriorityQueueItem<T>`
- Wrapper comparable con **doble prioridad**: primaria + boost secundario.
- Ordenamiento: `Prioridad Primaria > Boost Secundario > Timestamp`.
- Garantiza que solicitudes de mayor prioridad siempre se procesan primero.

#### `ConcurrentSignal<T>`
- Gestiona workers en espera y enrutamiento de tareas específicas de probes.
- Coordina la asignación de trabajo entre workers disponibles.

#### `WorkerTask<T>`
- Wrapper para `TaskCompletionSource` con acoplamiento de prioridad.
- Permite que un worker "espere" hasta que haya trabajo de su nivel de prioridad.

**Flujo de la cola**:
```
Enqueue(item, priority)
    │
    ├─ Priority == 0 (Probe)?
    │   └─ Lock-free: Matching directo con worker de probe
    │
    └─ Priority > 0 (Normal)?
        └─ Lock + Enqueue en PriorityQueue interna
            └─ SemaphoreSlim.Release() → Despierta worker
```

---

### Proxy Workers

**Directorio**: `src/SimpleL7Proxy/Proxy/`

#### `ProxyWorkerCollection`
Gestiona el pool de workers distribuidos por prioridad:
- **Workers de prioridad** (1-3): Procesan solicitudes normales según su nivel.
- **Workers de probe** (prioridad 0): Dedicados exclusivamente a health checks.
- La distribución se configura con `WorkerDistribution` (ej: `"3,2,1"` → 3 workers P1, 2 P2, 1 P3).

#### `ProxyWorker`
El motor principal de procesamiento. Cada worker ejecuta un loop infinito:

1. **Desencolar** la siguiente solicitud de la cola de prioridad.
2. **Seleccionar backend** óptimo (menor latencia, sin circuit breaker abierto).
3. **Proxear la solicitud** al backend seleccionado.
4. **Manejar la respuesta**:
   - **Sync**: Streamer directo al cliente HTTP.
   - **Async**: Escribir en Blob Storage, notificar por Service Bus.
5. **Reintentar** en caso de fallo (respetando el circuit breaker).
6. **Emitir telemetría** con métricas de tokens, latencia, etc.

#### `AsyncWorker`
Maneja el procesamiento de solicitudes asíncronas:
- Gestión de I/O con Blob Storage.
- Generación de tokens SAS para acceso temporal.
- Actualización de estado vía Service Bus.

#### `RequestLifecycleManager`
Máquina de estados para transiciones de solicitudes:
```
Pending → Processing → Processed/Failed/Expired
                    ↗
Pending → AsyncTriggered → BlobWritten → Notified
```

#### `HealthCheckService`
Rastrea el estado de cada worker (desencolando, proxeando, escribiendo, etc.) y proporciona métricas para el endpoint `/health` incluyendo diagnóstico de memoria.

---

### Backend y Balanceo de Carga

**Directorio**: `src/SimpleL7Proxy/Backend/`

#### `Backends`
Servicio que monitorea la salud de los backends configurados:
- **Polling periódico** a cada backend con intervalo y timeout configurables.
- **Tracking de latencia** con promedios móviles.
- **Filtrado por tasa de éxito** (umbral configurable, ej: 95%).
- **Categorización**: Hosts específicos por ruta vs. catch-all.

#### Algoritmos de Balanceo de Carga

El balanceo opera en **5 etapas**:

| Etapa | Nombre | Descripción |
|-------|--------|-------------|
| 1 | Filtrado por ruta | Selecciona hosts que coincidan con el path del request |
| 2 | Creación de iterador | Según `LoadBalanceMode`: round-robin, latencia, random |
| 3 | Estrategia de iteración | `SinglePass` o `MultiPass` con `MaxAttempts` |
| 4 | Iteradores compartidos | Con TTL, para consistencia entre requests concurrentes |
| 5 | Circuit Breaker | Verificación final antes de enviar al backend |

**Modos soportados**:
- **Latency**: Ordena backends por latencia medida (menor primero).
- **Round-Robin**: Distribución equitativa cíclica.
- **Random**: Selección aleatoria entre backends saludables.

#### `HostHealthCollection`
Mantiene colecciones de salud de hosts:
- `ProbeableHostHealth`: Hosts con endpoint de health check activo.
- `NonProbeableHostHealth`: Hosts serverless o directos sin sondeo.

---

### Circuit Breaker

**Archivo**: `src/SimpleL7Proxy/Backend/CircuitBreaker.cs`

Implementa el patrón Circuit Breaker con ventanas de tiempo deslizantes:

```
Estado CERRADO ──(fallos > umbral)──→ Estado ABIERTO
     ▲                                       │
     └───(fallos < umbral en ventana)────────┘
```

- **Ventana de tiempo** (`CBTimeslice`): 60 segundos por defecto.
- **Umbral de errores** (`CBErrorThreshold`): 50 fallos por defecto.
- **Auto-recuperación**: Se cierra automáticamente cuando los errores bajan del umbral.
- **Estado global**: Compartido entre todos los circuit breakers para visibilidad.
- Si **todos** los hosts están en estado abierto, retorna HTTP 503.

---

### Sistema Asíncrono (Feeder)

**Directorio**: `src/SimpleL7Proxy/Feeder/`

El sistema asíncrono permite procesar solicitudes que exceden los timeouts HTTP estándar (30+ minutos):

#### `AsyncFeeder` (IHostedService)
1. **Escucha** la cola "feeder" de Azure Service Bus.
2. **Parsea** mensajes (`RequestMessage` o `RequestAPIDocument`).
3. **Convierte** a `RequestData` aplicando perfiles de usuario y prioridades.
4. **Re-encola** en la `ConcurrentPriQueue` con hidratación opcional desde Blob Storage.

#### `NormalRequest` (IRequestProcessor)
Restaura solicitudes asíncronas desde Blob Storage durante recuperación de fallos.

#### `OpenAIBackgroundRequest`
Procesador especializado para solicitudes batch de la API de OpenAI.

**Flujo asíncrono completo**:
```
Cliente → Proxy → Timeout excedido → 202 Accepted
                                          │
                 ┌────────────────────────┘
                 ▼
         Blob Storage (request/response)
                 │
                 ▼
         Service Bus (notificación de estado)
                 │
                 ▼
         RequestAPI (Cosmos DB tracking)
                 │
                 ▼
         Cliente consulta estado por polling
```

---

### Gestión de Usuarios

**Directorio**: `src/SimpleL7Proxy/User/`

#### `UserPriority`
Sistema de priorización basada en el uso:
- **Tracking por usuario** con `ConcurrentDictionary` + operaciones `Interlocked`.
- **Cálculo de boost**: Si un usuario tiene menos del umbral% del total de solicitudes, recibe un boost de prioridad.
- **Fair queuing**: Previene que un solo usuario monopolice los recursos ("noisy neighbor").

#### `UserProfile`
Gestión dinámica de perfiles:
- **Carga desde URL** configurable (JSON externo).
- **Campos**: `userId`, `S7PPriorityKey`, configuración async, headers custom.
- **Soft-deletion** con TTL para limpieza automática.
- **Refresh periódico** sin necesidad de reiniciar el servicio.

#### `AsyncClientInfo`
Configuración per-usuario para operaciones asíncronas:
- Container de Blob Storage dedicado.
- Topic de Service Bus para notificaciones.
- Flag de generación de tokens SAS.
- Timeout personalizado.

---

### Blob Storage

**Directorio**: `src/SimpleL7Proxy/BlobStorage/`

#### `BlobWriter`
Abstracción principal para Azure Blob Storage:
- **Pool de contenedores**: Clientes cacheados por `userId` en `ConcurrentDictionary`.
- **Reintentos** para conflictos 409 con backoff exponencial.
- **Generación de tokens SAS**: Clave de delegación de usuario (MI) o clave de cuenta.
- **Streaming**: Soporte para escritura de respuestas grandes sin buffer completo.

#### `QueuedBlobWriter` / `BlobWriteQueue`
Escritura asíncrona encolada para no bloquear el procesamiento de solicitudes:
- Las operaciones de escritura se encolan y procesan en background.
- Garantiza el flush completo antes del shutdown.

#### `NullBlobWriter`
Implementación no-op para cuando Blob Storage está deshabilitado.

---

### Eventos y Telemetría

**Directorio**: `src/SimpleL7Proxy/Events/`

#### `ProxyEvent`
Objeto de evento que hereda de `ConcurrentDictionary<string, string>`:
- **Tipos**: Request, Dependency, Exception, CustomEvent.
- **Campos estándar**: `S7P_RequestId`, `BackendHost`, `S7P_Priority`, `CircuitBreakerStatus`, `Retries`.
- **Enrutamiento**: `SendEvent()` despacha a App Insights o Event Hub según el tipo.
- **Reutilización**: Instancias reutilizables para minimizar allocations (Clear/Release).

#### Sinks de eventos

| Sink | Clase | Uso |
|------|-------|-----|
| Application Insights | `AppInsightsEventClient` | Métricas de AI, trazas, excepciones |
| Event Hub | `EventHubClient` | Streaming de eventos de alto volumen |
| Consola/Log | `LogFileEventClient` | Logging local |
| Compuesto | `CompositeEventClient` | Multi-sink simultáneo |

#### Métricas de AI
- Extracción de `Usage.Prompt_Tokens` y `Usage.Completion_Tokens` de respuestas streaming de OpenAI.
- Captura en tiempo real incluso en respuestas SSE (Server-Sent Events).

---

### Configuración

**Archivo**: `src/SimpleL7Proxy/Config/BackendOptions.cs`

Clase POCO central que contiene **todas** las opciones del proxy, inyectada vía `IOptions<BackendOptions>`:

| Categoría | Ejemplos de Configuración |
|-----------|---------------------------|
| **Backends** | Hasta 9 hosts, con formato avanzado de connection string |
| **Health Check** | Intervalo de polling, timeout, umbral de éxito |
| **Workers** | Cantidad de workers, distribución por prioridad |
| **Async** | Habilitación, Service Bus, Blob Storage, TTL |
| **Balanceo** | Modo (latency/roundrobin/random), iteradores compartidos |
| **Circuit Breaker** | Ventana de tiempo, umbral de errores |
| **Seguridad** | OAuth, headers permitidos/prohibidos, Managed Identity |
| **Usuarios** | URL de perfiles, throttling |

**Niveles de recarga**:
- **WARM**: Recargable en caliente por request (timeouts, logging).
- **DRAIN**: Requiere drenar workers (hosts, balanceo, OAuth).
- **COLD**: Reinicio completo (circuit breaker, infrastructure).
- **PARTIAL**: Mixto (prioridades, perfiles).

---

### DTOs

**Directorio**: `src/SimpleL7Proxy/DTO/`

#### `RequestDataDtoV1`
Snapshot serializable de `RequestData`:
- Guid, método, path, headers, prioridades, timestamps, campos async.
- `PopulateInto()` restaura campos a un `RequestData` vivo.
- Versionado para evolución de esquema.

#### `RequestDataConverter` / `RequestDataBackupService`
Persistencia de `RequestData` a Blob Storage y restauración para durabilidad.

---

### Shutdown Coordinado

**Archivo**: `src/SimpleL7Proxy/CoordinatedShutdownService.cs`

Orquesta un apagado graceful en orden específico:

```
1. Detener Server (no más requests) + AsyncFeeder + Cola
2. Expulsar requests async + cancelar delays de re-encolado
3. Esperar workers (timeout: TerminationGracePeriodSeconds)
4. Detener BackendTokenProvider → BackupAPIService → ServiceBusRequestService
5. Detener BlobWriteQueue (garantiza flush final)
6. Detener Health Probes (último, para seguridad del orquestador)
```

---

## Proyectos de Soporte

### RequestAPI

**Directorio**: `src/RequestAPI/` (Azure Functions)

Sistema de procesamiento de mensajes y tracking de solicitudes:

| Función | Trigger | Descripción |
|---------|---------|-------------|
| `DocumentProcessor` | Service Bus | Crea/actualiza documentos de tracking en Cosmos DB |
| `ReFeeder` | Timer (1 min) | Encuentra docs con estado `NeedsReprocessing` y los reenvía |
| `backgroundReqChecker` | Timer (1 min) | Resubmite solicitudes background en proceso |

**Estados del ciclo de vida** (`RequestAPIStatusEnum`):
```
New → InProgress → Completed/Failed
                ↘ NeedsReprocessing → ReSubmitted → ReProcessing
                ↘ BackgroundProcessing
```

### HealthProbe

**Directorio**: `src/HealthProbe/` (Sidecar)

Servidor Kestrel ligero en **puerto 9000** que aísla los health checks del procesamiento principal:

| Endpoint | Propósito |
|----------|-----------|
| `/readiness` | ¿Hay al menos un backend saludable? |
| `/startup` | ¿Completó la primera pasada del poller? |
| `/liveness` | ¿Está vivo el proceso? |
| `/internal/update-status` | Actualización dinámica de estado desde el proxy |

**Beneficio**: Evita falsos reinicios de Kubernetes cuando el proxy está bajo carga pesada.

### Shared

**Directorio**: `src/Shared/`

Modelos y enums compartidos entre proyectos:
- `RequestAPIDocument`: Entidad de solicitud rastreada con metadatos completos.
- `RequestAPIStatusEnum`: 8 estados del ciclo de vida.
- `HealthStatusEnum`: 8 estados de salud del servicio.

### TestClient

**Directorio**: `TestClient/`

Cliente de prueba que integra con Azure OpenAI:
- Pruebas de chat completion (streaming y no-streaming).
- Cliente Echo API para pruebas HTTP básicas.
- Validación de enrutamiento con requests OpenAI.

### Functions (Backend Mock)

**Directorio**: `functions/`

Backend simulado para pruebas de carga:
- `Health`: Endpoint de salud simple.
- `Profile`: Perfil de usuario de prueba con claves de prioridad.
- `Delay`/`StreamDelay`/`Novel`: Endpoints para validación de escenarios.

---

## Flujo Completo de una Solicitud

### Flujo Síncrono (Típico)

```
1. Cliente envía POST /chat/completions con header S7PPriority: 1
        │
2. Server.Run() acepta la conexión HTTP
        │
3. Se crea RequestData con prioridad 1, timestamp, UserID
        │
4. Se encola en ConcurrentPriQueue(priority=1, boost=UserBoost)
        │
5. ProxyWorker (asignado a P1) desencola la solicitud
        │
6. Backend Selection:
   a. Filtrar hosts por path (/chat → hosts con /chat registrado)
   b. Crear iterador (latency-based: Host A=45ms, Host B=120ms)
   c. Verificar Circuit Breaker de Host A → CERRADO ✓
        │
7. Proxy request a Host A (Azure OpenAI East US)
        │
8. Recibir respuesta streaming (SSE chunks)
        │
9. Extraer token metrics (prompt_tokens, completion_tokens)
        │
10. Stream response directamente al cliente HTTP
        │
11. Emitir ProxyEvent a App Insights con métricas completas
        │
12. Liberar RequestData y recursos
```

### Flujo Asíncrono (Long-Running)

```
1. Cliente envía POST con header async-config: enabled=true
        │
2. Server encola → ProxyWorker inicia procesamiento
        │
3. Tiempo de procesamiento excede AsyncThreshold
        │
4. AsyncWorker toma control:
   a. Retorna HTTP 202 Accepted al cliente
   b. Continúa procesamiento en background
        │
5. Respuesta completa → BlobWriter escribe a Azure Blob Storage
        │
6. ServiceBusRequestService envía notificación de estado
        │
7. RequestAPI (Azure Function) actualiza Cosmos DB
        │
8. Cliente consulta estado vía polling o recibe notificación por Service Bus
```

---

## Patrones de Diseño Utilizados

| Patrón | Implementación | Propósito |
|--------|---------------|-----------|
| **Factory** | `AsyncWorkerFactory`, `StreamProcessorFactory` | Crear workers/procesadores específicos por request |
| **Strategy** | `LoadBalanceMode` (latency/roundrobin/random) | Algoritmos de selección de host intercambiables |
| **Composite** | `CompositeEventClient` | Despacho a múltiples sinks de telemetría |
| **Observer** | `ProxyEvent` → `IEventClient` | Telemetría desacoplada |
| **Circuit Breaker** | `CircuitBreaker` | Prevención de fallos en cascada |
| **Priority Queue** | `ConcurrentPriQueue` | Scheduling justo con boost |
| **Object Pool** | `BlobWriter` container clients | Reutilización de recursos costosos |
| **State Machine** | `RequestLifecycleManager` | Transiciones de estado válidas |
| **Sidecar** | `HealthProbe` | Aislamiento de health checks |
| **Graceful Shutdown** | `CoordinatedShutdownService` | Apagado ordenado sin pérdida de datos |
| **Background Service** | `Server`, `ProxyWorkerCollection`, etc. | Procesamiento continuo en background |

---

## Integraciones con Azure

| Servicio Azure | Uso en SimpleL7Proxy |
|----------------|----------------------|
| **Azure OpenAI / AI Foundry** | Backend principal para modelos LLM |
| **Azure API Management (APIM)** | Gateway frontend con políticas |
| **Azure Container Apps (ACA)** | Plataforma de despliegue principal |
| **Azure Blob Storage** | Persistencia de requests/responses async |
| **Azure Service Bus** | Mensajería para requests async y notificaciones |
| **Azure Cosmos DB** | Tracking de estado de requests (vía RequestAPI) |
| **Application Insights** | Telemetría, métricas de tokens, trazas |
| **Azure Event Hub** | Streaming de eventos de alto volumen |
| **Microsoft Entra ID** | Managed Identity, OAuth2 |
| **Azure Key Vault** | (Implícito) Gestión de secretos |
| **Azure DNS** | Propagación para failover multi-región |

---

## Seguridad

- **VNET Injection**: Despliegue dentro de redes virtuales Azure.
- **Managed Identity**: Autenticación sin secretos hacia backends Azure.
- **OAuth2**: Validación de tokens en requests entrantes.
- **Header Policy Enforcement**: Validación/restricción de headers entrantes.
- **Zero Trust**: Headers permitidos/prohibidos configurables.
- **Configuración dinámica de acceso**: Suspensión de usuarios en tiempo real sin cambios de código.
- **Sovereign Cloud**: Soporte para GCC High y nubes gubernamentales.

---

## Observabilidad y Monitoreo

### Métricas capturadas

| Métrica | Descripción |
|---------|-------------|
| `S7P_RequestId` | ID único de solicitud |
| `BackendHost` | Host backend utilizado |
| `S7P_Priority` | Nivel de prioridad del request |
| `CircuitBreakerStatus` | Estado del circuit breaker |
| `Retries` | Número de reintentos |
| `Usage.Prompt_Tokens` | Tokens de prompt (OpenAI) |
| `Usage.Completion_Tokens` | Tokens de completado (OpenAI) |
| `QueueDuration` | Tiempo en cola |
| `ProcessDuration` | Tiempo de procesamiento |
| `BackendLatency` | Latencia del backend |

### Configuración de logging
- `LogAllRequestHeaders` / `LogAllResponseHeaders`: Control granular.
- `HeaderBlacklist`: Lista de headers a excluir del logging.
- Soporte para console, App Insights, y Event Hub simultáneamente.

---

## Escenarios de Despliegue

| Escenario | Configuración Clave |
|-----------|---------------------|
| **Mínimo** | Puerto + 2 hosts + polling básico |
| **Alta Disponibilidad** | Múltiples hosts + health checks estrictos + circuit breaker |
| **Seguridad** | Headers filtrados + OAuth + validación de auth |
| **Alto Rendimiento** | 50+ workers + cola larga + HTTP/2 |
| **Asíncrono** | Service Bus + Blob Storage configurados |
| **Multi-Región** | Hosts regionales + DNS refresh |
| **Container Apps** | Puerto 443 + managed logging |

---

## Configuración Detallada

### Formato de Backend Host (Recomendado)

```
Host1="host=https://my-openai.openai.azure.com;probe=/health;mode=apim;useoauth=true;processor=OpenAI;usemi=true"
```

| Parámetro | Descripción |
|-----------|-------------|
| `host` | URL del backend |
| `probe` | Ruta del health check |
| `mode` | `apim` (con APIM) o `direct` (sin probes) |
| `useoauth` | Autenticación OAuth2 |
| `processor` | `OpenAI` para extracción de tokens |
| `usemi` | Managed Identity |

### Variables de Entorno Críticas

| Variable | Descripción | Tipo |
|----------|-------------|------|
| `APPINSIGHTS_CONNECTIONSTRING` | Telemetría App Insights | COLD |
| `Host1` ... `Host9` | Backends configurados | DRAIN |
| `LoadBalanceMode` | Modo de balanceo | DRAIN |
| `WorkerCount` | Número de workers | COLD |
| `AsyncModeEnabled` | Habilitar modo async | COLD |
| `ServiceBusConnectionString` | Conexión Service Bus | COLD |
| `BlobStorageConnectionString` | Conexión Blob Storage | COLD |

---

## Pruebas

| Suite | Framework | Alcance |
|-------|-----------|---------|
| `SimpleL7Proxy.Test/` | MSTest | Tests unitarios del proxy |
| `test/health_poller/` | Especializado | Monitoreo de salud |
| `test/generator/` | Especializado | Generación de carga |
| `test/servicebus/` | Especializado | Integración Service Bus |

---

## Conclusión

SimpleL7Proxy es una pieza de infraestructura sofisticada que resuelve problemas reales de producción al operar modelos LLM a escala empresarial. Su arquitectura modular, con subsistemas claramente delimitados (cola, backend, proxy, async, usuarios, telemetría), permite una extensibilidad considerable mientras mantiene un rendimiento que soporta más de 23M tokens por minuto.

La combinación de priorización de solicitudes, circuit breaking, procesamiento asíncrono y observabilidad profunda lo posiciona como una alternativa seria a gateways comerciales, con la ventaja adicional de ser completamente self-hosted y personalizable.
