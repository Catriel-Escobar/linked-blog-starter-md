---
tags: [observability, tracing, debugging, microservices, opentelemetry, jaeger]
date: 2026-01-05
related: [Observability, Logging, Metrics, OpenTelemetry, Correlation IDs]
status: reference
---

# Distributed Tracing

## 📋 ¿Qué es Distributed Tracing?

Un sistema que **rastrea una solicitud mientras viaja a través de múltiples servicios**, capturando timing, errores y dependencias entre ellos.

**Analogía:** Un paquete en un servicio de envíos:
- 📦 Paquete sale del almacén (trace_id = ABC123)
- 🚛 Se mueve por diferentes etapas (spans)
- 📍 Cada etapa registra: cuándo llegó, cuánto tardó, si hay problemas
- 📊 Al final: puedes ver la ruta completa, dónde se tardó más, dónde falló

---

## 🎯 Problema que Resuelve

### Sin Tracing Distribuido

```
Usuario: "Mi solicitud es lenta (2 segundos)"

Sin tracing:
- ¿Dónde está el cuello de botella?
- ¿Es el gateway? ¿Auth service? ¿Database?
- ¿Qué servicios se llamaron?

Resultado: Horas investigando logs dispersos en múltiples máquinas
```

### Con Distributed Tracing

```
Usuario: "Mi solicitud es lenta (2 segundos)"

Con tracing:
┌─ Gateway (50ms)
│  └─ Auth Service call (100ms)
│     └─ Database query (80ms)
│     └─ Cache lookup (5ms)
├─ Order Service (900ms) ← CUELLO DE BOTELLA
│  └─ Stripe API call (850ms)
│  └─ Inventory update (30ms)
└─ Notification Service (50ms, asincrónico)

Resultado: 30 segundos para identificar que Stripe está lento
```

---

## 🏗️ Conceptos Clave

### Trace

Una solicitud completa desde entrada a salida.

```
trace_id = "abc123def456"
┌─────────────────────────────────────────┐
│ Trace: Procesar orden                   │
│ Duración total: 2000ms                  │
│ Status: OK                              │
└─────────────────────────────────────────┘
```

### Span

Una unidad de trabajo dentro de un trace.

```
span_id = "span1"
parent_span_id = "abc123def456"
operation_name = "auth.login"
duration = 100ms
status = OK
attributes:
  user_id = "user-123"
  service = "auth-service"
```

### Context Propagation

Pasar trace_id y span_id a través de servicios.

```
Request A:
┌─────────────────────────────┐
│ trace_id: abc123            │
│ span_id: span1              │
└──────────┬──────────────────┘
           │
           v
      Gateway
      ├─ Agrega header:
      │  traceparent: 00-abc123-span2-01
      │
      └─ Llama a Auth Service
           │
           v
      Auth Service
      ├─ Lee header
      ├─ trace_id: abc123 (mismo)
      ├─ parent_span_id: span2
      ├─ span_id: span3 (nuevo)
      │
      └─ Llama a Database
           │
           v
      Database
      ├─ trace_id: abc123
      ├─ parent_span_id: span3
      └─ span_id: span4
```

---

## 💻 Implementación: OpenTelemetry en Go

### 1. Configurar Tracer

```go
package main

import (
    "go.opentelemetry.io/api/global"
    "go.opentelemetry.io/api/trace"
    "go.opentelemetry.io/exporters/jaeger"
    "go.opentelemetry.io/sdk/resource"
    "go.opentelemetry.io/sdk/trace"
)

func initTracer() error {
    // Exportador a Jaeger
    exporter, err := jaeger.NewRawExporter(
        jaeger.WithAgentHost("jaeger"),
        jaeger.WithAgentPort(6831),
    )
    if err != nil {
        return err
    }
    
    // Crear trace provider
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            "service.name", "gateway",
            "service.version", "1.0.0",
        )),
    )
    
    // Registrar globalmente
    global.SetTracerProvider(tp)
    
    return nil
}
```

### 2. Crear Spans en HTTP Handler

```go
import (
    "go.opentelemetry.io/api/global"
    "go.opentelemetry.io/api/attribute"
    "go.opentelemetry.io/api/codes"
)

type AuthHandler struct {
    tracer trace.Tracer
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    // Extraer contexto (si viene de otro servicio)
    ctx := r.Context()
    
    // Crear span
    ctx, span := h.tracer.Start(ctx, "http.login_handler")
    defer span.End()
    
    // Agregar atributos
    span.SetAttributes(
        attribute.String("http.method", r.Method),
        attribute.String("http.url", r.URL.String()),
        attribute.String("http.client_ip", r.RemoteAddr),
    )
    
    // Parsear body
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "failed to parse request")
        w.WriteHeader(http.StatusBadRequest)
        return
    }
    
    span.SetAttributes(
        attribute.String("user.email", req.Email),
    )
    
    // Llamar servicio de auth (con span anidado automático)
    user, err := h.authenticateUser(ctx, req)
    
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "authentication failed")
        w.WriteHeader(http.StatusUnauthorized)
        return
    }
    
    span.SetAttributes(
        attribute.String("user.id", user.ID),
    )
    span.SetStatus(codes.Ok, "")
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func (h *AuthHandler) authenticateUser(ctx context.Context, req LoginRequest) (*User, error) {
    ctx, span := h.tracer.Start(ctx, "auth.authenticate")
    defer span.End()
    
    // Logica de autenticación
    return h.authService.Login(ctx, req.Email, req.Password)
}
```

### 3. Middleware para HTTP Tracing

```go
func TracingMiddleware(tracer trace.Tracer) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extraer trace context del header (si viene de otro servicio)
            ctx := r.Context()
            
            // Si no hay trace, crear nuevo
            ctx, span := tracer.Start(ctx, r.URL.Path)
            defer span.End()
            
            // Agregar atributos HTTP
            span.SetAttributes(
                attribute.String("http.method", r.Method),
                attribute.String("http.url", r.URL.String()),
                attribute.Int("http.scheme", parseScheme(r.URL.Scheme)),
                attribute.String("http.client_ip", r.RemoteAddr),
            )
            
            // Crear wrapper para capturar status code
            wrapped := &responseWriter{ResponseWriter: w}
            
            // Pasar request con contexto actualizado
            next.ServeHTTP(wrapped, r.WithContext(ctx))
            
            // Agregar status code al span
            span.SetAttributes(
                attribute.Int("http.status_code", wrapped.statusCode),
            )
        })
    }
}
```

### 4. Propagación de Contexto (gRPC)

```go
import (
    "go.opentelemetry.io/instrumentation/google.golang.org/grpc"
)

// Cliente gRPC con tracing automático
conn, _ := grpc.NewClient(
    "auth-service:50051",
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)

// Servidor gRPC con tracing automático
lis, _ := net.Listen("tcp", ":50051")
s := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
```

---

## 📊 Visualización en Jaeger

```
Trace: abc123def456
├─ Gateway /api/orders (2000ms) ✓
│  ├─ Validate input (10ms) ✓
│  ├─ Auth Service login (100ms) ✓
│  │  ├─ Hash password (20ms) ✓
│  │  ├─ Database query (60ms) ✓
│  │  └─ Generate JWT (10ms) ✓
│  ├─ Order Service create (900ms) ✗ ERROR
│  │  ├─ Validate inventory (50ms) ✓
│  │  ├─ Payment Service charge (800ms) ✗ TIMEOUT
│  │  │  └─ Stripe API (800ms) ✗ TIMEOUT
│  │  └─ Database insert (0ms) ✗ CANCELLED
│  └─ Response (5ms) ✓
└─ Total: 2000ms
```

---

## 🔗 Correlation IDs (Request Tracking)

```go
import "github.com/google/uuid"

// Middleware para agregar request ID
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        
        if requestID == "" {
            requestID = uuid.New().String()
        }
        
        ctx := context.WithValue(r.Context(), "request_id", requestID)
        
        // Agregar al response header
        w.Header().Set("X-Request-ID", requestID)
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Usar en logs
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    requestID := r.Context().Value("request_id").(string)
    
    logger.WithFields(logrus.Fields{
        "request_id": requestID,
        "user_id":    userID,
    }).Info("User retrieved")
}
```

---

## 🎯 Tu Proyecto: Agregar Tracing

### Gateway con Tracing

```go
// gateway/internal/handlers/auth.go
type AuthHandler struct {
    authClient pb.AuthServiceClient
    tracer     trace.Tracer
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    ctx, span := h.tracer.Start(r.Context(), "auth.register")
    defer span.End()
    
    var req RegisterRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "parse error")
        w.WriteHeader(http.StatusBadRequest)
        return
    }
    
    span.SetAttributes(
        attribute.String("user.email", req.Email),
    )
    
    // Llamar auth service (contexto se propaga automáticamente)
    user, err := h.authClient.Register(ctx, &pb.RegisterRequest{
        Email:    req.Email,
        Password: req.Password,
    })
    
    if err != nil {
        span.RecordError(err)
        w.WriteHeader(http.StatusInternalServerError)
        return
    }
    
    span.SetAttributes(
        attribute.String("user.id", user.Id),
    )
    
    w.WriteJSON(user)
}
```

### Auth Service con Tracing

```go
// auth-service/internal/service/auth_service.go
type AuthService struct {
    userRepo UserRepository
    tracer   trace.Tracer
}

func (s *AuthService) Register(ctx context.Context, email, password string) (*User, error) {
    ctx, span := s.tracer.Start(ctx, "auth.register_service")
    defer span.End()
    
    // Validar
    if err := s.validateEmail(email); err != nil {
        span.RecordError(err)
        return nil, err
    }
    
    // Hash password (con span anidado)
    ctx2, span2 := s.tracer.Start(ctx, "password.hash")
    hashedPassword, err := hashPassword(password)
    span2.End()
    
    if err != nil {
        span.RecordError(err)
        return nil, err
    }
    
    // Crear usuario (con span anidado)
    ctx3, span3 := s.tracer.Start(ctx, "database.insert")
    user, err := s.userRepo.Create(ctx3, &User{
        Email:    email,
        Password: hashedPassword,
    })
    span3.End()
    
    if err != nil {
        span.RecordError(err)
        return nil, err
    }
    
    span.SetAttributes(
        attribute.String("user.id", user.ID),
    )
    
    return user, nil
}
```

---

## 📊 Stack de Tracing

```
OpenTelemetry (librería)
    │
    ├─ Jaeger (backend)
    │  └─ Almacena traces
    │
    ├─ Zipkin (alternativa)
    │  └─ Almacena traces
    │
    └─ OTLP Collector (collector)
       └─ Recibe de múltiples fuentes
          ├─ Jaeger
          ├─ Prometheus
          ├─ Loki
```

---

## ⚡ Best Practices

✅ **Samplea inteligentemente** (no traces de TODO)
✅ **Agrupa por trace_id** en logs/métricas
✅ **Propaga contexto** automáticamente
✅ **Nombra spans descriptivamente** (usa puntos: auth.login)
✅ **Agrega atributos relevantes** (user_id, order_id)
✅ **Registra errores** (span.RecordError())
✅ **Monitorea latencia P99** en spans críticos

---

## ⚠️ Antipatrones

❌ Tracing de TODO (performance hit)
❌ Perder contexto entre servicios
❌ Spans sin atributos contextuales
❌ No samplear (flooding backend)
❌ Exponer trace data al cliente

---

## 📚 Recursos

### OpenTelemetry
- https://opentelemetry.io
- Go instrumentation: https://github.com/open-telemetry/opentelemetry-go

### Backends
- Jaeger: https://www.jaegertracing.io
- Zipkin: https://zipkin.io
- DataDog: https://www.datadoghq.com

### Tutoriales
- "Distributed Tracing in Go" - Medium
- OpenTelemetry Go tutorial: https://opentelemetry.io/docs/instrumentation/go/

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo debuggearías una solicitud lenta en un sistema de microservicios?"

**Respuesta:**
> "Usaría distributed tracing con OpenTelemetry y Jaeger. Cada solicitud tiene un trace_id único que se propaga a través de todos los servicios. Cuando un usuario reporta 'solicitud lenta', busco su trace_id en Jaeger y veo: Gateway (50ms) → Auth Service (100ms) → Order Service (900ms) → Payment Service (800ms timeout). Inmediatamente veo que Stripe API es el cuello de botella. El trace muestra también spans anidados: exactamente qué queries de DB tardaron más, qué llamadas externas fallaron, dónde hay timeouts. Esto transforma debugging de 'algo está lento' a 'Stripe API timeout, implementar circuit breaker' en minutos."

---

#distributed-tracing #observability #opentelemetry #jaeger #microservices
