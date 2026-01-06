---
tags: [observability, monitoring, logging, metrics, tracing, devops]
date: 2026-01-05
related: [Error Handling Patterns, Status Codes, Prometheus, OpenTelemetry]
status: reference
---

# Observability

## 📋 ¿Qué es Observability?

La **capacidad de entender el estado interno de un sistema** observando sus salidas (logs, métricas, traces), sin necesidad de conocer su implementación interna.

**Las 3 pilares:**
- 📝 **Logging:** Eventos específicos que ocurrieron
- 📊 **Metrics:** Medidas numéricas (contador, gauge, histogram)
- 🔗 **Tracing:** Flujo de una solicitud a través del sistema

**Analogía:** Un cuerpo humano:
- 📝 Logs = Diario de eventos (me golpeé, comí, dormí)
- 📊 Metrics = Signos vitales (90 bpm, 37°C, 120/80 mmHg)
- 🔗 Tracing = Seguimiento de comida: boca → esófago → estómago → intestinos

---

## 🎯 Problema que Resuelve

### Sin Observabilidad

```
Usuario reporta: "La app no responde"

Sin logs:
- ¿Dónde falló? No sé
- ¿Cuándo falló? No sé
- ¿Por qué? No sé

Resultado: 2 horas debugging a ciegas
```

### Con Observabilidad

```
Usuario reporta: "La app no responde"

Con logs:
[2026-01-05 15:30:45] ERROR OrderService: Database connection timeout
[2026-01-05 15:30:46] ERROR PaymentService: Auth service unavailable

Con métricas:
- Database latency: 5000ms (normal: 50ms)
- PaymentService error rate: 100% (normal: 0.1%)

Con tracing:
OrderService → (5ms) → AuthService → (2000ms timeout) → Error

Resultado: 5 minutos para identificar el problema
```

---

## 📝 Logging

### Niveles de Log

```
DEBUG    - Información detallada para debugging
INFO     - Eventos normales del sistema
WARN     - Situaciones anómalas pero manejables
ERROR    - Errores que requieren atención
FATAL    - Error crítico, aplicación puede crashear
PANIC    - Error catastrófico, aplicación crashea
```

### Estructura de Log

```go
// ❌ Log poco estructurado
log.Printf("User registration failed")

// ✅ Log estructurado (máquina-legible)
log.WithFields(map[string]interface{}{
    "event":     "user_registration",
    "user_id":   "user-123",
    "email":     "test@test.com",
    "error":     "email_already_exists",
    "timestamp": "2026-01-05T15:30:45Z",
}).Error("User registration failed")

// JSON output:
{
  "level": "error",
  "event": "user_registration",
  "user_id": "user-123",
  "email": "test@test.com",
  "error": "email_already_exists",
  "timestamp": "2026-01-05T15:30:45Z",
  "message": "User registration failed"
}
```

### Implementación en Go

```go
import "github.com/sirupsen/logrus"

// Configurar logger
logger := logrus.New()
logger.SetFormatter(&logrus.JSONFormatter{})
logger.SetLevel(logrus.DebugLevel)

// Usar
logger.WithFields(logrus.Fields{
    "user_id": "123",
    "action":  "login",
}).Info("User logged in successfully")

logger.WithFields(logrus.Fields{
    "user_id": user.ID,
    "error":   err.Error(),
    "attempt": attempt,
}).Warn("Login failed, retrying")

logger.WithFields(logrus.Fields{
    "user_id": "123",
    "error":   err.Error(),
    "action":  "registration",
}).Error("User registration failed")
```

---

## 📊 Metrics (Prometheus)

### Tipos de Métricas

#### 1. **Counter** (Contador)

```go
// Siempre sube
httpRequestsTotal := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    },
    []string{"method", "status"},
)

// Usar
httpRequestsTotal.WithLabelValues("GET", "200").Inc()
httpRequestsTotal.WithLabelValues("POST", "400").Inc()
httpRequestsTotal.WithLabelValues("GET", "500").Inc()

// Resultado: {method="GET", status="200"} = 1
//           {method="POST", status="400"} = 1
//           {method="GET", status="500"} = 1
```

#### 2. **Gauge** (Medida)

```go
// Puede subir o bajar
activeDatabaseConnections := prometheus.NewGauge(
    prometheus.GaugeOpts{
        Name: "database_connections_active",
        Help: "Active database connections",
    },
)

// Usar
activeDatabaseConnections.Set(10)
activeDatabaseConnections.Inc()   // Sube a 11
activeDatabaseConnections.Dec()   // Baja a 10
```

#### 3. **Histogram** (Distribución)

```go
// Medir latencia
requestDuration := prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "request_duration_seconds",
        Help:    "Request latency in seconds",
        Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1, 2, 5},
    },
    []string{"endpoint"},
)

// Usar
start := time.Now()
// ... procesar request
duration := time.Since(start).Seconds()
requestDuration.WithLabelValues("/api/users").Observe(duration)

// Resultado: histograma de latencias
// le_0.01: 0 (0% tardó menos de 10ms)
// le_0.05: 45 (45% tardó menos de 50ms)
// le_0.1: 89 (89% tardó menos de 100ms)
// le_0.5: 95 (95% tardó menos de 500ms)
// le_1: 99 (99% tardó menos de 1s)
// le_+Inf: 100 (100% tardó menos de infinito)
```

### Ejemplo Completo

```go
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
    "time"
)

// Definir métricas
var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request latency",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
    
    activeRequests = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "http_requests_active",
            Help: "Active HTTP requests",
        },
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, requestDuration, activeRequests)
}

// Middleware para recolectar métricas
func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        activeRequests.Inc()
        defer activeRequests.Dec()
        
        start := time.Now()
        
        // Usar response writer wrapper para capturar status
        wrapped := &responseWriter{ResponseWriter: w}
        next.ServeHTTP(wrapped, r)
        
        duration := time.Since(start).Seconds()
        
        httpRequestsTotal.WithLabelValues(
            r.Method, r.URL.Path, string(rune(wrapped.status))).Inc()
        
        requestDuration.WithLabelValues(
            r.Method, r.URL.Path).Observe(duration)
    })
}

func main() {
    // Endpoint /metrics retorna métricas en formato Prometheus
    http.Handle("/metrics", promhttp.Handler())
    
    http.Handle("/", metricsMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("Hello"))
    })))
    
    http.ListenAndServe(":8080", nil)
}
```

---

## 🔗 Tracing (OpenTelemetry)

### Distributed Tracing

```
Request A:
    trace_id = abc123

Auth Service
    span_id = span1 (duration: 50ms)
    ├─ Database query
    │   span_id = span1a (duration: 20ms)
    └─ Cache lookup
        span_id = span1b (duration: 5ms)
        
Order Service
    span_id = span2 (duration: 100ms)
    ├─ Validation
    │   span_id = span2a (duration: 10ms)
    ├─ Payment Service call
    │   span_id = span2b (duration: 80ms)
    │   ├─ Stripe API call
    │   │   span_id = span2b1 (duration: 70ms)
    │   └─ Response handling
    │       span_id = span2b2 (duration: 5ms)
    └─ Database insert
        span_id = span2c (duration: 5ms)

Total trace: 150ms
```

### En Go con OpenTelemetry

```go
import (
    "go.opentelemetry.io/api/global"
    "go.opentelemetry.io/api/trace"
)

// Obtener tracer
tracer := global.Tracer("my-app")

// Crear span
ctx, span := tracer.Start(ctx, "process_order")
defer span.End()

// Agregar atributos
span.SetAttributes(
    attribute.String("order_id", order.ID),
    attribute.Int64("amount", order.Amount),
)

// Anidar spans
ctx2, span2 := tracer.Start(ctx, "validate_order")
defer span2.End()

// Si hay error
if err != nil {
    span2.RecordError(err)
    span2.SetStatus(codes.Error, "validation failed")
}
```

---

## 🎯 Observabilidad en Microservicios

```
┌──────────────────────────────────────────────┐
│ Observability Stack                          │
├──────────────────────────────────────────────┤
│                                              │
│  Prometheus (metrics)                        │
│  Loki (logs)                                 │
│  Jaeger (traces)                             │
│  Grafana (visualización)                     │
│                                              │
├──────────────────────────────────────────────┤
│ Tu Aplicación                                │
│                                              │
│  Gateway                Auth Service         │
│  ├─ Logs    ─────────→ Loki                  │
│  ├─ Metrics ─────────→ Prometheus            │
│  └─ Traces  ─────────→ Jaeger                │
│                                              │
│  Order Service        Payment Service        │
│  ├─ Logs    ─────────→ Loki                  │
│  ├─ Metrics ─────────→ Prometheus            │
│  └─ Traces  ─────────→ Jaeger                │
│                                              │
└──────────────────────────────────────────────┘
```

---

## 📝 Tu Proyecto: Agregar Observabilidad

### 1. Logging Estructurado

```go
// gateway/internal/handlers/auth.go
import "github.com/sirupsen/logrus"

type AuthHandler struct {
    authClient pb.AuthServiceClient
    logger     *logrus.Logger
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    h.logger.WithFields(logrus.Fields{
        "endpoint": "/auth/login",
        "ip":       r.RemoteAddr,
    }).Info("Login attempt started")
    
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.logger.WithFields(logrus.Fields{
            "error": err.Error(),
            "type":  "parse_error",
        }).Warn("Failed to parse login request")
        
        w.WriteHeader(http.StatusBadRequest)
        return
    }
    
    user, err := h.authClient.Login(r.Context(), &pb.LoginRequest{
        Email:    req.Email,
        Password: req.Password,
    })
    
    if err != nil {
        h.logger.WithFields(logrus.Fields{
            "email": req.Email,
            "error": err.Error(),
            "type":  "auth_error",
        }).Error("Login failed")
        
        w.WriteHeader(http.StatusUnauthorized)
        return
    }
    
    h.logger.WithFields(logrus.Fields{
        "user_id": user.Id,
        "email":   req.Email,
    }).Info("Login successful")
    
    w.WriteJSON(user)
}
```

### 2. Métricas Prometheus

```go
// gateway/internal/metrics/metrics.go
import "github.com/prometheus/client_golang/prometheus"

var (
    LoginAttempts = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "auth_login_attempts_total",
            Help: "Total login attempts",
        },
        []string{"status"},  // success, failure
    )
    
    LoginDuration = prometheus.NewHistogram(
        prometheus.HistogramOpts{
            Name: "auth_login_duration_seconds",
            Help: "Login request duration",
            Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1},
        },
    )
)

// gateway/internal/handlers/auth.go
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    
    user, err := h.authClient.Login(r.Context(), req)
    
    duration := time.Since(start).Seconds()
    
    if err != nil {
        metrics.LoginAttempts.WithLabelValues("failure").Inc()
        metrics.LoginDuration.Observe(duration)
        w.WriteHeader(http.StatusUnauthorized)
        return
    }
    
    metrics.LoginAttempts.WithLabelValues("success").Inc()
    metrics.LoginDuration.Observe(duration)
    
    w.WriteJSON(user)
}
```

### 3. Distributed Tracing

```go
// gateway/internal/handlers/auth.go
import "go.opentelemetry.io/api/trace"

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    ctx, span := h.tracer.Start(r.Context(), "auth.login")
    defer span.End()
    
    span.SetAttributes(
        attribute.String("user.email", req.Email),
    )
    
    user, err := h.authClient.Login(ctx, req)
    
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "authentication failed")
        return
    }
    
    span.SetAttributes(
        attribute.String("user.id", user.Id),
    )
    span.SetStatus(codes.Ok, "")
    
    w.WriteJSON(user)
}
```

---

## ⚡ Best Practices

✅ **Loguea a nivel correcto** (no TODO a DEBUG)
✅ **Usa logs estructurados** (JSON)
✅ **Instrumenta puntos críticos** (auth, payment, DB)
✅ **Correlaciona con request ID** (trace_id)
✅ **Métricas SLI** (latencia p99, error rate, availability)
✅ **Alertas en anomalías** (error rate > 1%, latency > 500ms)
✅ **Retención de logs** (30 días prod, 7 días dev)

---

## ⚠️ Antipatrones

❌ Loguear contraseñas/tokens
❌ Loguear TODO en producción
❌ Logs sin estructura (concatenación de strings)
❌ Métricas sin contexto (label cardinality explosion)
❌ Tracing en desarrollo pero no en producción
❌ No alertar en métricas críticas

---

## 📚 Recursos

### Herramientas
- Prometheus: https://prometheus.io
- Loki: https://grafana.com/loki/
- Jaeger: https://www.jaegertracing.io
- Grafana: https://grafana.com
- OpenTelemetry: https://opentelemetry.io

### Go
- logrus: https://github.com/sirupsen/logrus
- Prometheus Go client: https://pkg.go.dev/github.com/prometheus/client_golang
- OpenTelemetry Go: https://github.com/open-telemetry/opentelemetry-go

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo monitorearías si tu servicio de pagos está fallando?"

**Respuesta:**
> "Usaría los 3 pilares de observabilidad: Primero, logs estructurados con información de cada intento de pago (usuario, monto, resultado). Segundo, métricas Prometheus: contador de intentos exitosos/fallidos, histograma de latencia. Tercero, tracing distribuido para ver dónde exactamente falla (¿timeout en Stripe? ¿validación? ¿DB?). Además, alertas en Prometheus si error_rate > 5% o latencia_p99 > 5s. En dashboard Grafana visualizo en tiempo real: error rate trending, latency, requests in flight. Si falla, puedo correlacionar con: qué cambió (deploy), hay spike de tráfico, o servicio dependiente está down. Esto permite go from 'payments are broken' a 'Stripe API timeout increased, circuit breaker tripped' en minutos."

---

#observability #monitoring #logging #metrics #tracing #prometheus #opentelemetry
