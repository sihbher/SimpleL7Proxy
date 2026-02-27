# SimpleL7Proxy — Guía para Ingenieros Junior 🚀

## ¿Qué vas a aprender en este documento?

Este documento te explica, de manera clara y directa, qué es SimpleL7Proxy, qué problema resuelve, cómo funciona por dentro, y cómo podrías integrarlo con soluciones de Inteligencia Artificial. Está pensado para que lo entiendas aunque sea tu primer contacto con este tipo de tecnología.

---

## 1. ¿Qué es SimpleL7Proxy? (La Explicación Simple)

Imagina que tienes un **restaurante muy popular** (tu aplicación de AI) con **varios chefs** (backends de Azure OpenAI). Los clientes (usuarios) hacen pedidos (solicitudes HTTP) y necesitas alguien que:

- 🎫 **Decida quién va primero** → Los pedidos VIP se atienden antes que los normales.
- 🍳 **Elija al mejor chef disponible** → Si un chef está lento o enfermo, manda el pedido a otro.
- 🔁 **Reintente si algo sale mal** → Si un chef tira un plato, automáticamente pide otro.
- ⏱️ **Maneje pedidos largos** → Si un pedido tarda horas, no hace esperar al cliente en la barra.
- 📊 **Lleve registro de todo** → Cuántos ingredientes (tokens) usó cada plato.

**SimpleL7Proxy es ese "maître d'" inteligente**: un proxy de capa 7 (HTTP) que se coloca entre tus aplicaciones y los modelos de lenguaje (LLMs) de Azure OpenAI.

### En términos técnicos:

```
Tu App → [APIM] → [SimpleL7Proxy] → Azure OpenAI (múltiples regiones)
```

Es un **proxy inverso** construido en **.NET 9** que entiende el protocolo HTTP y puede tomar decisiones inteligentes sobre cada solicitud antes de enviarla al backend.

---

## 2. ¿Qué Problema Resuelve?

Cuando una empresa usa modelos de AI en producción, surgen problemas reales:

| Problema | Sin SimpleL7Proxy | Con SimpleL7Proxy |
|----------|--------------------|--------------------|
| Un usuario consume todo el recurso | ❌ Los demás esperan | ✅ Throttling por usuario, fair queuing |
| Un backend se cae | ❌ Errores para todos | ✅ Circuit breaker, failover automático |
| Solicitudes de chat y batch compiten | ❌ Chat se vuelve lento | ✅ Chat tiene prioridad sobre batch |
| No sabes cuántos tokens consumes | ❌ Facturas sorpresa | ✅ Métricas de tokens en tiempo real |
| Una solicitud tarda 2 horas | ❌ Timeout HTTP | ✅ Modo asíncrono con notificaciones |

---

## 3. Arquitectura — La Vista de 10,000 Pies

```
┌──────────────────┐
│   Tu Aplicación   │  (React, Python, C#, cualquier lenguaje)
└────────┬─────────┘
         │ HTTP POST /chat/completions
         ▼
┌──────────────────┐
│  Azure APIM       │  (Autenticación, Rate Limiting, Políticas)
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────────────┐
│            SimpleL7Proxy                      │
│                                               │
│  1. Recibe la solicitud                       │
│  2. Le asigna una prioridad                   │
│  3. La pone en una cola inteligente           │
│  4. Un worker la toma                         │
│  5. Elige el mejor backend disponible         │
│  6. Envía la solicitud y devuelve respuesta   │
│  7. Registra métricas de tokens               │
│                                               │
└──────┬──────────┬──────────┬─────────────────┘
       │          │          │
       ▼          ▼          ▼
   ┌───────┐ ┌───────┐ ┌───────┐
   │OpenAI │ │OpenAI │ │OpenAI │
   │East US│ │West EU│ │Japan  │
   └───────┘ └───────┘ └───────┘
```

---

## 4. Los 7 Componentes Clave (Lo que Necesitas Saber)

### 4.1 🚪 El Servidor HTTP (`server.cs`)

Es la "puerta de entrada". Escucha conexiones HTTP y crea un objeto `RequestData` por cada solicitud.

**Lo que hace**:
- Acepta la conexión.
- Lee el header `S7PPriority` para saber la prioridad (1 = alta, 3 = baja).
- Encola la solicitud.

**Analogía**: Es el recepcionista que toma tu nombre y te pone en la lista de espera.

### 4.2 📋 La Cola de Prioridad (`Queue/`)

Una cola donde las solicitudes se ordenan por importancia.

**Lo que hace**:
- Solicitudes de prioridad 1 siempre van primero.
- Dentro de la misma prioridad, los usuarios que han hecho menos solicitudes reciben un "boost".
- Los health checks (prioridad 0) tienen un fast-path especial sin locks.

**Analogía**: Como la fila del aeropuerto — hay fila prioritaria, económica, y una puerta especial para la tripulación.

### 4.3 ⚙️ Los Workers (`Proxy/`)

Son los "hilos de trabajo" que procesan solicitudes. Hay un pool configurable.

**Lo que hacen**:
1. Toman la siguiente solicitud de la cola.
2. Eligen un backend saludable.
3. Envían la solicitud al backend.
4. Devuelven la respuesta al cliente.

**Configuración típica**: 10-50 workers, distribuidos por prioridad (`"3,2,1"` = 3 workers para P1, 2 para P2, 1 para P3).

### 4.4 🏥 Backends y Circuit Breaker (`Backend/`)

Monitorea la salud de cada backend y decide cuál usar.

**Lo que hace**:
- **Polling**: Cada X segundos, hace un "ping" a cada backend.
- **Latencia**: Mide cuánto tarda cada backend en responder.
- **Circuit Breaker**: Si un backend falla mucho (>50 errores en 60s), lo "desconecta" temporalmente.

**Analogía**: Si un chef quema 50 platos seguidos, lo sacas de la cocina hasta que se recupere.

```
Backend A: ✅ Latencia 45ms  → ¡Usar este primero!
Backend B: ✅ Latencia 120ms → Backup
Backend C: ❌ CIRCUIT OPEN   → No enviar solicitudes
```

### 4.5 📨 El Sistema Asíncrono (`Feeder/`)

Para solicitudes que tardan mucho (embeddings masivos, batch processing).

**Flujo**:
```
1. Cliente envía solicitud → Proxy dice "202 Accepted, aquí tu ID"
2. Proxy procesa en background
3. Resultado se guarda en Azure Blob Storage
4. Notificación vía Azure Service Bus
5. Cliente consulta resultado cuando quiera
```

**Cuándo se activa**: Cuando el procesamiento excede un umbral configurable de tiempo.

### 4.6 👤 Perfiles de Usuario (`User/`)

Cada usuario puede tener configuración personalizada:
- **Prioridad**: Usuarios VIP con acceso prioritario.
- **Throttling**: Límites para evitar abusos.
- **Config async**: Container de blob y topic de Service Bus propios.
- **Suspensión**: Se puede desactivar un usuario sin reiniciar el servicio.

### 4.7 📊 Telemetría (`Events/`)

Todo queda registrado en Application Insights:
- Cuántos tokens usó cada solicitud (prompt + completion).
- Latencia de cada backend.
- Estado del circuit breaker.
- Errores y excepciones.

---

## 5. ¿Cómo se Despliega?

SimpleL7Proxy se ejecuta como un **contenedor Docker** en **Azure Container Apps (ACA)**:

```bash
# Construir la imagen
docker build -t simplel7proxy -f src/SimpleL7Proxy/Dockerfile .

# Desplegar con Azure Developer CLI
azd up
```

También incluye un **sidecar de health probe** (puerto 9000) que Kubernetes usa para saber si el proxy está vivo.

---

## 6. Integración con Soluciones de AI 🤖

Esta es la parte que más te interesa. SimpleL7Proxy está **diseñado específicamente** para ser el gateway de tus soluciones de AI.

### 6.1 Integración con Azure OpenAI (Lo Más Común)

**Escenario**: Tu aplicación usa GPT-4, GPT-4o, o cualquier modelo de Azure OpenAI.

**Configuración**:
```bash
Host1="host=https://mi-openai.openai.azure.com;processor=OpenAI;usemi=true;mode=direct"
```

**¿Qué ganas?**
- Balanceo de carga entre múltiples instancias de OpenAI.
- Circuit breaker si una región falla.
- Métricas de tokens automáticas.
- Priorización: chat interactivo > batch de embeddings.

**Ejemplo práctico**:
```python
# Tu código NO cambia. Solo apunta a SimpleL7Proxy en vez de directo a OpenAI.
import openai

client = openai.AzureOpenAI(
    azure_endpoint="https://mi-simplel7proxy.azurecontainerapps.io",  # ← Proxy
    api_key="tu-api-key",
    api_version="2024-02-01"
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hola mundo"}]
)
```

### 6.2 Integración con Azure AI Foundry

**Escenario**: Usas AI Foundry para desplegar modelos personalizados o modelos de la galería.

**Configuración**:
```bash
Host1="host=https://mi-ai-foundry.cognitiveservices.azure.com;processor=OpenAI;usemi=true"
```

**Modos de integración**:
- **Directo** (`mode=direct`): Sin health probes, ideal para endpoints serverless.
- **Via APIM** (`mode=apim`): Añade gobernanza, rate limiting, y scrubbing de PII.

### 6.3 Integración con LangChain / Semantic Kernel

**Escenario**: Estás construyendo un agente de AI con LangChain o Semantic Kernel.

```python
# LangChain - simplemente apunta el endpoint al proxy
from langchain_openai import AzureChatOpenAI

llm = AzureChatOpenAI(
    azure_endpoint="https://mi-simplel7proxy.azurecontainerapps.io",
    deployment_name="gpt-4o",
    api_version="2024-02-01",
)
```

```csharp
// Semantic Kernel - mismo concepto
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion(
        deploymentName: "gpt-4o",
        endpoint: "https://mi-simplel7proxy.azurecontainerapps.io",
        apiKey: "tu-api-key"
    )
    .Build();
```

**Beneficio**: Tu framework de AI obtiene automáticamente resiliencia, priorización y observabilidad sin cambiar una sola línea de lógica.

### 6.4 Escenario: Aplicación RAG (Retrieval-Augmented Generation)

Una aplicación RAG típica hace dos tipos de llamadas:

| Llamada | Prioridad | Por qué |
|---------|-----------|---------|
| Embedding de documentos (batch) | P3 (baja) | No es urgente, puede esperar |
| Chat del usuario (interactivo) | P1 (alta) | El usuario está esperando |

**Configuración en SimpleL7Proxy**:
```bash
# Tu app de batch envía header:
S7PPriority: 3

# Tu app de chat envía header:
S7PPriority: 1
```

**Resultado**: El chat del usuario NUNCA se ve afectado por el batch de embeddings, incluso cuando ambos comparten el mismo endpoint de OpenAI.

### 6.5 Escenario: Procesamiento Batch de AI (Async)

**Problema**: Necesitas procesar 10,000 documentos con AI, pero cada uno tarda 2 minutos.

**Solución con SimpleL7Proxy**:

```python
import requests

# 1. Enviar solicitud async
response = requests.post(
    "https://mi-proxy.azurecontainerapps.io/openai/deployments/gpt-4o/chat/completions",
    headers={
        "S7PPriority": "3",
        "async-config": "enabled=true,containername=mi-batch,topic=status-batch"
    },
    json={"messages": [{"role": "user", "content": "Resumir documento..."}]}
)

# 2. Recibir 202 Accepted con ID
request_id = response.headers["S7P_RequestId"]

# 3. Consultar resultado más tarde (o recibir notificación por Service Bus)
```

### 6.6 Escenario: Multi-Modelo / Multi-Región

```bash
# Modelo GPT-4o en East US (prioridad de latencia)
Host1="host=https://openai-eastus.openai.azure.com;processor=OpenAI;usemi=true"

# Modelo GPT-4o en West Europe (backup)
Host2="host=https://openai-westeu.openai.azure.com;processor=OpenAI;usemi=true"

# Modelo GPT-4 Turbo en Japan (overflow)
Host3="host=https://openai-japan.openai.azure.com;processor=OpenAI;usemi=true"

# Balanceo por latencia (usa el más rápido primero)
LoadBalanceMode=latency
```

---

## 7. ¿Cómo Empiezo? (Quick Start)

### Paso 1: Clonar y configurar

```bash
git clone <repo-url>
cd SimpleL7Proxy
# Ejecutar script de configuración interactivo
./local-setup.sh
```

### Paso 2: Configurar variables de entorno mínimas

```bash
export Port=5000
export Host1="host=https://mi-openai.openai.azure.com;processor=OpenAI;mode=direct"
export WorkerCount=5
export LoadBalanceMode=latency
```

### Paso 3: Ejecutar localmente

```bash
cd src/SimpleL7Proxy
dotnet run
```

### Paso 4: Probar

```bash
curl -X POST http://localhost:5000/openai/deployments/gpt-4o/chat/completions \
  -H "Content-Type: application/json" \
  -H "api-key: TU_API_KEY" \
  -d '{"messages": [{"role": "user", "content": "Hola"}]}'
```

---

## 8. Conceptos Clave que Debes Conocer

| Concepto | Qué es | Por qué importa |
|----------|--------|------------------|
| **Layer 7 Proxy** | Proxy que entiende HTTP (headers, paths, body) | Permite decisiones inteligentes por solicitud |
| **Circuit Breaker** | Patrón que "desconecta" backends con fallos | Evita enviar solicitudes a un servidor caído |
| **Priority Queue** | Cola donde items con mayor prioridad salen primero | Garantiza que el chat no espere por batch |
| **Load Balancing** | Distribuir carga entre múltiples servidores | Mejor rendimiento y resiliencia |
| **Streaming (SSE)** | Respuestas que llegan en "chunks" en tiempo real | Así funciona ChatGPT — token por token |
| **Managed Identity** | Autenticación sin contraseñas en Azure | Más seguro que API keys |
| **Async/Await** | Programación que no bloquea hilos | Permite manejar 1000+ solicitudes concurrentes |
| **Sidecar** | Servicio auxiliar que corre junto al principal | Aísla health checks del tráfico real |

---

## 9. Estructura de Archivos — Mapa Mental

```
Lo que DEBES mirar primero:
├── README.md                    ← Lee esto primero
├── src/SimpleL7Proxy/
│   ├── Program.cs               ← Punto de entrada (DI, startup)
│   ├── server.cs                ← Cómo recibe solicitudes HTTP
│   ├── Proxy/ProxyWorker.cs     ← Cómo procesa cada solicitud
│   ├── Queue/                   ← Cómo prioriza solicitudes
│   └── Backend/Backends.cs      ← Cómo elige el mejor backend
│
Lo que puedes mirar después:
├── src/SimpleL7Proxy/
│   ├── Config/BackendOptions.cs ← Todas las opciones configurables
│   ├── Events/ProxyEvent.cs     ← Cómo emite telemetría
│   ├── Feeder/AsyncFeeder.cs    ← Cómo maneja solicitudes largas
│   └── User/UserProfile.cs      ← Cómo gestiona usuarios
│
Documentación útil:
├── docs/
│   ├── CONFIGURATION_SETTINGS.md ← Referencia de configuración
│   ├── LOAD_BALANCING.md         ← Cómo funciona el balanceo
│   ├── CIRCUIT_BREAKER.md        ← Cómo funciona el circuit breaker
│   └── AI_FOUNDRY_INTEGRATION.md ← Integración con AI Foundry
```

---

## 10. Preguntas Frecuentes

### ¿Necesito cambiar mi código de aplicación?
**No.** Solo cambias la URL del endpoint de OpenAI para que apunte al proxy. Tu código de LangChain, Semantic Kernel, o SDK de OpenAI funciona sin cambios.

### ¿Funciona con streaming?
**Sí.** SimpleL7Proxy soporta SSE (Server-Sent Events) y además extrae métricas de tokens de los chunks de streaming en tiempo real.

### ¿Puedo usarlo fuera de Azure?
**Sí.** Aunque está optimizado para Azure, es un contenedor Docker estándar. Organizaciones lo han corrido en AWS y GCP exitosamente.

### ¿Es difícil de operar?
Depende del escenario. Para un setup básico (2 backends, balanceo simple), es bastante directo. Para escenarios avanzados (async, multi-región, perfiles de usuario), requiere más configuración pero la documentación es extensa.

### ¿Cuánto tráfico soporta?
El proxy ha sido probado manejando más de **23 millones de tokens por minuto (TPM)** y aproximadamente **1,000 tareas concurrentes** bajo carga normal.

---

## 11. Próximos Pasos Recomendados

1. **📖 Lee el README.md** del repositorio completo.
2. **🔧 Ejecuta localmente** con `local-setup.sh` y un backend mock.
3. **🧪 Prueba con el TestClient** incluido en el repo.
4. **📊 Configura Application Insights** para ver métricas de tokens.
5. **🏗️ Despliega en ACA** con `azd up` cuando estés listo.
6. **📚 Lee `docs/SCENARIOS.md`** para configuraciones por caso de uso.

---

> **Recuerda**: SimpleL7Proxy no reemplaza a Azure OpenAI ni a APIM. Es una **capa intermedia** que añade inteligencia, resiliencia y gobernanza a tus llamadas de AI. Piensa en él como el "sistema nervioso" de tu plataforma de AI empresarial. 🧠
