---
tags: [http, grpc, rest, status-codes, semantics, error-codes]
date: 2026-01-05
related: [Error Handling Patterns, gRPC, REST APIs, HTTP Protocol]
status: reference
---

# Status Codes

## 📋 ¿Qué son Status Codes?

Números estandarizados que comunican el **resultado de una solicitud** (éxito, error, redirección) entre cliente y servidor.

**Estructura:**
- **1xx:** Informativos (raramente usados)
- **2xx:** Éxito
- **3xx:** Redirección
- **4xx:** Error del cliente (solicitud inválida)
- **5xx:** Error del servidor

---

## 🎯 Problema que Resuelven

### Sin Status Codes (JSON Response)

```json
{
  "success": false,
  "message": "User not found",
  "error_code": "USER_NOT_FOUND"
}

{
  "success": false,
  "message": "Invalid email format",
  "error_code": "INVALID_EMAIL"
}

{
  "success": false,
  "message": "Internal database error",
  "error_code": "DB_ERROR"
}

// ❌ Problemas:
// - Todo retorna 200 OK (cliente no sabe si falló)
// - Cliente debe parsear JSON para saber qué pasó
// - No hay semántica HTTP
// - Caché rompe (caches errores como 200)
// - Proxies/CDN no entienden
```

### Con Status Codes

```
GET /users/123
← 404 Not Found
← Content-Type: application/json
{
  "error": "user not found"
}

GET /users (body inválido)
← 400 Bad Request
← Content-Type: application/json
{
  "error": "invalid email format"
}

GET /users (DB down)
← 500 Internal Server Error
← Content-Type: application/json
{
  "error": "database connection failed"
}

// ✅ Ventajas:
// - Semántica clara (cliente entiende sin parsear)
// - Caché funciona (no cachea 404/500)
// - Proxies/CDN pueden optimizar
// - Estándar HTTP seguido
```

---

## 📊 Status Codes Comunes

### 2xx - Éxito

```
200 OK
  GET /users/123
  ← Retorna usuario
  
201 Created
  POST /users
  ← Usuario creado
  ← Location: /users/456
  
202 Accepted
  POST /async-job
  ← Job aceptado, procesándose
  
204 No Content
  DELETE /users/123
  ← Eliminado, sin body
  
206 Partial Content
  GET /file (con Range header)
  ← Descarga parcial
```

### 3xx - Redirección

```
301 Moved Permanently
  GET /old-url
  ← Location: /new-url
  (cacheable, cambio permanente)
  
302 Found
  GET /login
  ← Location: /dashboard (después de login)
  (no cacheable, cambio temporal)
  
304 Not Modified
  GET /file (con If-Modified-Since)
  ← No hay cambios (usar caché)
```

### 4xx - Error del Cliente

```
400 Bad Request
  POST /users (body malformado)
  ← {"error": "invalid json"}
  
401 Unauthorized
  GET /orders (sin token)
  ← {"error": "missing authorization token"}
  
403 Forbidden
  GET /admin (usuario sin permisos)
  ← {"error": "insufficient permissions"}
  
404 Not Found
  GET /users/999 (no existe)
  ← {"error": "user not found"}
  
409 Conflict
  POST /users (email duplicado)
  ← {"error": "user already exists"}
  
422 Unprocessable Entity
  POST /users (validación de negocio falla)
  ← {"error": "password too weak"}
  
429 Too Many Requests
  GET /api (rate limit excedido)
  ← {"error": "rate limit exceeded"}
```

### 5xx - Error del Servidor

```
500 Internal Server Error
  GET /complex-operation
  ← Error inesperado
  
502 Bad Gateway
  GET /api (upstream service down)
  ← Servicio dependiente no responde
  
503 Service Unavailable
  GET /api (mantenimiento)
  ← Servicio temporalmente no disponible
  
504 Gateway Timeout
  GET /slow-op (timeout)
  ← Servicio tardó demasiado
```

---

## 🏗️ Semántica de Status Codes

### Elección Correcta

```
Usuario intenta registrase con email que ya existe:

❌ 500 Internal Server Error
   (No es culpa del servidor)

❌ 400 Bad Request
   (El email es válido, el problema es conflicto)

✅ 409 Conflict
   (Email existe = conflicto)
   
Response:
{
  "error": "user_already_exists",
  "message": "Email already registered"
}
```

### Otra Decisión

```
Cliente envía JSON malformado:

❌ 200 OK
   (No fue exitoso)

✅ 400 Bad Request
   (JSON inválido = cliente error)
   
Response:
{
  "error": "invalid_json",
  "message": "Unexpected token at position 45"
}
```

---

## 💻 En Go: HTTP Status Codes

### Patrón Estándar

```go
import "net/http"

type ErrorResponse struct {
    Error   string      `json:"error"`
    Message string      `json:"message"`
    Code    string      `json:"code,omitempty"`
}

// Handler
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    
    user, err := h.repo.GetByID(r.Context(), id)
    
    // Validación
    if id == "" {
        w.WriteHeader(http.StatusBadRequest)  // 400
        json.NewEncoder(w).Encode(ErrorResponse{
            Error:   "invalid_id",
            Message: "ID is required",
        })
        return
    }
    
    // No encontrado
    if err == sql.ErrNoRows {
        w.WriteHeader(http.StatusNotFound)  // 404
        json.NewEncoder(w).Encode(ErrorResponse{
            Error:   "user_not_found",
            Message: fmt.Sprintf("User %s not found", id),
        })
        return
    }
    
    // Error interno
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)  // 500
        json.NewEncoder(w).Encode(ErrorResponse{
            Error:   "internal_error",
            Message: "Failed to retrieve user",
        })
        log.Errorf("Database error: %v", err)
        return
    }
    
    // Éxito
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)  // 200
    json.NewEncoder(w).Encode(user)
}
```

### Patrón Decorator (Middleware)

```go
type ErrorHandler func(http.ResponseWriter, *http.Request) error

func HandleError(fn ErrorHandler) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        err := fn(w, r)
        
        if err == nil {
            return
        }
        
        // Mapear error a status code
        var statusCode int
        var message string
        
        switch {
        case errors.Is(err, ErrInvalidInput):
            statusCode = http.StatusBadRequest  // 400
            message = "Invalid input"
            
        case errors.Is(err, ErrUserNotFound):
            statusCode = http.StatusNotFound  // 404
            message = "User not found"
            
        case errors.Is(err, ErrUserExists):
            statusCode = http.StatusConflict  // 409
            message = "User already exists"
            
        case errors.Is(err, ErrUnauthorized):
            statusCode = http.StatusUnauthorized  // 401
            message = "Unauthorized"
            
        case errors.Is(err, ErrForbidden):
            statusCode = http.StatusForbidden  // 403
            message = "Forbidden"
            
        default:
            statusCode = http.StatusInternalServerError  // 500
            message = "Internal server error"
            log.Errorf("Unhandled error: %v", err)
        }
        
        w.WriteHeader(statusCode)
        json.NewEncoder(w).Encode(ErrorResponse{
            Error:   fmt.Sprintf("%d", statusCode),
            Message: message,
        })
    }
}

// Uso
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) error {
    id := chi.URLParam(r, "id")
    user, err := h.repo.GetByID(r.Context(), id)
    
    if err != nil {
        return err
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
    return nil
}

http.HandleFunc("/users/{id}", HandleError(h.GetUser))
```

---

## 🔗 gRPC: Codes vs HTTP Status

### Mapeo gRPC ↔ HTTP

```
gRPC Code              HTTP Status
───────────────────────────────────
OK (0)                 200 OK
CANCELLED (1)          408 Request Timeout
UNKNOWN (2)            500 Internal Server Error
INVALID_ARGUMENT (3)   400 Bad Request
DEADLINE_EXCEEDED (4)  504 Gateway Timeout
NOT_FOUND (5)          404 Not Found
ALREADY_EXISTS (6)     409 Conflict
PERMISSION_DENIED (7)  403 Forbidden
RESOURCE_EXHAUSTED (8) 429 Too Many Requests
FAILED_PRECONDITION(9) 400 Bad Request
ABORTED (10)           409 Conflict
OUT_OF_RANGE (11)      400 Bad Request
UNIMPLEMENTED (12)     501 Not Implemented
INTERNAL (13)          500 Internal Server Error
UNAVAILABLE (14)       503 Service Unavailable
DATA_LOSS (15)         500 Internal Server Error
UNAUTHENTICATED (16)   401 Unauthorized
```

### En Código Go

```go
import "google.golang.org/grpc/codes"
import "google.golang.org/grpc/status"

func (s *AuthServer) Register(ctx context.Context, req *pb.RegisterRequest) (*pb.User, error) {
    // Validación
    if req.Email == "" {
        return nil, status.Error(codes.InvalidArgument, "email is required")
    }
    
    user, err := s.repo.Create(ctx, req)
    
    if err != nil {
        if errors.Is(err, ErrUserExists) {
            return nil, status.Error(codes.AlreadyExists, "user already exists")
        }
        
        if errors.Is(err, ErrDatabaseDown) {
            return nil, status.Error(codes.Unavailable, "database unavailable")
        }
        
        return nil, status.Error(codes.Internal, "registration failed")
    }
    
    return user, nil
}
```

---

## 📊 Tabla Rápida de Decisión

```
Pregunta                           Status Code
─────────────────────────────────────────────────
¿Es válido el request?
  No → 400 Bad Request
  Sí → Continuar

¿Necesita autenticación?
  No autenticado → 401 Unauthorized
  Autenticado pero sin permisos → 403 Forbidden
  Sí → Continuar

¿Recurso existe?
  No → 404 Not Found
  Sí → Continuar

¿Hay conflicto (ej: duplicado)?
  Sí → 409 Conflict
  No → Continuar

¿Se procesó exitosamente?
  No → 500 Internal Server Error
  Sí → 200 OK (o 201 Created, 204 No Content)
```

---

## 🎯 Tu Proyecto: Status Codes en Gateway

### Mapeo Actual

```go
// ❌ gateway/internal/handlers/auth.go
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // Todos retornan 200
    user, err := h.authClient.Login(r.Context(), req)
    
    if err != nil {
        // ❌ Debería usar status codes
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]interface{}{
            "success": false,
            "error":   err.Error(),
        })
        return
    }
    
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(user)
}
```

### Mejorado

```go
// ✅ gateway/internal/handlers/auth.go
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{
            "error": "invalid request body",
        })
        return
    }
    
    if req.Email == "" || req.Password == "" {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{
            "error": "email and password are required",
        })
        return
    }
    
    user, err := h.authClient.Login(r.Context(), req)
    
    if err != nil {
        // Mapear gRPC error a HTTP status
        status := status.FromContextError(err)
        
        switch status.Code() {
        case codes.InvalidArgument:
            w.WriteHeader(http.StatusBadRequest)
            
        case codes.Unauthenticated:
            w.WriteHeader(http.StatusUnauthorized)
            
        case codes.Unavailable:
            w.WriteHeader(http.StatusServiceUnavailable)
            
        default:
            w.WriteHeader(http.StatusInternalServerError)
        }
        
        json.NewEncoder(w).Encode(map[string]string{
            "error": status.Message(),
        })
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(user)
}
```

---

## ⚡ Best Practices

✅ **Usa status codes estándar**
✅ **Retorna JSON con error details**
✅ **Mapea correctamente errores internos a códigos**
✅ **Loguea 5xx pero no expongas detalles**
✅ **Usa 4xx para errores del cliente**
✅ **Usa 5xx para errores del servidor**
✅ **Sé consistente en toda la API**

---

## ⚠️ Errores Comunes

❌ Retornar 200 para errores
❌ Usar 400 para todo
❌ No enviar error details
❌ Inconsistencia en diferentes endpoints
❌ Exponer stack traces al cliente

---

## 📚 Recursos

### RFC
- HTTP Semantics (RFC 9110): https://tools.ietf.org/html/rfc9110
- gRPC Status Codes: https://grpc.io/docs/guides/error

### Go
- net/http status constants: https://pkg.go.dev/net/http
- grpc/codes: https://pkg.go.dev/google.golang.org/grpc/codes

---

## 💼 En Entrevistas

**Pregunta:** "¿Qué status code retornarías si un usuario intenta crear una orden pero el inventario está agotado?"

**Respuesta:**
> "Sería 422 Unprocessable Entity. El request es sintácticamente válido (no es 400 Bad Request), pero la lógica de negocio rechaza procesarlo (inventario agotado). Aunque 400 es aceptable, 422 es más específico y comunica 'entiendo el request, pero no puedo procesarlo por razones de negocio'. La respuesta incluiría un error específico: `{\"error\": \"out_of_stock\", \"message\": \"Product X is out of stock\"}`. Importante: no es 500 (error del servidor), es una validación de negocio que el cliente debe manejar. En gRPC sería `codes.FailedPrecondition`."

---

#status-codes #http #rest #grpc #api-design
