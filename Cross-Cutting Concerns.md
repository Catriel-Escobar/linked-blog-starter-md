---
tags: [architecture, concerns, logging, security, monitoring, cross-cutting]
date: 2026-01-05
related: [AOP, Middleware, Decorator Pattern, Chain of Responsibility]
status: reference
---

# Cross-Cutting Concerns

## 📋 ¿Qué son Cross-Cutting Concerns?

Funcionalidades que **atraviesan múltiples componentes** de la aplicación y no pertenecen a ninguno específicamente.

**Ejemplos:**
- 🔐 **Seguridad:** Autenticación, autorización
- 📝 **Logging:** Registrar eventos
- 📊 **Monitoreo:** Métricas, performance
- ⏱️ **Transacciones:** Control de transacciones DB
- 🔄 **Caching:** Guardar resultados
- ⚠️ **Manejo de errores:** Gestión global de excepciones

**Analogía:** Un edificio:
- Funcionalidad principal: Oficinas (apartamentos)
- Cross-cutting concerns: Electricidad, plomería, aire acondicionado (atraviesan todo el edificio)

---

## 🎯 Problema que Resuelven

### Sin Separación de Concerns

```go
// ❌ La lógica de negocio está mezclada con concerns transversales

func (s *UserService) Register(ctx context.Context, email, password string) (*User, error) {
    // Logging
    log.Printf("User registration started: %s", email)
    
    // Seguridad: validar entrada
    if !isValidEmail(email) {
        log.Printf("Invalid email: %s", email)
        return nil, ErrInvalidEmail
    }
    
    // Manejo de error específico
    user, err := s.repo.GetByEmail(ctx, email)
    if err != nil {
        log.Printf("Database error: %v", err)
        return nil, err
    }
    
    if user != nil {
        log.Printf("User already exists: %s", email)
        return nil, ErrUserExists
    }
    
    // Transacción manual
    tx, _ := s.db.Begin()
    defer func() {
        if r := recover(); r != nil {
            tx.Rollback()
            log.Printf("Panic during registration: %v", r)
        }
    }()
    
    // Caching: invalidar cache
    s.cache.Delete("users:all")
    
    // FINALMENTE: Lógica de negocio real (3 líneas)
    hashedPassword, _ := hashPassword(password)
    newUser := &User{Email: email, Password: hashedPassword}
    result, err := s.repo.Create(ctx, newUser)
    
    // Logging del resultado
    if err != nil {
        log.Printf("User creation failed: %v", err)
        tx.Rollback()
        return nil, err
    }
    
    tx.Commit()
    log.Printf("User registered successfully: %s", email)
    
    // Monitoreo
    prometheus.IncrementUserRegistrations()
    
    return result, nil
}

// ❌ Problemas:
// - Función gigante (200+ líneas)
// - Lógica mezclada
// - Difícil de testear (todo junto)
// - Repetir mismo patrón en otros métodos
// - Cambios a logging/seguridad afectan toda la función
```

### Con Separación de Concerns

```go
// ✅ Concerns transversales separados mediante middleware/decoradores

// Lógica de negocio PURA
func (s *UserService) Register(ctx context.Context, email, password string) (*User, error) {
    // Validación básica
    if email == "" || password == "" {
        return nil, ErrInvalidInput
    }
    
    // Obtener usuario existente
    user, _ := s.repo.GetByEmail(ctx, email)
    if user != nil {
        return nil, ErrUserExists
    }
    
    // Crear nuevo usuario
    hashedPassword, _ := hashPassword(password)
    return s.repo.Create(ctx, &User{
        Email:    email,
        Password: hashedPassword,
    })
}

// Los concerns transversales se aplican mediante middleware
@Logging
@Authentication
@RateLimit
@Transaction
@Caching
func (s *UserService) Register(ctx context.Context, email, password string) (*User, error) {
    // ... mismo código limpio
}

// ✅ Ventajas:
// - Función pequeña (10 líneas, solo lógica)
// - Fácil de testear
// - Concerns reutilizables
// - Cambios no afectan la lógica
// - Separación clara de responsabilidades
```

---

## 🏗️ Tipos de Cross-Cutting Concerns

### 1. **Seguridad**

```go
// Autenticación
func WithAuth(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next(w, r)
    }
}

// Autorización
func WithAuthorization(requiredRole string) func(Handler) Handler {
    return func(next Handler) Handler {
        return func(w http.ResponseWriter, r *http.Request) {
            user := r.Context().Value("user").(*User)
            if !hasRole(user, requiredRole) {
                http.Error(w, "Forbidden", http.StatusForbidden)
                return
            }
            next(w, r)
        }
    }
}

// Rate Limiting
func WithRateLimit(requestsPerSec int) func(Handler) Handler {
    limiter := rate.NewLimiter(rate.Limit(requestsPerSec), 1)
    return func(next Handler) Handler {
        return func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
                return
            }
            next(w, r)
        }
    }
}
```

### 2. **Logging y Observabilidad**

```go
// Logging
func WithLogging(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        log.Printf("[%s] %s %s", time.Now(), r.Method, r.URL.Path)
        next(w, r)
        duration := time.Since(start)
        log.Printf("[%s] Completed in %v", r.URL.Path, duration)
    }
}

// Métricas
func WithMetrics(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next(w, r)
        duration := time.Since(start).Seconds()
        
        prometheus.HTTPRequestDuration.WithLabelValues(
            r.Method, r.URL.Path,
        ).Observe(duration)
    }
}

// Tracing distribuido
func WithTracing(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, span := tracer.Start(r.Context(), "http.request")
        defer span.End()
        
        r = r.WithContext(ctx)
        next(w, r)
    }
}
```

### 3. **Validación**

```go
// Validación global de entrada
func WithValidation(validator *Validator) func(Handler) Handler {
    return func(next Handler) Handler {
        return func(w http.ResponseWriter, r *http.Request) {
            var payload interface{}
            json.NewDecoder(r.Body).Decode(&payload)
            
            if err := validator.Validate(payload); err != nil {
                http.Error(w, err.Error(), http.StatusBadRequest)
                return
            }
            next(w, r)
        }
    }
}
```

### 4. **Transacciones**

```go
// Gestión automática de transacciones
func WithTransaction(db *sql.DB) func(Handler) Handler {
    return func(next Handler) Handler {
        return func(w http.ResponseWriter, r *http.Request) {
            tx, _ := db.BeginTx(r.Context(), nil)
            
            ctx := context.WithValue(r.Context(), "tx", tx)
            r = r.WithContext(ctx)
            
            next(w, r)
            
            tx.Commit()  // Auto-commit al salir
        }
    }
}
```

### 5. **Caching**

```go
// Caching de respuestas
func WithCaching(cache Cache, duration time.Duration) func(Handler) Handler {
    return func(next Handler) Handler {
        return func(w http.ResponseWriter, r *http.Request) {
            cacheKey := r.Method + ":" + r.URL.String()
            
            if cached, ok := cache.Get(cacheKey); ok {
                w.Write(cached)
                return
            }
            
            next(w, w)  // Respuesta va a caché
            
            cache.Set(cacheKey, w.Body, duration)
        }
    }
}
```

---

## 💻 Implementación: Middleware Stack en Go

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "time"
)

type Handler func(http.ResponseWriter, *http.Request)
type Middleware func(Handler) Handler

// Middleware 1: Logging
func Logging(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        fmt.Printf("[%s] %s %s\n", time.Now().Format("15:04:05"), r.Method, r.URL.Path)
        next(w, r)
    }
}

// Middleware 2: Authentication
func Authentication(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("X-Token")
        if token == "" {
            w.WriteHeader(http.StatusUnauthorized)
            fmt.Fprintf(w, "No token provided")
            return
        }
        
        // Validar token
        user := "user-123"  // Token válido
        ctx := context.WithValue(r.Context(), "user", user)
        r = r.WithContext(ctx)
        
        next(w, r)
    }
}

// Middleware 3: Timeout
func Timeout(duration time.Duration) Middleware {
    return func(next Handler) Handler {
        return func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), duration)
            defer cancel()
            
            r = r.WithContext(ctx)
            next(w, r)
        }
    }
}

// Middleware 4: Metrics
func Metrics(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next(w, r)
        duration := time.Since(start)
        fmt.Printf("Duration: %v\n", duration)
    }
}

// Aplicar middleware: Chain of responsibility
func chain(h Handler, middlewares ...Middleware) Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

// Handler de negocio
func business(w http.ResponseWriter, r *http.Request) {
    user := r.Context().Value("user")
    fmt.Fprintf(w, "Hello, %v! This is business logic\n", user)
}

func main() {
    // Aplicar middlewares en orden
    handler := chain(
        business,
        Metrics,
        Timeout(5*time.Second),
        Authentication,
        Logging,
    )
    
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

**Ejecución:**
```
GET /api/data (con token)
    ↓
Logging: [15:30:45] GET /api/data
    ↓
Authentication: Token validado, user = user-123
    ↓
Timeout: Context con timeout 5s
    ↓
Metrics: Iniciar timer
    ↓
Business: Hello, user-123! This is business logic
    ↓
Metrics: Duration: 2ms
```

---

## 🎯 Tu Proyecto: Aplicar Concerns

### Estructura Actual (con concerns mezclados)

```go
// ❌ gateway/internal/handlers/auth.go
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    // Logging
    log.Printf("Login attempt from %s", r.RemoteAddr)
    
    // Validación
    var req LoginRequest
    json.NewDecoder(r.Body).Decode(&req)
    if req.Email == "" {
        w.WriteHeader(http.StatusBadRequest)
        return
    }
    
    // Timeout manual (ya haces esto con middleware)
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    // Llamar servicio
    user, err := h.authClient.Login(ctx, &pb.LoginRequest{...})
    
    // Serializar
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

### Refactorizar con Separation of Concerns

```go
// ✅ gateway/internal/handlers/auth.go
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // Lógica de negocio PURA
    user, err := h.authClient.Login(r.Context(), &pb.LoginRequest{
        Email:    req.Email,
        Password: req.Password,
    })
    
    if err != nil {
        http.Error(w, err.Error(), http.StatusUnauthorized)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

// ✅ gateway/internal/http/router.go
func setupRoutes(r *chi.Mux) {
    // Aplicar concerns transversales globalmente
    r.Use(middleware.Logger)        // Logging
    r.Use(middleware.RequestID)     // Tracing
    r.Use(middleware.Timeout(5*time.Second))  // Timeout
    
    // Rutas
    r.Post("/auth/login", h.Login)
    r.Post("/auth/register", withRateLimit(h.Register))
}
```

---

## 📊 Matriz: Concerns en Arquitectura de Microservicios

```
Service A                Service B                Service C
├─ Auth       ────┐      ├─ Auth       ────┐      ├─ Auth
├─ Logging    ────┼──────┼─ Logging    ────┼──────┼─ Logging
├─ Metrics    ────┼──────┼─ Metrics    ────┼──────┼─ Metrics
├─ Timeout    ────┤      ├─ Timeout    ────┤      ├─ Timeout
└─ RateLimit  ────┴──────┴─ RateLimit  ────┴──────┴─ RateLimit

Cross-cutting: Los mismos concerns en todos los servicios
```

**Solución:** Service Mesh (Istio, Linkerd) maneja estos concerns automáticamente.

---

## ⚡ Técnicas para Implementar

| Técnica | Cuando Usar | Ejemplo |
|---------|-------------|---------|
| **Middleware** | HTTP handlers | chi.Use(logger) |
| **Decorators** | Funciones/métodos | @Transactional |
| **AOP (Aspects)** | Lenguajes con soporte | Spring AOP (Java) |
| **Service Mesh** | Microservicios | Istio/Linkerd |
| **Interceptors** | gRPC | grpc.UnaryInterceptor |
| **Proxies** | Infraestructura | Nginx, Envoy |

---

## 📚 Recursos

### Conceptos
- "Separation of Concerns" - Wikipedia
- "Aspect-Oriented Programming" - AspectJ

### Go
- chi middleware: https://pkg.go.dev/github.com/go-chi/chi
- gRPC interceptors: https://grpc.io/docs/guides/performance-best-practices

### Microservicios
- Service Mesh Patterns: https://servicemeshpatterns.com
- Istio: https://istio.io

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo mantendrías logging, autenticación y timeout sin que contaminen la lógica de negocio?"

**Respuesta:**
> "Usaría Cross-Cutting Concerns separados mediante middleware. La lógica de negocio debe ser pura: solo hace lo que tiene que hacer. Todos los concerns transversales (logging, auth, timeout, métricas, caching) se aplican como middleware reutilizable. En Go, uso chi middleware: `r.Use(middleware.Logger, middleware.Timeout(...), middleware.Auth)`. Cada middleware es una función que envuelve el siguiente handler, permitiendo que cada concern sea independiente. En gRPC, usaría unary interceptors para lo mismo. En una arquitectura de microservicios grande, usaría un Service Mesh como Istio que maneja estos concerns a nivel de infraestructura, sin tocar el código. La ventaja es que cambios a logging o autenticación no afectan la lógica, y los mismos concerns se reutilizan en todos los servicios."

---

#cross-cutting-concerns #middleware #separation-of-concerns #architecture #aop
