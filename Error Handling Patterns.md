---
tags: [error-handling, reliability, resilience, best-practices, go]
date: 2026-01-05
related: [Status Codes, Observability, Graceful Degradation, Circuit Breaker]
status: reference
---

# Error Handling Patterns

## 📋 ¿Qué son Error Handling Patterns?

Estrategias y técnicas para **manejar, registrar y recuperarse de errores** de forma consistente, permitiendo que el sistema continúe funcionando o falle de manera controlada.

**Analogía:** Un piloto de avión:
- ❌ Ignorar errores: Desastre
- ❌ Parar todo: Sin función
- ✅ Identificar error, avisar, ajustar, continuar: Profesional

---

## 🎯 Problema que Resuelven

### Sin Manejo Adecuado

```go
// ❌ Ignorar errores
func RegisterUser(ctx context.Context, email, password string) (*User, error) {
    hashedPassword, _ := hashPassword(password)  // Error ignorado!
    user, _ := repo.Create(ctx, &User{...})     // Error ignorado!
    
    // Si hashPassword falla, guardamos contraseña en plain text ⚠️
    // Si repo.Create falla, función retorna User nil pero sin error
    return user, nil
}

// ❌ Propagación sin contexto
func GetUser(ctx context.Context, id string) (*User, error) {
    return repo.GetByID(ctx, id)
    // El error del repo se retorna sin saber dónde falló
}

// ❌ Pánico en producción
func ProcessPayment(amount float64) {
    if amount <= 0 {
        panic("Invalid amount!")  // Crash de la aplicación
    }
}

// ❌ Problemas:
// - Errores silenciosos
// - Sin contexto
// - Difícil debuggear
// - Sistema inestable
```

### Con Manejo Adecuado

```go
// ✅ Manejar cada error
func RegisterUser(ctx context.Context, email, password string) (*User, error) {
    // Validar entrada
    if email == "" {
        return nil, ErrInvalidEmail
    }
    
    // Hash password con manejo
    hashedPassword, err := hashPassword(password)
    if err != nil {
        return nil, fmt.Errorf("hash password failed: %w", err)
    }
    
    // Crear usuario con manejo
    user, err := repo.Create(ctx, &User{
        Email:    email,
        Password: hashedPassword,
    })
    
    if err != nil {
        if errors.Is(err, ErrUserExists) {
            return nil, ErrUserExists  // Error conocido
        }
        return nil, fmt.Errorf("create user failed: %w", err)  // Envolver con contexto
    }
    
    return user, nil
}

// ✅ Beneficios:
// - Errores explícitos
// - Contexto preservado
// - Fácil debuggear
// - Sistema predecible
```

---

## 🏗️ Patrones de Manejo

### 1. **Error as Value (Go idiomático)**

```go
// Go promueve retornar errores como valores
func (r *UserRepository) GetByID(ctx context.Context, id string) (*User, error) {
    if id == "" {
        return nil, ErrInvalidID
    }
    
    row := r.db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = ?", id)
    
    var user User
    err := row.Scan(&user.ID, &user.Email, ...)
    
    if err == sql.ErrNoRows {
        return nil, ErrUserNotFound
    }
    
    if err != nil {
        return nil, fmt.Errorf("scan user: %w", err)
    }
    
    return &user, nil
}

// Uso
user, err := repo.GetByID(ctx, "123")
if err != nil {
    if errors.Is(err, ErrUserNotFound) {
        // Manejar 404
    } else {
        // Manejar error genérico
        log.Errorf("Database error: %v", err)
    }
}
```

### 2. **Sentinel Errors (Errores Conocidos)**

```go
// Definir errores específicos
var (
    ErrInvalidEmail    = errors.New("invalid email format")
    ErrUserNotFound    = errors.New("user not found")
    ErrUserExists      = errors.New("user already exists")
    ErrInvalidPassword = errors.New("invalid password")
    ErrUnauthorized    = errors.New("unauthorized")
    ErrForbidden       = errors.New("forbidden")
)

// Usar en código
func (s *AuthService) Login(ctx context.Context, email, password string) (*User, error) {
    user, err := s.repo.GetByEmail(ctx, email)
    
    if err == ErrUserNotFound {
        return nil, ErrInvalidPassword  // No revelar si usuario existe
    }
    
    if !s.passwordHasher.Verify(user.Password, password) {
        return nil, ErrInvalidPassword
    }
    
    return user, nil
}

// Cliente puede usar errors.Is() para verificar
_, err := authService.Login(ctx, "test@test.com", "wrong")
if errors.Is(err, ErrInvalidPassword) {
    // Mostrar mensaje al usuario
    return "Invalid email or password"
}
```

### 3. **Custom Errors (Errores con Contexto)**

```go
// Tipo personalizado para errores complejos
type ValidationError struct {
    Field   string
    Message string
    Code    string
}

func (ve *ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s=%s", ve.Field, ve.Message)
}

type DatabaseError struct {
    Operation string
    Table     string
    Cause     error
    Timestamp time.Time
}

func (de *DatabaseError) Error() string {
    return fmt.Sprintf("db error [%s]: %s.%s: %v",
        de.Timestamp.Format("15:04:05"),
        de.Table,
        de.Operation,
        de.Cause,
    )
}

// Uso
func (r *UserRepository) Create(ctx context.Context, user *User) (*User, error) {
    if user.Email == "" {
        return nil, &ValidationError{
            Field:   "email",
            Message: "email is required",
            Code:    "REQUIRED",
        }
    }
    
    result, err := r.db.ExecContext(ctx, "INSERT INTO users...", user.Email)
    
    if err != nil {
        return nil, &DatabaseError{
            Operation: "insert",
            Table:     "users",
            Cause:     err,
            Timestamp: time.Now(),
        }
    }
    
    return user, nil
}

// Verificar tipo específico
if err != nil {
    var valErr *ValidationError
    if errors.As(err, &valErr) {
        // Retornar 400 Bad Request
        return fmt.Sprintf("Field '%s' error: %s", valErr.Field, valErr.Message)
    }
    
    var dbErr *DatabaseError
    if errors.As(err, &dbErr) {
        // Retornar 500 Internal Server Error
        log.Errorf("Database error: %v", dbErr)
        return "Internal server error"
    }
}
```

### 4. **Error Wrapping (Preservar Contexto)**

```go
// ❌ Perder contexto
err := operation()
if err != nil {
    return err  // ¿De dónde viene el error?
}

// ✅ Envolver para agregar contexto
func (s *AuthService) Register(ctx context.Context, email, password string) error {
    user, err := s.repo.Create(ctx, newUser)
    if err != nil {
        // Envolver: %w preserva el error original
        return fmt.Errorf("register user: %w", err)
    }
    
    err = s.emailSender.SendVerification(ctx, email)
    if err != nil {
        // Cada nivel agrega contexto
        return fmt.Errorf("send verification email: %w", err)
    }
    
    return nil
}

// Stack de errores:
// send verification email: smtp connection failed: connection refused

// Usar errors.Unwrap() para acceder al error original
err := s.Register(ctx, "test@test.com", "password")
cause := errors.Unwrap(err)  // smtp connection failed
```

### 5. **Graceful Degradation**

```go
// Si un servicio opcional falla, continuar
func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    order := parseOrder(r)
    
    // Crear orden (crítico)
    createdOrder, err := h.orderService.Create(order)
    if err != nil {
        http.Error(w, "Failed to create order", http.StatusInternalServerError)
        return
    }
    
    // Enviar email (no crítico)
    err = h.emailService.SendOrderConfirmation(createdOrder)
    if err != nil {
        // Log el error pero no falla la respuesta
        log.Warnf("Failed to send email: %v", err)
        // Continuar: usuario tiene su orden creada
    }
    
    // Actualizar analytics (no crítico)
    err = h.analytics.Track("order_created", createdOrder.ID)
    if err != nil {
        // Log pero no afecta al usuario
        log.Warnf("Analytics tracking failed: %v", err)
    }
    
    w.WriteJSON(createdOrder)  // Retorna éxito a pesar de servicios opcionales
}
```

### 6. **Retry Logic**

```go
import "github.com/cenkalti/backoff"

// Reintentar con backoff exponencial
func (c *AuthClient) LoginWithRetry(ctx context.Context, email, password string) (*pb.LoginResponse, error) {
    var lastErr error
    
    operation := func() error {
        resp, err := c.client.Login(ctx, &pb.LoginRequest{
            Email:    email,
            Password: password,
        })
        
        if err != nil {
            lastErr = err
            return err
        }
        
        return nil
    }
    
    // Reintentar hasta 3 veces con backoff exponencial
    err := backoff.RetryNotify(
        operation,
        backoff.WithMaxRetries(
            backoff.NewExponentialBackOff(),
            3,
        ),
        func(err error, duration time.Duration) {
            log.Warnf("Retry login after %v: %v", duration, err)
        },
    )
    
    return nil, err
}

// O implementación manual
func (c *AuthClient) LoginWithRetry(ctx context.Context, email, password string) (*pb.LoginResponse, error) {
    maxRetries := 3
    backoff := 100 * time.Millisecond
    
    for attempt := 0; attempt < maxRetries; attempt++ {
        resp, err := c.client.Login(ctx, &pb.LoginRequest{
            Email:    email,
            Password: password,
        })
        
        if err == nil {
            return resp, nil
        }
        
        if attempt < maxRetries-1 {
            select {
            case <-time.After(backoff):
                backoff *= 2  // Exponencial
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }
    }
    
    return nil, fmt.Errorf("login failed after %d retries", maxRetries)
}
```

---

## 🎯 Tu Proyecto: Error Handling

### Actual (sin envolver)

```go
// ❌ auth-service/internal/handler/grpc_auth_handler.go
func (h *AuthHandler) Register(ctx context.Context, req *pb.RegisterRequest) (*pb.User, error) {
    user, err := h.service.Register(ctx, req.Email, req.Password)
    if err != nil {
        return nil, status.Error(codes.Internal, "registration failed")
        // Error original perdido, sin contexto
    }
    return user, nil
}
```

### Mejorado (con envolver)

```go
// ✅ auth-service/internal/handler/grpc_auth_handler.go
func (h *AuthHandler) Register(ctx context.Context, req *pb.RegisterRequest) (*pb.User, error) {
    user, err := h.service.Register(ctx, req.Email, req.Password)
    if err != nil {
        // Envolver error con contexto
        wrappedErr := fmt.Errorf("register user %s: %w", req.Email, err)
        log.Errorf("Registration error: %v", wrappedErr)
        
        // Mapear a código gRPC apropiado
        code := mapErrorToCode(err)
        return nil, status.Error(code, "registration failed")
    }
    return user, nil
}

func mapErrorToCode(err error) codes.Code {
    if errors.Is(err, ErrUserExists) {
        return codes.AlreadyExists
    }
    if errors.Is(err, ErrInvalidEmail) {
        return codes.InvalidArgument
    }
    return codes.Internal
}
```

---

## 📊 Estrategia de Errores

```
Tipo de Error          Acción
─────────────────────────────────────
Validación (400)       Retornar al cliente, no reintentar
Autenticación (401)    Retornar al cliente, no reintentar
Autorización (403)     Retornar al cliente, no reintentar
No encontrado (404)    Retornar al cliente, no reintentar
Conflicto (409)        Retornar al cliente, puede reintentar

Timeout (408)          Reintentar con backoff
Unavailable (503)      Reintentar con backoff
Connection (5xx)       Reintentar con backoff

Internal (500)         Loguear, retornar error genérico
```

---

## ⚡ Best Practices

✅ **Siempre retorna error explícitamente**
✅ **Envuelve errores con contexto** (`%w` en fmt)
✅ **Define errores específicos** (sentinel errors)
✅ **Loguea errores** con stack trace
✅ **Mapea a códigos HTTP/gRPC**
✅ **Retry con backoff** para errores transitorios
✅ **Graceful degradation** para servicios opcionales
✅ **No expongas detalles internos** al cliente

---

## ⚠️ Antipatrones

❌ **Ignorar errores** (`_` sin razón)
❌ **Pánico en producción** (`panic()`)
❌ **No envolver errores** (perder contexto)
❌ **Retornar error genérico** (difícil debuggear)
❌ **Reintentar infinitamente** (timeout)
❌ **Exponer stack trace** al cliente

---

## 📚 Recursos

### Go
- "Error Handling" - Effective Go
- "Working with Errors" - Go Blog
- errors package: https://pkg.go.dev/errors

### Librerías
- github.com/pkg/errors (deprecated, usar std errors)
- github.com/cenkalti/backoff (retry)
- github.com/grpc-ecosystem/go-grpc-middleware (interceptors)

### Artículos
- "Error Handling in Go" - Medium
- "Sentinel Errors" - Dave Cheney

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo manejarías errores en un servicio microservicios?"

**Respuesta:**
> "Implementaría una estrategia multinivel: Primero, defino errores específicos (sentinel errors) para casos conocidos (UserNotFound, UserExists). Segundo, envuelvo errores con contexto usando %w de fmt, preservando el error original para debuggear. Tercero, mapeo errores a códigos apropiados: validación a 400, autenticación a 401, errores internos a 500. Cuarto, implemento retry con backoff exponencial para errores transitorios (timeout, connection failed). Quinto, logueo todos los errores con contexto completo. Para servicios opcionales, uso graceful degradation: si el servicio de email falla, la orden se crea igual pero sin email de confirmación. En Go, uso errors.Is() y errors.As() para verificar tipos específicos de errores."

---

#error-handling #reliability #go #best-practices #microservices
