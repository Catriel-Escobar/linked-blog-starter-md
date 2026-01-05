---
tags: [go, microservices, grpc, gateway, middleware, concurrency]
date: 2026-01-05
project: Auth Microservice + API Gateway
status: completed
---

# Microservicios con Go - Gateway Pattern + gRPC

## 📋 Resumen del Proyecto

Implementación de arquitectura de microservicios con:
- **API Gateway** (HTTP/REST) en Go con chi router
- **Auth Service** (gRPC) para autenticación
- **PostgreSQL** como base de datos
- **Docker Compose** para orquestación
- **Middleware pipeline** para cross-cutting concerns

---

## 🏗️ 1. Arquitectura de Microservicios

### Concepto
Separar una aplicación monolítica en servicios independientes que se comunican entre sí.

### Patrón Implementado: API Gateway

```
Cliente HTTP/REST
        ↓
   API Gateway (port 8080)
   - Timeout Middleware
   - JWT Middleware
   - CORS Middleware
        ↓
   Auth Service (gRPC port 50051)
   - Handler Layer
   - Service Layer
   - Repository Layer
        ↓
   PostgreSQL (port 5433)
```

### Por qué es valioso
- ✅ Escalabilidad independiente de cada servicio
- ✅ Tecnologías diferentes por servicio
- ✅ Deploy independiente
- ✅ Fallas aisladas (un servicio caído ≠ todo caído)
- ✅ Teams independientes pueden trabajar en paralelo

### Conceptos relacionados
- [[Service Mesh]]
- [[Circuit Breaker Pattern]]
- [[Event-Driven Architecture]]

---

## 🔌 2. gRPC - Comunicación entre Microservicios

### ¿Qué es gRPC?
Framework RPC (Remote Procedure Call) de Google usando HTTP/2 y Protocol Buffers.

### Ventajas vs REST
| Feature | REST | gRPC |
|---------|------|------|
| **Protocolo** | HTTP/1.1 | HTTP/2 |
| **Formato** | JSON/XML | Protocol Buffers |
| **Performance** | ~100ms | ~10ms |
| **Type Safety** | No | Sí (proto) |
| **Streaming** | No nativo | Bidireccional |
| **Tamaño** | ~500 bytes | ~50 bytes |

### Implementación

**Definir contrato (.proto):**
```proto
syntax = "proto3";

service AuthService {
  rpc Login(LoginRequest) returns (LoginResponse);
  rpc Register(RegisterRequest) returns (RegisterResponse);
}

message LoginRequest {
  string email = 1;
  string password = 2;
}
```

**Cliente gRPC:**
```go
type AuthClient struct {
    conn   *grpc.ClientConn
    client pb.AuthServiceClient
}

func NewAuthClient(url string) (*AuthClient, error) {
    conn, err := grpc.NewClient(url, 
        grpc.WithTransportCredentials(insecure.NewCredentials()))
    
    return &AuthClient{
        conn:   conn,
        client: pb.NewAuthServiceClient(conn),
    }, nil
}

func (c *AuthClient) Login(ctx context.Context, email, password string) (*pb.LoginResponse, error) {
    return c.client.Login(ctx, &pb.LoginRequest{
        Email:    email,
        Password: password,
    })
}
```

**Servidor gRPC:**
```go
type GRPCAuthHandler struct {
    proto.UnimplementedAuthServiceServer
    authService ports.AuthService
}

func (h *GRPCAuthHandler) Login(ctx context.Context, req *proto.LoginRequest) (*proto.LoginResponse, error) {
    token, refreshToken, user, err := h.authService.Login(ctx, req.Email, req.Password)
    if err != nil {
        return nil, mapErrorToGRPCStatus(err)
    }
    
    return &proto.LoginResponse{
        AccessToken:  token,
        RefreshToken: refreshToken,
        User:         user,
    }, nil
}
```

### Cuándo usar gRPC
- ✅ Comunicación interna entre microservicios
- ✅ Performance crítico
- ✅ Streaming de datos
- ✅ Contratos estrictos (type-safe)

### Cuándo usar REST
- ✅ APIs públicas
- ✅ Browsers (no soportan gRPC nativamente)
- ✅ Documentación fácil (Swagger)
- ✅ Debugging simple (curl)

### Conceptos relacionados
- [[Protocol Buffers]]
- [[HTTP/2]]
- [[Streaming APIs]]

---

## 🎭 3. Middleware Pattern

### Concepto
Funciones que se ejecutan **antes** o **después** del handler principal, permitiendo agregar funcionalidades sin modificar el handler.

### Patrón de Cadena

```go
Request
   ↓
Timeout Middleware
   ↓
JWT Middleware
   ↓
Logging Middleware
   ↓
Handler
   ↓
Response
```

### Anatomía de un Middleware en Go

```go
func MyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // BEFORE: Código antes del handler
        log.Println("Request received")
        
        // Llamar al siguiente handler
        next.ServeHTTP(w, r)
        
        // AFTER: Código después del handler
        log.Println("Response sent")
    })
}
```

### Middleware Implementado: Timeout

**Problema:** Necesitamos timeouts diferentes por ruta/servicio sin repetir código.

**Solución:**
```go
type TimeoutConfig map[string]time.Duration

func Timeout(config TimeoutConfig, defaultTimeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 1. Determinar timeout según ruta
            timeout := getTimeoutForPath(r.URL.Path, config, defaultTimeout)
            
            // 2. Crear contexto con timeout
            ctx, cancel := context.WithTimeout(r.Context(), timeout)
            defer cancel()
            
            // 3. Canal para sincronización
            done := make(chan struct{})
            
            // 4. Ejecutar handler en goroutine
            go func() {
                next.ServeHTTP(tw, r.WithContext(ctx))
                close(done)
            }()
            
            // 5. Esperar resultado o timeout
            select {
            case <-done:
                return  // Success
            case <-ctx.Done():
                if !tw.wroteHeader {
                    http.Error(w, "Request timeout", 408)
                }
            }
        })
    }
}
```

**Configuración:**
```go
timeouts := TimeoutConfig{
    "/api/v1/auth/*":    500 * time.Millisecond,
    "/api/v1/orders/*":  2 * time.Second,
    "/api/v1/payments/*": 5 * time.Second,
}

router.Use(middleware.Timeout(timeouts, 10*time.Second))
```

### Middleware Implementado: JWT

```go
func JWT(secret []byte) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 1. Extraer token del header
            authHeader := r.Header.Get("Authorization")
            tokenString := strings.TrimPrefix(authHeader, "Bearer ")
            
            // 2. Validar token
            token, err := jwt.Parse(tokenString, func(t *jwt.Token) (any, error) {
                return secret, nil
            })
            
            if err != nil || !token.Valid {
                http.Error(w, "Unauthorized", 401)
                return
            }
            
            // 3. Extraer claims
            claims := token.Claims.(jwt.MapClaims)
            userID := claims["sub"].(string)
            
            // 4. Inyectar en contexto
            ctx := context.WithValue(r.Context(), "user_id", userID)
            
            // 5. Continuar
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### Usos Comunes de Middleware
- Authentication/Authorization
- Logging
- Metrics/Observability
- Rate Limiting
- CORS
- Compression
- Request tracing
- Error recovery

### Conceptos relacionados
- [[Decorator Pattern]]
- [[Chain of Responsibility]]
- [[Cross-Cutting Concerns]]

---

## 🎁 4. Wrapper Pattern - Decorating Objects

### Concepto
Envolver un objeto para agregar funcionalidad sin modificar el original.

### Problema Resuelto
En el middleware de timeout, necesitábamos:
- Detectar si el handler ya escribió una respuesta
- Evitar que timeout escriba si handler ya escribió
- Thread-safe (goroutines concurrentes)

### Implementación

```go
// Wrapper que envuelve http.ResponseWriter
type timeoutWriter struct {
    http.ResponseWriter  // Composición (embedded field)
    mu          sync.Mutex
    wroteHeader bool     // Estado adicional
}

// Override del método Write
func (tw *timeoutWriter) Write(b []byte) (int, error) {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    
    if !tw.wroteHeader {
        tw.wroteHeader = true
    }
    
    return tw.ResponseWriter.Write(b)
}

// Override del método WriteHeader
func (tw *timeoutWriter) WriteHeader(code int) {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    
    if !tw.wroteHeader {
        tw.wroteHeader = true
        tw.ResponseWriter.WriteHeader(code)
    }
}
```

### Uso en Middleware

```go
// Crear wrapper
tw := &timeoutWriter{ResponseWriter: w, wroteHeader: false}

// Pasar wrapper al handler (en lugar del ResponseWriter original)
next.ServeHTTP(tw, r)

// Verificar si ya se escribió
if !tw.wroteHeader {
    http.Error(tw.ResponseWriter, "Timeout", 408)
}
```

### Por qué funciona
1. **Composición en Go**: `timeoutWriter` "hereda" todos los métodos de `http.ResponseWriter`
2. **Override selectivo**: Solo reemplazamos `Write()` y `WriteHeader()`
3. **Transparente**: El handler no sabe que está usando un wrapper

### Otros Usos de Wrappers
```go
// Logging wrapper
type loggingWriter struct {
    http.ResponseWriter
    statusCode int
}

// Compression wrapper
type gzipWriter struct {
    http.ResponseWriter
    Writer *gzip.Writer
}

// Metrics wrapper
type metricsWriter struct {
    http.ResponseWriter
    bytesWritten int
}
```

### Conceptos relacionados
- [[Decorator Pattern]]
- [[Composition Over Inheritance]]
- [[Open/Closed Principle]]

---

## ⚠️ 5. Manejo de Errores Multi-Capa

### Problema
En una arquitectura de microservicios con gRPC, los errores necesitan:
1. Ser específicos en el servicio (gRPC codes)
2. Traducirse correctamente a HTTP en el gateway
3. Ser user-friendly para el cliente
4. Permitir debugging (logs con contexto)

### Códigos gRPC → HTTP

| gRPC Code | HTTP Status | Uso |
|-----------|-------------|-----|
| `OK` | 200 | Éxito |
| `InvalidArgument` | 400 | Validación fallida |
| `Unauthenticated` | 401 | Login fallido, token inválido |
| `PermissionDenied` | 403 | Sin permisos |
| `NotFound` | 404 | Recurso no existe |
| `AlreadyExists` | 409 | Email duplicado |
| `FailedPrecondition` | 412 | Email no verificado |
| `Internal` | 500 | Error interno |
| `Unavailable` | 503 | Servicio caído |
| `DeadlineExceeded` | 504 | Timeout |

### Implementación en Auth Service

**1. Definir errores de negocio:**
```go
// auth-service/internal/service/auth_service.go
var (
    ErrInvalidCredentials  = errors.New("invalid credentials")
    ErrEmailAlreadyExists  = errors.New("email already exists")
    ErrUserNotFound        = errors.New("user not found")
    ErrEmailNotVerified    = errors.New("email not verified")
    ErrInvalidToken        = errors.New("invalid or expired token")
)
```

**2. Mapear errores a gRPC:**
```go
func mapErrorToGRPCStatus(err error) error {
    if err == nil {
        return nil
    }

    // Nivel 1: Errores de negocio
    switch err {
    case service.ErrInvalidCredentials:
        return status.Error(codes.Unauthenticated, "Invalid email or password")
    case service.ErrEmailAlreadyExists:
        return status.Error(codes.AlreadyExists, "Email already registered")
    case service.ErrUserNotFound:
        return status.Error(codes.NotFound, "User not found")
    }

    // Nivel 2: Errores de infraestructura (DB)
    errMsg := err.Error()
    
    if strings.Contains(errMsg, "relation") && strings.Contains(errMsg, "does not exist") {
        return status.Error(codes.FailedPrecondition, "Database not initialized")
    }
    
    if strings.Contains(errMsg, "connection refused") {
        return status.Error(codes.Unavailable, "Database connection failed")
    }

    // Nivel 3: Error genérico
    return status.Error(codes.Internal, "Internal server error")
}
```

**3. Handler devuelve error gRPC:**
```go
func (h *GRPCAuthHandler) Register(ctx context.Context, req *proto.RegisterRequest) (*proto.RegisterResponse, error) {
    err := h.authService.Register(ctx, req.Email, req.Password)
    if err != nil {
        return nil, mapErrorToGRPCStatus(err)  // ✅ Error gRPC
    }
    
    return &proto.RegisterResponse{
        Success: true,
        Message: "User registered successfully",
    }, nil
}
```

### Implementación en Gateway

**Mapear gRPC → HTTP:**
```go
func handleGRPCError(w http.ResponseWriter, err error) {
    st, ok := status.FromError(err)
    if !ok {
        respondWithError(w, 500, "Internal server error", "")
        return
    }

    var httpStatus int
    switch st.Code() {
    case codes.NotFound:
        httpStatus = http.StatusNotFound
    case codes.AlreadyExists:
        httpStatus = http.StatusConflict
    case codes.InvalidArgument:
        httpStatus = http.StatusBadRequest
    case codes.Unauthenticated:
        httpStatus = http.StatusUnauthorized
    case codes.PermissionDenied:
        httpStatus = http.StatusForbidden
    case codes.DeadlineExceeded:
        httpStatus = http.StatusRequestTimeout
    case codes.Unavailable:
        httpStatus = http.StatusServiceUnavailable
    default:
        httpStatus = http.StatusInternalServerError
    }

    respondWithError(w, httpStatus, st.Message(), "")
}
```

### Flujo Completo

```
1. Service Layer
   return service.ErrInvalidCredentials
       ↓
2. Handler gRPC
   return nil, status.Error(codes.Unauthenticated, "Invalid credentials")
       ↓ (gRPC por red)
3. Gateway Client
   resp, err := client.Login(...)
   if err != nil { handleGRPCError(w, err) }
       ↓
4. Gateway Handler
   st.Code() == codes.Unauthenticated → HTTP 401
       ↓
5. Cliente HTTP
   HTTP/1.1 401 Unauthorized
   {"error": "Invalid credentials", "code": 401}
```

### Mejores Prácticas
- ✅ Errores de negocio → `var Err...` específicos
- ✅ Mensajes user-friendly (no detalles técnicos)
- ✅ Log detalles internos, retornar mensajes simples
- ✅ Distinguir 4xx (cliente) vs 5xx (servidor)
- ❌ Nunca exponer stack traces al cliente
- ❌ Nunca poner errores en campos del mensaje (Success: false)

### Conceptos relacionados
- [[Error Handling Patterns]]
- [[Status Codes]]
- [[Observability]]

---

## ⏱️ 6. Context Propagation

### ¿Qué es Context?
Un objeto que transporta:
- **Deadlines/Timeouts**: Cuándo cancelar operaciones
- **Cancelación**: Señal para abortar trabajo
- **Valores**: Metadata (user_id, request_id, etc.)

### Por qué es crítico
```go
// Sin context
func SlowOperation() {
    // Si el cliente cancela, esto sigue ejecutando
    time.Sleep(10 * time.Second)
}

// Con context
func SlowOperation(ctx context.Context) error {
    select {
    case <-time.After(10 * time.Second):
        return nil
    case <-ctx.Done():
        return ctx.Err()  // Cliente canceló, abortar
    }
}
```

### Timeout en Cascada

```go
// HTTP Request (cliente espera 5s)
ctx := r.Context()  // Timeout: 5s
    ↓
// Middleware agrega timeout más estricto
ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
    ↓
// gRPC Client usa ese context
client.Login(ctx, email, password)
    ↓
// Auth Service verifica context
if ctx.Err() == context.DeadlineExceeded {
    return nil, status.Error(codes.DeadlineExceeded, "Timeout")
}
```

### Propagación de Metadata

```go
// Gateway agrega metadata al context
ctx = metadata.AppendToOutgoingContext(ctx,
    "user_id", userID,
    "request_id", requestID,
    "client_ip", clientIP,
)

// Auth Service extrae metadata
md, _ := metadata.FromIncomingContext(ctx)
userID := md.Get("user_id")[0]
```

### Mejores Prácticas
- ✅ Siempre pasar `context.Context` como **primer parámetro**
- ✅ Usar `context.Background()` solo en main/tests
- ✅ Propagar context a través de toda la cadena
- ✅ Usar `defer cancel()` después de `WithTimeout`
- ❌ Nunca guardar context en structs
- ❌ Nunca pasar `nil` como context

### Usos Avanzados
```go
// Cancelación manual
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

go func() {
    if error := doWork(ctx) {
        cancel()  // Cancelar todo si hay error
    }
}()

// Timeout con deadline
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)

// Valores en context (con key type-safe)
type ctxKey string
const userIDKey ctxKey = "user_id"

ctx = context.WithValue(ctx, userIDKey, "user-123")
userID := ctx.Value(userIDKey).(string)
```

### Conceptos relacionados
- [[Distributed Tracing]]
- [[Graceful Shutdown]]
- [[Request Cancellation]]

---

## 🧵 7. Concurrencia en Go

### Goroutines

**¿Qué son?**
Threads ligeros manejados por Go runtime (no threads del OS).

```go
// Secuencial (2 segundos)
doWork1()  // 1 segundo
doWork2()  // 1 segundo

// Concurrente (1 segundo)
go doWork1()  // No bloquea
go doWork2()  // No bloquea
time.Sleep(2 * time.Second)  // Esperar a que terminen
```

**Uso en Timeout Middleware:**
```go
done := make(chan struct{})

// Lanzar handler en goroutine
go func() {
    next.ServeHTTP(w, r)
    close(done)
}()

// Mientras tanto, monitorear timeout
select {
case <-done:
    // Handler terminó
case <-time.After(500 * time.Millisecond):
    // Timeout!
}
```

### Channels

**¿Qué son?**
Tubos para comunicación entre goroutines.

```go
// Crear canal
ch := make(chan string)

// Enviar (bloquea hasta que alguien reciba)
go func() {
    ch <- "hello"
}()

// Recibir (bloquea hasta que alguien envíe)
msg := <-ch
```

**Tipos de canales:**
```go
// Unbuffered (bloqueante)
ch := make(chan int)

// Buffered (no bloquea hasta llenar buffer)
ch := make(chan int, 10)

// Solo envío
ch := make(chan<- int)

// Solo recepción
ch := make(<-chan int)

// Cerrar canal
close(ch)
```

### Select - Multiplexing de Canales

```go
select {
case msg := <-ch1:
    fmt.Println("Received from ch1:", msg)
case msg := <-ch2:
    fmt.Println("Received from ch2:", msg)
case <-time.After(1 * time.Second):
    fmt.Println("Timeout")
default:
    fmt.Println("No messages")
}
```

**Uso en Timeout Middleware:**
```go
select {
case <-done:          // Handler terminó primero
    return
case <-ctx.Done():    // Timeout ocurrió primero
    http.Error(w, "Timeout", 408)
}
```

### Mutex - Protección de Race Conditions

**Problema:**
```go
var counter int

go func() { counter++ }()  // Goroutine 1
go func() { counter++ }()  // Goroutine 2

// Race condition: resultado impredecible
```

**Solución con Mutex:**
```go
var (
    counter int
    mu      sync.Mutex
)

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()
```

**Uso en timeoutWriter:**
```go
type timeoutWriter struct {
    http.ResponseWriter
    mu          sync.Mutex
    wroteHeader bool
}

func (tw *timeoutWriter) Write(b []byte) (int, error) {
    tw.mu.Lock()            // Bloquear acceso
    defer tw.mu.Unlock()    // Desbloquear al salir
    
    if !tw.wroteHeader {
        tw.wroteHeader = true
    }
    return tw.ResponseWriter.Write(b)
}
```

### Patrones Comunes

**1. Worker Pool:**
```go
jobs := make(chan int, 100)
results := make(chan int, 100)

// Lanzar workers
for w := 1; w <= 3; w++ {
    go worker(w, jobs, results)
}

// Enviar trabajos
for j := 1; j <= 9; j++ {
    jobs <- j
}
close(jobs)

// Recoger resultados
for a := 1; a <= 9; a++ {
    <-results
}
```

**2. Fan-out, Fan-in:**
```go
// Fan-out: distribuir trabajo a múltiples goroutines
func fanOut(input <-chan int) []<-chan int {
    outputs := make([]<-chan int, 3)
    for i := range outputs {
        outputs[i] = process(input)
    }
    return outputs
}

// Fan-in: combinar resultados de múltiples goroutines
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    for _, ch := range channels {
        go func(c <-chan int) {
            for v := range c {
                out <- v
            }
        }(ch)
    }
    return out
}
```

### Mejores Prácticas
- ✅ Siempre usar `defer` para desbloquear mutex
- ✅ Detectar race conditions con `go run -race`
- ✅ Cerrar canales solo desde el sender
- ✅ Usar `sync.WaitGroup` para esperar goroutines
- ❌ Nunca pasar mutex por valor (usar puntero)
- ❌ Evitar goroutine leaks (siempre tener forma de terminar)

### Conceptos relacionados
- [[CSP (Communicating Sequential Processes)]]
- [[Race Conditions]]
- [[Deadlocks]]

---

## 🐳 8. Docker & Orquestación

### Multi-Stage Builds

**Ventajas:**
- Imagen final pequeña (solo runtime, sin compilador)
- Layer caching (rebuild rápido si no cambió go.mod)
- Seguridad (no incluye herramientas de build)

```dockerfile
# ============ Stage 1: Builder ============
FROM golang:1.23-alpine AS builder

WORKDIR /app

# Copiar solo go.mod/go.sum (cache layer)
COPY go.mod go.sum ./
RUN go mod download

# Copiar código y compilar
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app ./cmd/server

# ============ Stage 2: Runtime ============
FROM alpine:3.20

WORKDIR /app

# Solo copiar binario (imagen pequeña)
COPY --from=builder /app/app /app/app

RUN chmod +x /app/app

CMD ["/app/app"]
```

### Docker Compose - Orquestación

```yaml
version: '3'

services:
  postgres-auth:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: authuser
      POSTGRES_PASSWORD: authpassword
      POSTGRES_DB: authdb
    ports:
      - "5433:5432"
    volumes:
      - postgres_auth_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U authuser"]
      interval: 10s
      timeout: 5s
      retries: 5

  auth-service:
    build:
      context: ./../auth-service
      dockerfile: ./auth-service.dockerfile
    ports:
      - "50051:50051"
    environment:
      DB_URI: "host=postgres-auth port=5432 user=authuser password=authpassword dbname=authdb"
    depends_on:
      postgres-auth:
        condition: service_healthy  # Espera a que DB esté ready

  gateway:
    build:
      context: ./../gateway
      dockerfile: ./gateway-service.dockerfile
    ports:
      - "8080:80"
    environment:
      AUTH_SERVICE_URL: "auth-service:50051"  # DNS interno
    depends_on:
      - auth-service

volumes:
  postgres_auth_data:
```

### Networking en Docker Compose

**DNS automático:**
```
gateway → auth-service:50051
         ↓
    auth-service → postgres-auth:5432
```

Los nombres de servicio se resuelven automáticamente en la red interna.

### Health Checks

**Por qué son importantes:**
- ✅ Servicio no inicia hasta que dependencias estén ready
- ✅ Evita errores de "connection refused" al inicio
- ✅ Restart automático si health check falla

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U authuser"]
  interval: 10s      # Verificar cada 10s
  timeout: 5s        # Dar 5s para responder
  retries: 5         # 5 intentos antes de marcar unhealthy
```

### Variables de Entorno

**12 Factor App: Config separado del código**

```yaml
# docker-compose.yml
environment:
  AUTH_SERVICE_TIMEOUT: "500ms"
  JWT_SECRET: ${JWT_SECRET}  # Desde .env file

# .env (no commitear)
JWT_SECRET=super-secret-key
```

**En código:**
```go
func getEnv(key, fallback string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return fallback
}

timeout := getDurationEnv("AUTH_SERVICE_TIMEOUT", 500*time.Millisecond)
```

### Makefile para Automatización

```makefile
SHELL=cmd.exe

up:
	docker-compose up -d

up_build: build_auth build_gateway
	docker-compose down
	docker-compose up --build -d

down:
	docker-compose down

build_auth:
	cd ..\auth-service && go build -o authApp ./cmd/server

logs:
	docker-compose logs -f
```

### Conceptos relacionados
- [[Kubernetes]]
- [[Service Discovery]]
- [[Container Orchestration]]

---

## 🎯 Patrones de Diseño Aplicados

### 1. Gateway Pattern
- Single entry point para múltiples servicios
- Traducción de protocolos (HTTP ↔ gRPC)
- Agregación de respuestas

### 2. Repository Pattern
- Abstracción de acceso a datos
- Facilita testing (mock repositories)
- Cambiar DB sin afectar lógica de negocio

```go
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    FindByEmail(ctx context.Context, email string) (*User, error)
}

// Implementación PostgreSQL
type userRepositoryPG struct {
    db *sqlx.DB
}
```

### 3. Dependency Injection
- Inyectar dependencias en constructores
- Facilita testing
- Reduce acoplamiento

```go
type AuthService struct {
    userRepo    ports.UserRepository
    tokenRepo   ports.TokenRepository
}

func NewAuthService(
    userRepo ports.UserRepository,
    tokenRepo ports.TokenRepository,
) *AuthService {
    return &AuthService{
        userRepo: userRepo,
        tokenRepo: tokenRepo,
    }
}
```

### 4. Middleware/Chain of Responsibility
- Pipeline de procesamiento
- Agregar funcionalidades sin modificar código
- Cross-cutting concerns

### 5. Decorator (Wrapper)
- Agregar funcionalidad a objetos existentes
- Sin modificar el original
- Composición sobre herencia

---

## 📊 Clean Architecture / Hexagonal

### Capas Implementadas

```
┌─────────────────────────────────────────┐
│           Handler Layer (gRPC)          │
│  - Recibe requests                      │
│  - Valida entrada                       │
│  - Traduce errores                      │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│          Service Layer (Business)       │
│  - Lógica de negocio                   │
│  - Validaciones complejas              │
│  - Orquestación                        │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│        Repository Layer (Data)          │
│  - Acceso a BD                         │
│  - Queries SQL                         │
│  - Transacciones                       │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│              Database                   │
└─────────────────────────────────────────┘
```

### Ports & Adapters

**Ports (Interfaces):**
```go
// domain/ports/user_port.go
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    FindByEmail(ctx context.Context, email string) (*User, error)
}

type AuthService interface {
    Register(ctx context.Context, email, password string) error
    Login(ctx context.Context, email, password string) (string, string, *User, error)
}
```

**Adapters (Implementaciones):**
```go
// repository/user_repository_pg.go (Adapter para PostgreSQL)
type userRepositoryPG struct {
    db *sqlx.DB
}

// handler/grpc_auth_handler.go (Adapter para gRPC)
type GRPCAuthHandler struct {
    authService ports.AuthService
}
```

### Ventajas
- ✅ Testeable (mock de interfaces)
- ✅ Cambiar tecnología sin afectar lógica
- ✅ Lógica de negocio aislada
- ✅ Independiente de frameworks

---

## 🔐 JWT Authentication

### Flow Completo

```
1. Usuario → POST /register
   Gateway → Auth Service (gRPC)
   Auth Service → Crea usuario en DB
   
2. Usuario → POST /login
   Gateway → Auth Service (gRPC)
   Auth Service → Valida credenciales
   Auth Service → Genera access_token + refresh_token
   
3. Usuario → GET /protected (con Authorization: Bearer <token>)
   Gateway → Middleware JWT valida token
   Gateway → Extrae user_id de token
   Gateway → Inyecta user_id en contexto
   Handler → Usa user_id del contexto
```

### Estructura del JWT

```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user-id-123",
    "email": "user@example.com",
    "roles": ["user", "admin"],
    "iat": 1609459200,
    "exp": 1609462800
  },
  "signature": "..."
}
```

### Implementación

**Generar token:**
```go
func GenerateToken(userID, email string, roles []string, duration time.Duration, secret []byte) (string, error) {
    claims := jwt.MapClaims{
        "sub":   userID,
        "email": email,
        "roles": roles,
        "iat":   time.Now().Unix(),
        "exp":   time.Now().Add(duration).Unix(),
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secret)
}
```

**Validar token:**
```go
func ValidateToken(tokenString string, secret []byte) (*jwt.MapClaims, error) {
    token, err := jwt.Parse(tokenString, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method")
        }
        return secret, nil
    })
    
    if err != nil || !token.Valid {
        return nil, err
    }
    
    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok {
        return nil, fmt.Errorf("invalid claims")
    }
    
    return &claims, nil
}
```

### Access Token vs Refresh Token

| Feature | Access Token | Refresh Token |
|---------|-------------|---------------|
| **Duración** | Corta (15min) | Larga (7 días) |
| **Uso** | Cada request | Solo /refresh |
| **Almacenado** | Memoria | DB |
| **Revocable** | No (stateless) | Sí (DB) |

---

## 📈 Observabilidad (Siguiente Paso)

### Logging
```go
log.Printf("[INFO] User %s logged in", userID)
log.Printf("[ERROR] DB connection failed: %v", err)
```

### Metrics (Prometheus)
```go
httpRequestsTotal.WithLabelValues("POST", "/login", "200").Inc()
httpRequestDuration.WithLabelValues("POST", "/login").Observe(duration.Seconds())
```

### Tracing (Jaeger)
```go
span := tracer.StartSpan("auth.Login")
defer span.Finish()

span.SetTag("user_id", userID)
span.SetTag("email", email)
```

---

## 🎓 Recursos para Profundizar

### Libros
- **"Designing Data-Intensive Applications"** - Martin Kleppmann
- **"Building Microservices"** - Sam Newman
- **"Go Programming Patterns"** - Packt

### Videos/Cursos
- **Microservices with Go** - Nic Jackson (YouTube)
- **gRPC Course** - freeCodeCamp

### Documentación
- gRPC Official: https://grpc.io
- Go Concurrency Patterns: https://go.dev/blog/pipelines

---

## ✅ Checklist de Conceptos Dominados

- [x] Arquitectura de Microservicios
- [x] API Gateway Pattern
- [x] gRPC & Protocol Buffers
- [x] HTTP/2 basics
- [x] Middleware Pattern
- [x] Wrapper/Decorator Pattern
- [x] Context Propagation
- [x] Timeout Management
- [x] Error Handling Multi-Capa
- [x] gRPC codes → HTTP status
- [x] Goroutines & Concurrency
- [x] Channels & Select
- [x] Mutex & Race Conditions
- [x] Clean Architecture
- [x] Repository Pattern
- [x] Dependency Injection
- [x] JWT Authentication
- [x] Docker Multi-Stage Builds
- [x] Docker Compose Orchestration
- [x] Health Checks
- [x] Configuration Management

---

## 🚀 Próximos Pasos

### Nivel Intermedio
- [ ] Circuit Breaker Pattern
- [ ] Rate Limiting
- [ ] API Versioning
- [ ] Observability (Prometheus + Jaeger)
- [ ] Unit Tests + Integration Tests
- [ ] Database Migrations

### Nivel Avanzado
- [ ] Service Mesh (Istio/Linkerd)
- [ ] Event-Driven Architecture (Kafka)
- [ ] CQRS Pattern
- [ ] Saga Pattern (Distributed Transactions)
- [ ] Feature Flags
- [ ] Blue-Green Deployment

---

## 💼 Aplicación Profesional

### Para Entrevistas
"Implementé una arquitectura de microservicios usando Go con un API Gateway que expone REST al cliente pero se comunica internamente vía gRPC para eficiencia. Manejé timeouts configurables por servicio usando middleware con goroutines y channels, implementé autenticación JWT con propagación de contexto, y orquesté todo con Docker Compose incluyendo health checks y configuración externalizada."

### Skills Destacados
- Microservices Architecture ⭐⭐⭐⭐⭐
- gRPC & Protocol Buffers ⭐⭐⭐⭐⭐
- Middleware Pattern ⭐⭐⭐⭐⭐
- Go Concurrency ⭐⭐⭐⭐⭐
- Clean Architecture ⭐⭐⭐⭐
- Docker/DevOps ⭐⭐⭐⭐

---

#go #microservices #grpc #architecture #concurrency #docker
