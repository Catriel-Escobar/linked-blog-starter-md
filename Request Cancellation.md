---
tags: [context, cancellation, timeouts, goroutines, request-lifecycle]
date: 2026-01-05
related: [Graceful Shutdown, Context, Timeouts, Goroutine Management]
status: reference
---

# Request Cancellation

## 📋 ¿Qué es Request Cancellation?

El mecanismo para **detener una solicitud en progreso** cuando:
- Usuario cierra navegador (en HTTP)
- Timeout expira
- Servicio dependiente falla
- Load balancer descarga el pod

**Analogía:** Lavar platos:
- Sin cancellation: Aunque alguien se vaya, sigo lavando (desperdicio)
- Con cancellation: Me avisan que se van, paro de lavar (eficiente)

---

## 🎯 Problema que Resuelve

### Sin Request Cancellation

```go
// ❌ Sin context cancellation
func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    order := parseOrder(r)
    
    // Operación larga: 60 segundos
    result := expensiveComputation()  // Sigue aunque usuario cerró navegador
    
    // Si usuario cerró navegador después de 5s:
    // - Handler sigue corriendo 55s más
    // - Desperdicida CPU
    // - Otra solicitud podría usar ese recurso
    
    w.WriteJSON(result)
}

// Problemas:
❌ Desperdicia recursos
❌ Lentitud en sistema
❌ Timeouts no funcionan
❌ Impossibe detener operaciones largas
```

### Con Request Cancellation

```go
// ✅ Con context cancellation
func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // context con cancel
    
    // Seleccionar: operación termina O context cancela
    select {
    case result := <-expensiveComputationChan(ctx):
        w.WriteJSON(result)
        
    case <-ctx.Done():  // Usuario cerró navegador
        log.Println("Request cancelled")
        return  // Stop inmediatamente
    }
}

// Ventajas:
✅ Libera recursos inmediatamente
✅ Sistem responsivo
✅ Timeouts funcionan
✅ Detiene operaciones largas
```

---

## 🏗️ Context en Go

### Context Tree

```
Background context (raíz)
├─ Request 1 (HTTP GET /users)
│  ├─ Database query (con timeout 5s)
│  └─ Cache lookup (sin timeout)
│
├─ Request 2 (HTTP POST /orders)
│  ├─ Validation (con timeout 1s)
│  ├─ Database transaction (con timeout 10s)
│  └─ Payment API call (con timeout 30s)
│
└─ Background job (con timeout 1h)
   ├─ Email sending (con timeout 5m)
   └─ Analytics update (con timeout 1m)
```

### Tipos de Context

```go
import "context"

// 1. Background (raíz, nunca se cancela)
ctx := context.Background()

// 2. TODO (para código incompleto)
ctx := context.TODO()

// 3. Con timeout (se cancela automáticamente)
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// 4. Con deadline (se cancela en tiempo específico)
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(30*time.Second))
defer cancel()

// 5. Con cancel (se cancela manualmente)
ctx, cancel := context.WithCancel(context.Background())
// ... en algún lugar: cancel()

// 6. Con valores (pasar datos)
ctx := context.WithValue(context.Background(), "user_id", "123")
userID := ctx.Value("user_id")
```

---

## 💻 Ejemplos Prácticos

### 1. **Timeout en Request HTTP**

```go
// Cliente HTTP
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// Request se cancela automáticamente después de 5s
req, _ := http.NewRequestWithContext(ctx, "GET", "https://api.example.com/slow", nil)
resp, err := http.DefaultClient.Do(req)

if err == context.DeadlineExceeded {
    log.Println("Request timeout")
}
```

### 2. **Timeout en Database Query**

```go
func (r *UserRepository) GetByID(ctx context.Context, id string) (*User, error) {
    // Query se cancela si context se cancela
    row := r.db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = ?", id)
    
    var user User
    err := row.Scan(&user.ID, &user.Email, ...)
    
    if err == context.DeadlineExceeded {
        log.Println("Database query timeout")
        return nil, err
    }
    
    return &user, nil
}

// Usar con timeout
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

user, err := repo.GetByID(ctx, "123")
```

### 3. **Detener Goroutines**

```go
func (s *Service) ProcessOrders(ctx context.Context) error {
    // Crear worker goroutines
    for i := 0; i < 10; i++ {
        go s.worker(ctx)
    }
    
    return nil
}

func (s *Service) worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            log.Println("Worker cancelled")
            return  // Exit goroutine
            
        case order := <-s.orderQueue:
            s.processOrder(order)
        }
    }
}

// Cuando shutdown:
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

s.ProcessOrders(ctx)
// Todos los workers se detienen cuando ctx.Done() se cierra
```

### 4. **Cascada de Cancellations**

```go
// ✅ Correctamente: Cada nivel agrega timeout
func (h *OrderHandler) CreateOrder(ctx context.Context, order *Order) error {
    // Handler timeout: 30s
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    // Validar (timeout: 1s)
    ctx1, cancel1 := context.WithTimeout(ctx, 1*time.Second)
    if err := h.validateOrder(ctx1, order); err != nil {
        cancel1()
        return err
    }
    cancel1()
    
    // Crear en DB (timeout: 10s)
    ctx2, cancel2 := context.WithTimeout(ctx, 10*time.Second)
    createdOrder, err := h.orderService.Create(ctx2, order)
    cancel2()
    
    if err != nil {
        return err
    }
    
    // Procesar pago (timeout: 20s)
    ctx3, cancel3 := context.WithTimeout(ctx, 20*time.Second)
    if err := h.paymentService.Charge(ctx3, createdOrder); err != nil {
        cancel3()
        return err
    }
    cancel3()
    
    return nil
}

// Cascada:
// Si handler timeout expira → cancela ctx
// Si ctx se cancela → todos los ctx1, ctx2, ctx3 se cancelan
// Resultado: TODO se detiene, sin limpieza incompleta
```

---

## 📊 Propagación de Context

### En HTTP Request

```go
// Gateway recibe HTTP request
func (h *GatewayHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    // r.Context() contiene:
    // - cancel automático si cliente cierra conexión
    // - timeout del servidor
    ctx := r.Context()
    
    // Pasar context a llamadas gRPC
    user, err := h.authClient.Login(ctx, req)  // Context se propaga
}

// Auth Service recibe gRPC call
func (s *AuthServer) Login(ctx context.Context, req *pb.LoginRequest) (*pb.User, error) {
    // ctx es el mismo context del HTTP request original
    // Si cliente cierra conexión en gateway, ctx se cancela aquí también
    
    // Pasar a database
    user, err := s.repo.GetByEmail(ctx, req.Email)
}

// Database query
row := db.QueryRowContext(ctx, "SELECT ...")
// Si ctx se cancela, query se detiene
```

### Flujo Completo

```
Cliente HTTP
    │
    ├─ Abre conexión (GET /api/users)
    │
    └─ ctx = request context
        ├─ Timeout: 30s (servidor)
        │
        └─ Gateway Handler
            ├─ ctx (del request)
            ├─ Timeout: 30s
            │
            └─ Llama Auth Service (gRPC)
                ├─ ctx (mismo)
                ├─ Timeout: 30s
                │
                └─ Database Query
                    ├─ ctx (mismo)
                    ├─ Timeout: 30s
                    │
                    └─ Si query tarda >30s → context.DeadlineExceeded

Si cliente cierra conexión:
    → ctx.Done() se cierra
    → Query se cancela
    → Database connection se libera
    → Auth Service se detiene
    → Gateway se detiene
    → Recurso liberado
```

---

## 🎯 Tu Proyecto: Aplicar Cancellation

### Gateway Handler

```go
// gateway/internal/handlers/auth.go
func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    // Context del request (se cancela si cliente cierra conexión)
    ctx := r.Context()
    
    // Timeout total: 30 segundos
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    var req RegisterRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        return
    }
    
    // Validación: timeout 1 segundo
    ctx1, cancel1 := context.WithTimeout(ctx, 1*time.Second)
    if err := validateEmail(ctx1, req.Email); err != nil {
        cancel1()
        w.WriteHeader(http.StatusBadRequest)
        return
    }
    cancel1()
    
    // Llamar auth service: context se propaga automáticamente
    // Si timeout expira, llamada se cancela
    user, err := h.authClient.Register(ctx, &pb.RegisterRequest{
        Email:    req.Email,
        Password: req.Password,
    })
    
    if err == context.DeadlineExceeded {
        w.WriteHeader(http.StatusGatewayTimeout)
        w.Write([]byte("Request timeout"))
        return
    }
    
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        return
    }
    
    w.WriteJSON(user)
}
```

### Auth Service

```go
// auth-service/internal/handler/grpc_auth_handler.go
func (h *AuthHandler) Register(ctx context.Context, req *pb.RegisterRequest) (*pb.User, error) {
    // ctx heredado del gateway (con timeout)
    
    // Si context se cancela, estos también se cancelan:
    user, err := h.service.Register(ctx, req.Email, req.Password)
    
    if err == context.DeadlineExceeded {
        return nil, status.Error(codes.DeadlineExceeded, "registration timeout")
    }
    
    if err != nil {
        return nil, status.Error(codes.Internal, err.Error())
    }
    
    return user, nil
}

// auth-service/internal/service/auth_service.go
func (s *AuthService) Register(ctx context.Context, email, password string) (*User, error) {
    // ctx con timeout se propaga a repo
    user, err := s.repo.Create(ctx, email, password)
    
    if err == context.DeadlineExceeded {
        return nil, fmt.Errorf("database timeout: %w", err)
    }
    
    return user, nil
}

// auth-service/internal/repository/user_repository.go
func (r *UserRepository) Create(ctx context.Context, email, password string) (*User, error) {
    // ctx se pasa a QueryRowContext
    // Si timeout expira, query se cancela
    row := r.db.QueryRowContext(ctx, "INSERT INTO users...")
    
    // Si context se cancela mientras esperamos resultado:
    var user User
    if err := row.Scan(...); err != nil {
        if err == context.DeadlineExceeded {
            return nil, fmt.Errorf("scan timeout")
        }
        return nil, err
    }
    
    return &user, nil
}
```

---

## ⚡ Best Practices

✅ **Siempre usa context** en funciones async
✅ **Propaga context** a través de capas
✅ **Usa `defer cancel()`** después de WithCancel/Timeout/Deadline
✅ **Respeta context.Done()** en loops
✅ **Establece timeouts razonables**
✅ **No reintentar después de DeadlineExceeded**

---

## ⚠️ Antipatrones

❌ Ignorar `context.Done()`
❌ No propagar context
❌ Timeouts muy cortos o muy largos
❌ No hacer defer cancel()
❌ Usar context como almacén de datos
❌ Ignorar DeadlineExceeded

---

## 📊 Timeout por Operación

```
Cliente HTTP Timeout:    30s
├─ Gateway Timeout:      25s
   ├─ Auth Service:      20s
   │  └─ Database:       15s
   ├─ Order Service:     20s
   │  ├─ Validation:     5s
   │  ├─ Inventory:      8s
   │  └─ Payment API:    10s
   └─ Response:          5s

Resultado: Todas las operaciones respetan timeout cascada
```

---

## 📚 Recursos

### Go Documentación
- context package: https://pkg.go.dev/context
- Context Best Practices: https://go.dev/blog/context

### Artículos
- "Context in Go" - Go Blog
- "Using Context" - Effective Go

---

## 💼 En Entrevistas

**Pregunta:** "¿Qué pasa si una query de base de datos se cuelga indefinidamente?"

**Respuesta:**
> "Sin context cancellation, la query sigue bloqueando el thread, consumiendo memoria, hasta que se agote o timeout del SO. Con context: establezco un timeout antes de la query usando `context.WithTimeout(ctx, 5*time.Second)`. Si la query tarda más de 5 segundos, context se cancela automáticamente y `db.QueryRowContext()` retorna `context.DeadlineExceeded`. De ahí puedo hacer cleanup y retornar error al cliente. Importante: propago el mismo context a través de capas (gateway → auth-service → database), así si el cliente cierra la conexión, todos los goroutines se detienen cascadamente, liberando recursos. En Kubernetes, esto combinado con graceful shutdown: cuando recibo SIGTERM, cancelo todos los contexts, requests terminan limpiamente, sin stalled queries."

---

#context #cancellation #timeouts #request-lifecycle #go
