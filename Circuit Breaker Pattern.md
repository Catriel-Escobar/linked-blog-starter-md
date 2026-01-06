---
tags: [resilience, microservices, fault-tolerance, patterns]
date: 2026-01-05
related: [Service Mesh, Retry Pattern, Timeout, Bulkhead Pattern]
status: reference
---

# Circuit Breaker Pattern

## 📋 ¿Qué es un Circuit Breaker?

Un **patrón de diseño** que previene que una aplicación intente ejecutar una operación que probablemente fallará, permitiendo continuar sin esperar a que el error se corrija y evitando cascadas de fallos.

**Analogía:** Como el interruptor eléctrico de tu casa:
- ⚡ Sobrecarga → interruptor se abre (corta electricidad)
- 🛠️ Arreglas el problema
- ✅ Cierras el interruptor (restaura electricidad)

---

## 🎯 Problema que Resuelve

### Sin Circuit Breaker

```
Gateway → Auth Service (caído)
   ↓
   Espera 30s (timeout)
   Retry 1 → Espera 30s
   Retry 2 → Espera 30s
   Retry 3 → Espera 30s
   
   Total: 120 segundos de espera
   Resultado: Usuario frustrado + recursos desperdiciados
```

**Cascada de fallos:**
```
Auth Service (caído)
    ↑
Gateway (sobrecargado esperando)
    ↑
Usuario (esperando)
    
Múltiples usuarios → Gateway se cae también
```

### Con Circuit Breaker

```
Gateway → Auth Service (caído)
   ↓
   Intento 1: Falla (5s)
   Intento 2: Falla (5s)
   Intento 3: Falla (5s)
   
   🔴 Circuit OPEN
   
   Siguientes requests: Falla inmediatamente
   Total: <1ms (sin esperar)
   
   ⏱️ Después de 30s: Intenta de nuevo
   ✅ Si funciona: Circuit CLOSED
```

---

## 🔄 Estados del Circuit Breaker

```
         ┌─────────────┐
   ┌────→│   CLOSED    │←────┐
   │     │  (Normal)   │     │
   │     └──────┬──────┘     │
   │            │             │
   │      Threshold           │
   │      Exceeded           Success
   │            │              │
   │     ┌──────↓──────┐      │
   │     │    OPEN     │      │
   │     │  (Circuit   │      │
   │     │   Abierto)  │      │
   │     └──────┬──────┘      │
   │            │              │
   │       Timeout             │
   │            │              │
   │     ┌──────↓──────┐      │
   └─────│  HALF-OPEN  │──────┘
  Falla  │  (Probando) │  Success
         └─────────────┘
```

### 1. **CLOSED (Cerrado - Normal)**
- ✅ Requests pasan normalmente
- 📊 Cuenta errores
- ⚠️ Si errores > threshold → OPEN

### 2. **OPEN (Abierto - Protección)**
- ❌ Requests fallan inmediatamente (fail-fast)
- ⏱️ No intenta llamar al servicio
- 🕐 Después de timeout → HALF-OPEN

### 3. **HALF-OPEN (Semi-abierto - Prueba)**
- 🧪 Permite 1 request de prueba
- ✅ Si success → CLOSED
- ❌ Si falla → OPEN

---

## 💻 Implementación en Go

### Versión Simple

```go
package circuitbreaker

import (
    "errors"
    "sync"
    "time"
)

type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    maxFailures  int           // Threshold de errores
    timeout      time.Duration // Cuánto esperar antes de probar
    state        State
    failures     int
    lastFailTime time.Time
    mu           sync.Mutex
}

func New(maxFailures int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        maxFailures: maxFailures,
        timeout:     timeout,
        state:       StateClosed,
    }
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    // Estado OPEN: fail-fast
    if cb.state == StateOpen {
        if time.Since(cb.lastFailTime) > cb.timeout {
            cb.state = StateHalfOpen
            cb.failures = 0
        } else {
            return errors.New("circuit breaker is open")
        }
    }

    // Intentar ejecutar función
    err := fn()

    // Manejar resultado
    if err != nil {
        cb.failures++
        cb.lastFailTime = time.Now()

        if cb.failures >= cb.maxFailures {
            cb.state = StateOpen
        }
        return err
    }

    // Success: reset
    if cb.state == StateHalfOpen {
        cb.state = StateClosed
    }
    cb.failures = 0
    return nil
}
```

### Uso

```go
// Crear circuit breaker
cb := circuitbreaker.New(
    3,              // Max 3 errores
    30*time.Second, // 30s antes de reintentar
)

// Usar en cliente gRPC
func (c *AuthClient) Login(ctx context.Context, email, password string) (*pb.LoginResponse, error) {
    var resp *pb.LoginResponse
    
    err := cb.Call(func() error {
        var err error
        resp, err = c.client.Login(ctx, &pb.LoginRequest{
            Email:    email,
            Password: password,
        })
        return err
    })
    
    return resp, err
}
```

---

## 📊 Ejemplo Práctico

### Escenario: Auth Service Caído

```go
// Request 1 (t=0s)
resp, err := client.Login(ctx, email, pass)
// Falla después de 5s timeout
// State: CLOSED → failures = 1

// Request 2 (t=5s)
resp, err := client.Login(ctx, email, pass)
// Falla después de 5s timeout
// State: CLOSED → failures = 2

// Request 3 (t=10s)
resp, err := client.Login(ctx, email, pass)
// Falla después de 5s timeout
// State: CLOSED → OPEN (failures = 3 >= maxFailures)

// Request 4 (t=15s)
resp, err := client.Login(ctx, email, pass)
// Falla INMEDIATAMENTE (<1ms)
// Error: "circuit breaker is open"
// State: OPEN

// ... más requests fallan instantáneamente

// Request N (t=45s, después de 30s timeout)
resp, err := client.Login(ctx, email, pass)
// Intenta de nuevo (State: HALF-OPEN)
// Si Auth Service volvió:
//   Success → State: CLOSED
// Si sigue caído:
//   Falla → State: OPEN (otros 30s)
```

---

## 🎨 Variantes y Extensiones

### 1. **Sliding Window Circuit Breaker**

En lugar de contar errores consecutivos, cuenta errores en una ventana de tiempo.

```go
type SlidingWindowCB struct {
    window      time.Duration
    threshold   float64 // Porcentaje de errores (0.5 = 50%)
    requests    []requestRecord
}

type requestRecord struct {
    timestamp time.Time
    success   bool
}

func (cb *SlidingWindowCB) Call(fn func() error) error {
    // Limpiar requests antiguos
    now := time.Now()
    cutoff := now.Add(-cb.window)
    
    cb.requests = filterRecords(cb.requests, func(r requestRecord) bool {
        return r.timestamp.After(cutoff)
    })
    
    // Calcular tasa de error
    errorRate := calculateErrorRate(cb.requests)
    
    if errorRate > cb.threshold {
        return errors.New("circuit breaker open")
    }
    
    // Ejecutar y registrar
    err := fn()
    cb.requests = append(cb.requests, requestRecord{
        timestamp: now,
        success:   err == nil,
    })
    
    return err
}
```

### 2. **Con Backoff Exponencial**

```go
type ExponentialBackoffCB struct {
    baseTimeout time.Duration
    maxTimeout  time.Duration
    attempts    int
}

func (cb *ExponentialBackoffCB) getTimeout() time.Duration {
    timeout := cb.baseTimeout * time.Duration(1<<cb.attempts)
    if timeout > cb.maxTimeout {
        return cb.maxTimeout
    }
    return timeout
}

// Primera vez: 5s
// Segunda vez: 10s
// Tercera vez: 20s
// Cuarta vez: 40s (max)
```

### 3. **Con Fallback**

```go
func (cb *CircuitBreaker) CallWithFallback(fn func() error, fallback func() error) error {
    err := cb.Call(fn)
    
    if err != nil && cb.state == StateOpen {
        // Circuit abierto, usar fallback
        return fallback()
    }
    
    return err
}

// Uso
err := cb.CallWithFallback(
    func() error {
        // Llamar al servicio real
        return client.Login(ctx, email, pass)
    },
    func() error {
        // Fallback: Cache, servicio alternativo, etc.
        return loginFromCache(email)
    },
)
```

---

## 🔧 Librerías Populares

### 1. **sony/gobreaker** (Recomendada)

```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "Auth Service",
    MaxRequests: 3,  // Half-open: intentar 3 requests
    Interval:    60 * time.Second,
    Timeout:     30 * time.Second,
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 3 && failureRatio >= 0.6
    },
    OnStateChange: func(name string, from, to gobreaker.State) {
        log.Printf("Circuit breaker '%s' changed from %s to %s", name, from, to)
    },
})

// Uso
resp, err := cb.Execute(func() (interface{}, error) {
    return client.Login(ctx, email, password)
})
```

### 2. **hystrix-go** (Netflix)

```go
import "github.com/afex/hystrix-go/hystrix"

hystrix.ConfigureCommand("auth_login", hystrix.CommandConfig{
    Timeout:                1000, // 1s
    MaxConcurrentRequests:  100,
    ErrorPercentThreshold:  50,   // 50% error rate
    RequestVolumeThreshold: 3,    // Mínimo 3 requests
    SleepWindow:            5000, // 5s antes de retry
})

err := hystrix.Do("auth_login", func() error {
    _, err := client.Login(ctx, email, password)
    return err
}, func(err error) error {
    // Fallback
    return loginFromCache(email)
})
```

---

## 📈 Métricas Importantes

### Qué monitorear

```go
type Metrics struct {
    TotalRequests     int64
    SuccessfulCalls   int64
    FailedCalls       int64
    CircuitOpenCount  int64
    CircuitCloseCount int64
    AverageLatency    time.Duration
}

// Prometheus metrics
var (
    circuitBreakerState = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "circuit_breaker_state",
            Help: "Circuit breaker state (0=closed, 1=half-open, 2=open)",
        },
        []string{"service"},
    )
    
    circuitBreakerFailures = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "circuit_breaker_failures_total",
            Help: "Total number of failures",
        },
        []string{"service"},
    )
)
```

### Dashboard

```
Circuit Breaker: Auth Service
┌─────────────────────────────┐
│ State: 🟢 CLOSED            │
│ Failures: 2/5               │
│ Success Rate: 95%           │
│ Last Trip: 2h ago           │
└─────────────────────────────┘

Recent History:
━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓✓✓✓✓✓✗✗✓✓✓✓✓✓✓✓✗✓✓✓
```

---

## 🎯 Integración con tu Proyecto

### En el Gateway

```go
// gateway/internal/client/auth_client.go

type AuthClient struct {
    conn           *grpc.ClientConn
    client         pb.AuthServiceClient
    circuitBreaker *gobreaker.CircuitBreaker
}

func NewAuthClient(url string) (*AuthClient, error) {
    conn, err := grpc.NewClient(url, 
        grpc.WithTransportCredentials(insecure.NewCredentials()))
    
    cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
        Name:        "Auth Service",
        MaxRequests: 3,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures >= 3
        },
    })
    
    return &AuthClient{
        conn:           conn,
        client:         pb.NewAuthServiceClient(conn),
        circuitBreaker: cb,
    }, nil
}

func (c *AuthClient) Login(ctx context.Context, email, password string) (*pb.LoginResponse, error) {
    resp, err := c.circuitBreaker.Execute(func() (interface{}, error) {
        return c.client.Login(ctx, &pb.LoginRequest{
            Email:    email,
            Password: password,
        })
    })
    
    if err != nil {
        return nil, err
    }
    
    return resp.(*pb.LoginResponse), nil
}
```

---

## ✅ Mejores Prácticas

### Do's ✅
- ✅ Timeout razonable (no muy largo)
- ✅ Threshold basado en datos reales
- ✅ Logging de cambios de estado
- ✅ Métricas/alertas en circuit open
- ✅ Fallbacks cuando sea posible
- ✅ Considerar sliding window para tráfico variable

### Don'ts ❌
- ❌ Threshold muy bajo (falsos positivos)
- ❌ Timeout muy corto (trips innecesarios)
- ❌ Ignorar logs de circuit breaker
- ❌ Mismo circuit breaker para todos los endpoints
- ❌ No monitorear estado del circuit

---

## 🔗 Patrones Relacionados

### Combinación con otros patrones

```
Request
    ↓
Retry Pattern (3 intentos)
    ↓
Circuit Breaker (proteger si muchos fallos)
    ↓
Timeout (no esperar forever)
    ↓
Bulkhead (límite de recursos)
    ↓
Fallback (si todo falla)
```

**Ejemplo completo:**
```go
// 1. Circuit Breaker (nivel más alto)
err := circuitBreaker.Execute(func() error {
    
    // 2. Timeout
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    // 3. Retry con backoff
    return retry.Do(func() error {
        // 4. Request real
        return client.Login(ctx, email, password)
    },
        retry.Attempts(3),
        retry.Delay(100*time.Millisecond),
    )
})

// 5. Fallback si circuit breaker falla
if err != nil {
    return loginFromCache(email)
}
```

---

## 📚 Recursos

### Librerías
- sony/gobreaker: https://github.com/sony/gobreaker
- hystrix-go: https://github.com/afex/hystrix-go
- go-resiliency: https://github.com/eapache/go-resiliency

### Artículos
- "Circuit Breaker" - Martin Fowler
- "Release It!" - Michael Nygard (libro)

### Videos
- "Circuit Breaker Pattern" - Gaurav Sen (YouTube)
- "Microservices Patterns: Circuit Breaker" - IBM Technology

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo manejarías una situación donde un servicio dependiente está fallando?"

**Respuesta:**
> "Implementaría un Circuit Breaker pattern. Básicamente, después de cierto número de fallos consecutivos (por ejemplo, 3), el circuit breaker entra en estado OPEN y las siguientes requests fallan inmediatamente sin intentar llamar al servicio, evitando cascadas de fallos y liberando recursos. Después de un timeout (30s), entra en HALF-OPEN para probar si el servicio se recuperó. Si funciona, vuelve a CLOSED. Además, combinaría esto con un fallback (cache, respuesta default) para mantener funcionalidad degradada. En mi proyecto actual uso la librería sony/gobreaker que maneja los estados automáticamente."

---

#circuit-breaker #resilience #fault-tolerance #patterns #microservices
