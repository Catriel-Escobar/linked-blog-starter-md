---
tags: [microservices, infrastructure, networking, kubernetes]
date: 2026-01-05
related: [Microservices, Kubernetes, Istio, Load Balancing]
status: reference
---

# Service Mesh

## 📋 ¿Qué es un Service Mesh?

Una **capa de infraestructura** dedicada para manejar la comunicación servicio-a-servicio en arquitecturas de microservicios, de forma transparente y sin modificar el código de la aplicación.

**Analogía:** Es como tener un "asistente de comunicaciones" para cada microservicio que maneja todo el networking, seguridad, y observabilidad automáticamente.

---

## 🎯 Problema que Resuelve

### Sin Service Mesh
```
Auth Service ──────────────→ Order Service
    ↓ Implementa:
    - Retry logic
    - Timeouts
    - Circuit breakers
    - Load balancing
    - mTLS
    - Metrics
    - Tracing
    
    ❌ Cada servicio duplica lógica
    ❌ 10 lenguajes = 10 implementaciones
    ❌ Cambio = recompilar todos
```

### Con Service Mesh
```
Auth Service ──→ [Sidecar Proxy] ──→ [Sidecar Proxy] ──→ Order Service
                       ↓                      ↓
                  Maneja todo:
                  ✅ Retry, timeout, circuit breaker
                  ✅ Load balancing
                  ✅ mTLS automático
                  ✅ Metrics + tracing
                  
    ✅ Código de app simple
    ✅ Configuración centralizada
    ✅ Funciona con cualquier lenguaje
```

---

## 🏗️ Arquitectura

### Control Plane vs Data Plane

```
┌─────────────────────────────────────────────────┐
│           Control Plane (Cerebro)               │
│  - Configuración centralizada                   │
│  - Políticas de seguridad                       │
│  - Telemetría agregada                          │
│  - Service discovery                            │
└────────────────┬────────────────────────────────┘
                 │ Configura
                 ↓
┌─────────────────────────────────────────────────┐
│           Data Plane (Manos)                    │
│                                                  │
│  Service A ←→ [Proxy] ←→ [Proxy] ←→ Service B  │
│                 ↑                    ↑           │
│              Sidecar              Sidecar        │
│                                                  │
└─────────────────────────────────────────────────┘
```

### Sidecar Pattern

```yaml
# Pod en Kubernetes
Pod: order-service
  ├── Container: order-service (tu app)
  └── Container: envoy-proxy (sidecar)
       ↑ Intercepta todo el tráfico
```

**Flujo:**
```
1. order-service hace request a auth-service:50051
2. Request es interceptado por envoy-proxy (sidecar)
3. Envoy aplica: retry, timeout, load balancing, mTLS
4. Envoy rutea a auth-service (a través de su sidecar)
5. Response regresa por el mismo camino
```

---

## 🔧 Funcionalidades Principales

### 1. **Traffic Management**

**Load Balancing:**
```yaml
# Distribuir tráfico entre instancias
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: auth-service
spec:
  hosts:
  - auth-service
  http:
  - route:
    - destination:
        host: auth-service
        subset: v1
      weight: 90  # 90% a v1
    - destination:
        host: auth-service
        subset: v2
      weight: 10  # 10% a v2 (canary)
```

**Circuit Breaker:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: auth-service
spec:
  host: auth-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

**Timeouts & Retries:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: auth-service
spec:
  http:
  - timeout: 500ms
    retries:
      attempts: 3
      perTryTimeout: 100ms
      retryOn: 5xx,reset,connect-failure
```

### 2. **Security**

**mTLS Automático:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT  # Todos los servicios usan mTLS
```

**Autorización:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-gateway
spec:
  selector:
    matchLabels:
      app: auth-service
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/gateway"]
```

### 3. **Observability**

**Métricas automáticas:**
- Request rate (requests/second)
- Error rate (%)
- Latency (p50, p95, p99)
- Success rate

**Distributed Tracing:**
```
Gateway (span 1)
    └─> Auth Service (span 2)
            └─> DB Query (span 3)
```

**Service Graph:**
```
        ┌─────────┐
        │ Gateway │
        └────┬────┘
             │
    ┌────────┴────────┐
    ↓                 ↓
┌────────┐      ┌──────────┐
│  Auth  │      │  Orders  │
└────┬───┘      └─────┬────┘
     │                │
     └────────┬───────┘
              ↓
         ┌────────┐
         │   DB   │
         └────────┘
```

---

## 🌟 Service Mesh Populares

### 1. **Istio** (Más Popular)
- ✅ Feature-rich
- ✅ Usa Envoy proxy
- ✅ Gran ecosistema
- ⚠️ Complejo de configurar
- ⚠️ Overhead alto

### 2. **Linkerd**
- ✅ Ligero y rápido
- ✅ Fácil de usar
- ✅ Rust-based proxy
- ⚠️ Menos features que Istio

### 3. **Consul Connect** (HashiCorp)
- ✅ Service discovery integrado
- ✅ Multi-cloud
- ⚠️ Ecosistema más pequeño

### 4. **AWS App Mesh**
- ✅ Integración con AWS
- ✅ Usa Envoy
- ⚠️ Solo AWS

---

## 📊 Comparación: Con vs Sin Service Mesh

| Feature | Sin Service Mesh | Con Service Mesh |
|---------|-----------------|------------------|
| **Retry Logic** | Código en cada servicio | Configuración YAML |
| **Load Balancing** | DNS round-robin | Intelligent (latency-aware) |
| **Circuit Breaker** | Implementar en código | Configuración |
| **mTLS** | Manual (certificados) | Automático |
| **Observability** | Instrumentar código | Automático |
| **Canary Deploy** | Complejo | Configuración simple |
| **Cambios** | Recompilar + redeploy | Solo config |

---

## 🚀 Cuándo Usar Service Mesh

### ✅ Usar Service Mesh si:
- Tienes **muchos microservicios** (10+)
- Necesitas **seguridad estricta** (mTLS, autorización)
- Requieres **observabilidad profunda**
- Deployments **complejos** (canary, blue-green)
- **Multi-lenguaje** (Go, Java, Python, etc.)
- Ya usas **Kubernetes**

### ❌ NO usar Service Mesh si:
- Aplicación **monolítica** o pocos servicios (<5)
- **Recursos limitados** (overhead de sidecars)
- Equipo **pequeño** (complejidad operacional)
- **No usas Kubernetes** (más difícil)

---

## 💡 Ejemplo con Istio

### Setup Básico

**1. Instalar Istio:**
```bash
istioctl install --set profile=demo
kubectl label namespace default istio-injection=enabled
```

**2. Deploy con sidecar automático:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  template:
    metadata:
      labels:
        app: auth-service
        version: v1
    spec:
      containers:
      - name: auth-service
        image: auth-service:latest
        ports:
        - containerPort: 50051
```

Istio inyecta automáticamente el sidecar proxy.

**3. Configurar timeout:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: auth-service
spec:
  hosts:
  - auth-service
  http:
  - timeout: 500ms
    route:
    - destination:
        host: auth-service
```

---

## 🔗 Integración con tu Proyecto

### Cómo se vería tu arquitectura con Service Mesh:

```
Cliente HTTP
    ↓
Gateway Pod
├── gateway-service (container)
└── envoy-proxy (sidecar)
    ↓ mTLS, retry, timeout
Auth Service Pod
├── auth-service (container)
└── envoy-proxy (sidecar)
    ↓ connection pooling
PostgreSQL
```

### Ventajas para tu caso:
1. **Eliminar middleware de timeout** → manejado por Istio
2. **mTLS automático** → Gateway ↔ Auth Service cifrado
3. **Métricas gratis** → request rate, latency, errors
4. **Circuit breaker** → proteger Auth Service de sobrecarga
5. **Canary deploys** → testear v2 con 10% de tráfico

---

## 📚 Recursos

### Documentación
- Istio: https://istio.io
- Linkerd: https://linkerd.io
- Envoy Proxy: https://envoyproxy.io

### Tutoriales
- Istio by Example: https://istiobyexample.dev
- Service Mesh Patterns: https://servicemeshpatterns.com

### Videos
- "What is a Service Mesh?" - IBM Technology (YouTube)
- "Istio in 5 Minutes" - TechWorld with Nana

---

## 🎯 Próximos Pasos

### Nivel Básico
- [ ] Entender Sidecar Pattern
- [ ] Instalar Istio localmente (minikube)
- [ ] Desplegar app simple con mTLS

### Nivel Intermedio
- [ ] Configurar circuit breakers
- [ ] Implementar canary deployments
- [ ] Dashboard de observabilidad (Kiali)

### Nivel Avanzado
- [ ] Multi-cluster service mesh
- [ ] Service mesh federation
- [ ] Custom policies

---

## 🔑 Conceptos Clave

- **Sidecar Proxy**: Container adicional que intercepta tráfico
- **Control Plane**: Cerebro que configura los proxies
- **Data Plane**: Proxies que manejan el tráfico real
- **mTLS**: Mutual TLS, encriptación automática
- **Envoy**: Proxy de alto performance (C++)
- **Service Discovery**: Encontrar servicios dinámicamente

---

## 💼 En Entrevistas

**Pregunta:** "¿Qué es un service mesh y por qué lo usarías?"

**Respuesta:**
> "Un service mesh es una capa de infraestructura que maneja la comunicación entre microservicios de forma transparente usando sidecar proxies. Lo usaría cuando tengo múltiples microservicios y necesito funcionalidades como circuit breakers, retry logic, mTLS, y observabilidad sin modificar el código de cada servicio. Por ejemplo, con Istio puedo configurar timeouts y retries con YAML en lugar de implementarlo en cada servicio, lo que facilita el mantenimiento y funciona con cualquier lenguaje."

---

#service-mesh #istio #kubernetes #microservices #infrastructure #envoy
