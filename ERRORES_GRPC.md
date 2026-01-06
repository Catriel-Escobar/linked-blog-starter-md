# 📘 Guía de Manejo de Errores en gRPC

## 🎯 Regla de Oro

**En gRPC, los errores se devuelven como `error`, NO como campos en el mensaje de respuesta.**

### ❌ INCORRECTO
```go
func (h *Handler) Register(...) (*Response, error) {
    err := h.service.Register(...)
    if err != nil {
        return &Response{
            Success: false,  // ❌ Error en el mensaje
            Message: err.Error(),
        }, nil  // ❌ No error gRPC
    }
}
```

### ✅ CORRECTO
```go
func (h *Handler) Register(...) (*Response, error) {
    err := h.service.Register(...)
    if err != nil {
        return nil, mapErrorToGRPCStatus(err)  // ✅ Error gRPC real
    }
    
    return &Response{
        Success: true,
        Message: "Success",
    }, nil
}
```

---

## 📊 Códigos gRPC y su Mapeo a HTTP

| Código gRPC | HTTP Status | Cuándo Usar |
|-------------|-------------|-------------|
| `codes.OK` | 200 | Éxito |
| `codes.InvalidArgument` | 400 | Validación fallida, parámetros inválidos |
| `codes.Unauthenticated` | 401 | Credenciales inválidas, token expirado |
| `codes.PermissionDenied` | 403 | Usuario sin permisos |
| `codes.NotFound` | 404 | Recurso no encontrado |
| `codes.AlreadyExists` | 409 | Email ya existe, recurso duplicado |
| `codes.FailedPrecondition` | 412 | Email no verificado, estado inválido |
| `codes.Internal` | 500 | Error interno, bugs, excepciones |
| `codes.Unavailable` | 503 | Servicio no disponible, DB caída |
| `codes.DeadlineExceeded` | 504 | Timeout |

---

## 🔧 Función mapErrorToGRPCStatus()

### **Niveles de Manejo de Errores**

```go
func mapErrorToGRPCStatus(err error) error {
    if err == nil {
        return nil
    }

    // ============== NIVEL 1: Errores de Negocio ==============
    // Errores conocidos y esperados del dominio
    switch err {
    case service.ErrInvalidCredentials:
        return status.Error(codes.Unauthenticated, "Invalid email or password")
    case service.ErrEmailAlreadyExists:
        return status.Error(codes.AlreadyExists, "Email already registered")
    // ... más errores de negocio
    }

    // ============== NIVEL 2: Errores de Infraestructura ==============
    // Errores de base de datos, red, etc.
    errMsg := err.Error()
    
    if strings.Contains(errMsg, "relation") && strings.Contains(errMsg, "does not exist") {
        return status.Error(codes.FailedPrecondition, "Database not initialized")
    }
    
    if strings.Contains(errMsg, "connection refused") {
        return status.Error(codes.Unavailable, "Database connection failed")
    }
    
    // ============== NIVEL 3: Error Genérico ==============
    // Cualquier error no esperado
    return status.Error(codes.Internal, "Internal server error")
}
```

---

## 🎓 Tipos de Errores

### **1. Errores de Cliente (4xx)**

**Culpa del cliente**, puede corregirse cambiando el request.

```go
// 400 Bad Request
codes.InvalidArgument → "Invalid email format"
codes.InvalidArgument → "Password too short"

// 401 Unauthorized
codes.Unauthenticated → "Invalid credentials"
codes.Unauthenticated → "Token expired"

// 403 Forbidden
codes.PermissionDenied → "Admin access required"

// 404 Not Found
codes.NotFound → "User not found"

// 409 Conflict
codes.AlreadyExists → "Email already registered"

// 412 Precondition Failed
codes.FailedPrecondition → "Email not verified"
codes.FailedPrecondition → "Database not initialized"
```

### **2. Errores de Servidor (5xx)**

**Culpa del servidor**, el cliente no puede hacer nada.

```go
// 500 Internal Server Error
codes.Internal → "Unexpected error"
codes.Internal → "Database query error"

// 503 Service Unavailable
codes.Unavailable → "Database connection failed"
codes.Unavailable → "Service temporarily down"

// 504 Gateway Timeout
codes.DeadlineExceeded → "Database operation timed out"
```

---

## 🔍 Ejemplos de Errores de Base de Datos

### **Tabla no existe**
```
Error: pq: relation "users" does not exist
→ codes.FailedPrecondition
→ HTTP 412
→ "Database schema not initialized. Please run migrations."
```

**Causa**: No se ejecutaron las migraciones.

**Solución**: Ejecutar `psql` y crear las tablas.

---

### **Conexión rechazada**
```
Error: dial tcp 127.0.0.1:5432: connect: connection refused
→ codes.Unavailable
→ HTTP 503
→ "Database connection failed"
```

**Causa**: PostgreSQL no está corriendo.

**Solución**: Iniciar Docker con `make up`.

---

### **Constraint violado (duplicate key)**
```
Error: pq: duplicate key value violates unique constraint "users_email_key"
→ codes.AlreadyExists
→ HTTP 409
→ "Record already exists"
```

**Causa**: Email ya registrado.

**Solución**: El cliente debe usar otro email.

---

### **Timeout**
```
Error: context deadline exceeded
→ codes.DeadlineExceeded
→ HTTP 504
→ "Database operation timed out"
```

**Causa**: Query muy lento o DB sobrecargada.

**Solución**: Optimizar query, agregar índices, escalar DB.

---

## 🚀 Flujo de Errores Completo

```
┌─────────────────────────────────────────────────────────┐
│ 1. Error ocurre en Service Layer                        │
│    return service.ErrInvalidCredentials                 │
└────────────────────┬────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 2. Handler gRPC intercepta                              │
│    err := h.authService.Login(...)                      │
│    if err != nil {                                      │
│        return nil, mapErrorToGRPCStatus(err)            │
│    }                                                     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 3. mapErrorToGRPCStatus() traduce a código gRPC        │
│    return status.Error(codes.Unauthenticated, "...")   │
└────────────────────┬────────────────────────────────────┘
                     │ gRPC error enviado por red
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 4. Gateway recibe error gRPC                            │
│    resp, err := h.authClient.Login(ctx, ...)           │
│    if err != nil {                                      │
│        handleGRPCError(w, err)  // Mapea gRPC → HTTP   │
│    }                                                     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 5. handleGRPCError() mapea a HTTP status               │
│    st, _ := status.FromError(err)                      │
│    switch st.Code() {                                   │
│    case codes.Unauthenticated:                          │
│        httpStatus = 401                                 │
│    }                                                     │
│    http.Error(w, st.Message(), httpStatus)             │
└────────────────────┬────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 6. Cliente HTTP recibe respuesta                        │
│    HTTP/1.1 401 Unauthorized                            │
│    {"error": "Invalid email or password", "code": 401}  │
└─────────────────────────────────────────────────────────┘
```

---

## ✅ Checklist de Implementación

- [x] Definir errores de negocio en `service/auth_service.go`
- [x] Usar `return nil, mapErrorToGRPCStatus(err)` en handlers
- [x] Implementar `mapErrorToGRPCStatus()` con 3 niveles
- [x] Mapear códigos gRPC a HTTP en gateway
- [x] Nunca devolver `Success: false` en el mensaje
- [x] Usar `codes.Internal` para errores inesperados
- [x] Mensajes de error user-friendly (no stack traces)

---

## 🐛 Debugging de Errores

### **Ver el error gRPC en el servidor**
```go
// En el handler
log.Printf("Error: %v, Code: %v", err, status.Code(err))
```

### **Ver el error HTTP en el cliente**
```bash
curl -v http://localhost:8080/api/v1/auth/register \
  -d '{"email":"test@test.com"}'

# Respuesta:
# HTTP/1.1 400 Bad Request
# {"error":"Invalid email format","code":400}
```

### **Ver logs de Docker**
```bash
docker logs auth-service -f
docker logs gateway-service -f
```

---

## 📝 Mejores Prácticas

1. ✅ **Siempre** devolver `error` en gRPC, no campos de error
2. ✅ **Mapear** errores de DB a códigos gRPC apropiados
3. ✅ **Mensajes** user-friendly, no técnicos
4. ✅ **Logging** de errores internos para debugging
5. ✅ **Distinguir** entre errores de cliente (4xx) y servidor (5xx)
6. ✅ **Testear** cada tipo de error
7. ❌ **Nunca** exponer detalles internos en mensajes
8. ❌ **Nunca** devolver `codes.OK` con error en el mensaje

---

## 🎯 Resumen

**Antes (❌):**
```go
return &Response{Success: false, Message: err.Error()}, nil
```

**Ahora (✅):**
```go
return nil, mapErrorToGRPCStatus(err)
```

**Beneficios:**
- ✅ Gateway sabe el código HTTP correcto automáticamente
- ✅ Logs estructurados con códigos de error
- ✅ Cliente puede manejar errores programáticamente
- ✅ Métricas precisas (cuántos 4xx vs 5xx)
- ✅ Estándar de la industria (gRPC best practices)
