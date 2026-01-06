---
tags: [service-discovery, networking, microservices, dns, load-balancing, registration]
date: 2026-01-06
related: [Kubernetes, Container Orchestration, Networking, Microservices Architecture]
status: reference
---

# Service Discovery

## 📋 ¿Qué es Service Discovery?

El mecanismo para que **servicios encuentren y se conecten a otros servicios** automáticamente, sin hardcodear direcciones IP.

**Problema:** En microservicios, servicios aparecen y desaparecen constantemente (deployments, restarts, scaling). ¿Cómo sabe gateway dónde está auth-service?

**Analogía:** Registro de oficinas:
- ❌ Sin service discovery: "Auth-service está en 192.168.1.42:50051" → Cambio IP, gateway se rompe
- ✅ Con service discovery: "Necesito auth-service" → Sistema busca "service registry" → Retorna dirección actual

---

## 🎯 Problema que Resuelve

### Sin Service Discovery

```
Configuración hardcoded:
├─ gateway: AUTH_SERVICE_HOST=192.168.1.42:50051
├─ gateway: ORDER_SERVICE_HOST=192.168.1.43:5000
└─ gateway: PAYMENT_SERVICE_HOST=192.168.1.44:8080

Cuando auth-service se mueve (redeploy):
├─ IP cambia a 192.168.1.100
├─ Gateway sigue intentando 192.168.1.42
├─ Conexiones fallan
└─ Requiere redeploy de gateway (downtime)

Resultado: Frágil, propenso a fallos
```

### Con Service Discovery

```
Registro dinámico:
├─ auth-service: "Hola, soy auth-service en 192.168.1.42:50051"
├─ order-service: "Hola, soy order-service en 192.168.1.43:5000"
└─ payment-service: "Hola, soy payment-service en 192.168.1.44:8080"

Cuando auth-service se redeploya:
├─ Nuevo pod se levanta en 192.168.1.100:50051
├─ Se registra en service registry
├─ Viejo pod: desregistrado
├─ Gateway quiere conectar: "¿Dónde auth-service?"
├─ Registry responde: 192.168.1.100:50051
└─ Conexión automática, sin cambios

Resultado: Resiliente, automático
```

---

## 🏗️ Tipos de Service Discovery

### 1. **Client-Side Discovery**

```
Cliente (Gateway):
1. Consulta Service Registry: "¿Dónde está auth-service?"
2. Registry retorna: [192.168.1.42:50051, 192.168.1.43:50051]
3. Cliente elige una (load balancing)
4. Conecta a servidor seleccionado

Ventajas: Simple, control total del cliente
Desventajas: Cliente hace load balancing, lógica duplicada
```

### 2. **Server-Side Discovery**

```
Cliente (Gateway):
1. Conecta a Load Balancer: lb.service.internal:50051
2. Load Balancer consulta Service Registry
3. Load Balancer elige backend
4. Load Balancer forwards tráfico

Ventajas: Cliente simple, load balancing centralizado
Desventajas: Punto central de fallo
```

---

## 💻 Implementaciones

### 1. **Kubernetes (Built-in)**

```yaml
# Auth Service - Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: auth-service  # Label crítico
    spec:
      containers:
      - name: auth
        image: auth-service:1.0

---
# Auth Service - Service (Service Discovery)
apiVersion: v1
kind: Service
metadata:
  name: auth-service  # Nombre DNS: auth-service.default.svc.cluster.local
spec:
  selector:
    app: auth-service  # Selecciona Pods con este label
  ports:
  - port: 50051
    targetPort: 50051
  type: ClusterIP
```

En Kubernetes, cliente usa DNS:
```go
// gateway/internal/handlers/auth.go
// Kubernetes inyecta automáticamente:
// AUTH_SERVICE_HOST=auth-service.default.svc.cluster.local
// AUTH_SERVICE_PORT=50051

conn, _ := grpc.Dial(
    "auth-service.default.svc.cluster.local:50051",
    grpc.WithInsecure(),
)
```

### 2. **Consul (Standalone)**

```go
// Service Registration
package main

import (
    "github.com/hashicorp/consul/api"
)

func registerService() error {
    client, _ := api.NewClient(api.DefaultConfig())
    
    // Registrar este servicio
    registration := &api.AgentServiceRegistration{
        ID:      "auth-service-1",
        Name:    "auth-service",
        Port:    50051,
        Address: "192.168.1.42",
        Check: &api.AgentServiceCheck{
            HTTP:     "http://192.168.1.42:8080/health",
            Interval: "10s",
            Timeout:  "5s",
        },
    }
    
    return client.Agent().ServiceRegister(registration)
}

// Service Discovery
func discoverService(serviceName string) ([]string, error) {
    client, _ := api.NewClient(api.DefaultConfig())
    
    // Consultar servicio
    services, _, _ := client.Health().Service(serviceName, "", true, nil)
    
    var addresses []string
    for _, service := range services {
        addr := fmt.Sprintf("%s:%d", service.Service.Address, service.Service.Port)
        addresses = append(addresses, addr)
    }
    
    return addresses, nil
}

func main() {
    // Registrar
    registerService()
    
    // Descubrir
    addrs, _ := discoverService("auth-service")
    fmt.Println("Auth service instances:", addrs)
}
```

### 3. **Eureka (Netflix)**

```go
import "github.com/hudl/fargo"

func registerWithEureka() error {
    instance := &fargo.Instance{
        InstanceID: "auth-service-1",
        HostName:   "auth-service.example.com",
        App:        "AUTH-SERVICE",
        IPAddr:     "192.168.1.42",
        Port:       50051,
        Status:     fargo.UP,
        HomePageURL: "http://192.168.1.42:8080",
        HealthCheckURL: "http://192.168.1.42:8080/health",
    }
    
    conn, _ := fargo.NewConnWithPath("http://eureka-server:8080/eureka/")
    return conn.RegisterInstance(instance)
}

func discoverWithEureka(appName string) ([]*fargo.Instance, error) {
    conn, _ := fargo.NewConnWithPath("http://eureka-server:8080/eureka/")
    app, _ := conn.GetApp(appName)
    return app.Instances, nil
}
```

### 4. **DNS SRV Records**

```bash
# DNS SRV Record
_auth._tcp.service.consul.  IN SRV  0 0 50051 auth-1.node.consul.
_auth._tcp.service.consul.  IN SRV  0 0 50051 auth-2.node.consul.

# Resolver automáticamente encuentra todos
```

---

## 🎯 Tu Proyecto: Service Discovery

### Kubernetes + Gateway

```go
// gateway/internal/handlers/auth.go
package handlers

import (
    "fmt"
    "os"
    "google.golang.org/grpc"
    pb "github.com/myapp/auth-service/proto"
)

type AuthHandler struct {
    authClient pb.AuthServiceClient
}

func NewAuthHandler() *AuthHandler {
    // Service discovery: Kubernetes
    // Dirección: auth-service:50051 (Kubernetes DNS automático)
    authServiceHost := os.Getenv("AUTH_SERVICE_HOST")
    if authServiceHost == "" {
        authServiceHost = "auth-service:50051"
    }
    
    conn, err := grpc.Dial(
        authServiceHost,
        grpc.WithInsecure(),
        grpc.WithDefaultCallOptions(
            grpc.MaxCallRecvMsgSize(1024*1024*10), // 10MB
        ),
    )
    if err != nil {
        panic(err)
    }
    
    return &AuthHandler{
        authClient: pb.NewAuthServiceClient(conn),
    }
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    var req RegisterRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // Service discovery maneja la conexión
    user, err := h.authClient.Register(r.Context(), &pb.RegisterRequest{
        Email:    req.Email,
        Password: req.Password,
    })
    
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        return
    }
    
    w.WriteJSON(user)
}
```

### Con Load Balancer en Kubernetes

```yaml
# Kubernetes Service with load balancing
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  type: ClusterIP  # Internal load balancer
  selector:
    app: auth-service
  ports:
  - port: 50051
    targetPort: 50051
  sessionAffinity: ClientIP  # Sticky sessions (opcional)
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

Kubernetes automáticamente:
- Descubre 3 pods de auth-service
- Balancea tráfico entre ellos
- Si un pod falla, lo remueve del pool
- Si se levanta nuevamente, lo agrega

---

## 📊 Patrones de Service Discovery

### Pattern 1: DNS

```go
// Simple DNS lookup
func discoverServiceByDNS(serviceName string) (string, error) {
    addrs, err := net.LookupHost(serviceName)
    if err != nil {
        return "", err
    }
    
    if len(addrs) == 0 {
        return "", fmt.Errorf("no hosts found")
    }
    
    // En producción, implementar load balancing
    return addrs[0], nil
}
```

### Pattern 2: Client-Side Load Balancing

```go
import "google.golang.org/grpc"

// gRPC con client-side load balancing
func dialAuthService() (*grpc.ClientConn, error) {
    // Múltiples endpoints (descubiertos de registro)
    targets := []string{
        "auth-1.service.consul:50051",
        "auth-2.service.consul:50051",
        "auth-3.service.consul:50051",
    }
    
    // gRPC round-robin automático
    conn, err := grpc.Dial(
        fmt.Sprintf("///", targets...), // Round-robin resolver
        grpc.WithDefaultServiceConfig(`{
            "loadBalancingConfig": [{"round_robin":{}}]
        }`),
        grpc.WithInsecure(),
    )
    
    return conn, err
}
```

### Pattern 3: Health Check Integration

```go
// Service registry con health checks
type ServiceRegistry struct {
    client *api.Client
}

func (sr *ServiceRegistry) RegisterWithHealthCheck(
    serviceName string,
    addr string,
    port int,
    healthCheckURL string,
) error {
    registration := &api.AgentServiceRegistration{
        ID:      fmt.Sprintf("%s-%s", serviceName, addr),
        Name:    serviceName,
        Address: addr,
        Port:    port,
        Check: &api.AgentServiceCheck{
            HTTP:     healthCheckURL,
            Interval: "10s",  // Revisar cada 10s
            Timeout:  "5s",   // Timeout de 5s
            DeregisterCriticalServiceAfter: "30s", // Remover si 30s unhealthy
        },
    }
    
    return sr.client.Agent().ServiceRegister(registration)
}
```

---

## ⚡ Best Practices

✅ **Usa Kubernetes nativo si estás en K8s**
✅ **Implementa health checks**
✅ **Cachea respuestas de discovery localmente**
✅ **Reintentar con backoff exponencial**
✅ **Monitorea service registry**
✅ **Usa circuit breaker + service discovery**

---

## ⚠️ Antipatrones

❌ Hardcodear IPs en configuración
❌ Sin health checks
❌ Descubrir en cada request (muy lento)
❌ Ignorar cambios de topología
❌ Sin timeout en lookups
❌ Fallar si registry no responde (no tener fallbacks)

---

## 🔗 Service Discovery + Other Patterns

```
Service Discovery
├─ [[Kubernetes]] ← Native
├─ [[Load Balancing]] ← Distribuciones de tráfico
├─ [[Circuit Breaker Pattern]] ← Manejo de fallos
├─ [[Graceful Shutdown]] ← Deregistración limpia
└─ [[Observability]] ← Monitoreo de servicios
```

---

## 📚 Recursos

### Kubernetes
- Services: https://kubernetes.io/docs/concepts/services-networking/service/
- Service Discovery: https://kubernetes.io/docs/concepts/services-networking/

### Consul
- Service Discovery: https://www.consul.io/docs/discovery/dns
- Client API: https://pkg.go.dev/github.com/hashicorp/consul/api

### Otros
- Eureka: https://github.com/Netflix/eureka
- etcd: https://etcd.io/

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo hace tu gateway saber dónde está el auth-service?"

**Respuesta:**
> "Usaría service discovery. En Kubernetes, es automático: definí un Service resource que selecciona todos los Pods con label `app: auth-service`. Kubernetes automáticamente crea una entrada DNS: `auth-service.default.svc.cluster.local` que resuelve a las IPs de todos los Pods. En el gateway, conecto a ese hostname: `grpc.Dial('auth-service:50051')`. Kubernetes automáticamente balancea tráfico entre las 3 replicas. Si un Pod muere, Kubernetes lo remueve del pool de IPs. Si se levanta uno nuevo, lo agrega. Resultado: gateway nunca necesita saber IPs específicas. Si estuviera en ambiente standalone sin Kubernetes, usaría Consul: auth-service se registra al startup ('Hola, soy auth-service en 192.168.1.42:50051'), gateway consulta Consul ('¿Dónde auth-service?'), Consul retorna las instancias disponibles. Importante: health checks - Consul revisa periódicamente si auth-service responde, si no, lo remueve automáticamente. Sin service discovery, tendría que hardcodear direcciones IP en configuración del gateway, cada cambio requeriría redeploy."

---

#service-discovery #microservices #kubernetes #networking #devops
