---
tags: [grpc, serialization, protobuf, proto3, messaging]
date: 2026-01-05
related: [gRPC, HTTP/2, Streaming APIs, Serialization]
status: reference
---

# Protocol Buffers

## 📋 ¿Qué son Protocol Buffers?

Un **formato de serialización de datos** desarrollado por Google que es:
- 🚀 Más rápido que JSON/XML
- 📦 Más compacto (menor tamaño en bytes)
- 🔐 Type-safe (esquema definido)
- 🔄 Versionable (compatible hacia atrás/adelante)

**Analogía:** Es como JSON, pero compilado y optimizado:

```
JSON:
{"name": "Carlos", "age": 25, "email": "carlos@example.com"}
48 bytes

Protocol Buffers:
[binary data]
15 bytes (68% más pequeño)
```

---

## 🎯 Problema que Resuelve

### Sin Protocol Buffers (JSON)

```go
// Go Service
type User struct {
    Name  string `json:"name"`
    Age   int    `json:"age"`
    Email string `json:"email"`
}

// Serializar
user := User{Name: "Carlos", Age: 25, Email: "carlos@example.com"}
data, _ := json.Marshal(user)
// data = {"name":"Carlos","age":25,"email":"carlos@example.com"} (48 bytes)

// Enviar por red
send(data)

// Python Service recibe
import json
data = receive()
user = json.loads(data)
print(user["name"])  # Sin validación de tipo

// ❌ Problemas:
// - Si cambias estructura: rompes servicios en otros lenguajes
// - JSON es texto (48 bytes vs 15 con protobuf)
// - Sin schema compartido
// - Errores en runtime si tipo es incorrecto
```

### Con Protocol Buffers

```protobuf
// user.proto (compartido entre servicios)
syntax = "proto3";

message User {
    string name = 1;
    int32 age = 2;
    string email = 3;
}
```

```bash
# Compilar para Go
protoc --go_out=. user.proto
# Genera: user.pb.go

# Compilar para Python
protoc --python_out=. user.proto
# Genera: user_pb2.py
```

```go
// Go Service
user := &User{Name: "Carlos", Age: 25, Email: "carlos@example.com"}
data, _ := proto.Marshal(user)
// data = [binary] (15 bytes)

send(data)
```

```python
# Python Service recibe
data = receive()
user = User()
user.ParseFromString(data)
print(user.name)  # Type-safe, validado

# ✅ Ventajas:
# - Mismo schema en todos los lenguajes
# - 68% más pequeño
# - Type-safe automático
# - Compatible con cambios versionados
# - Generación de código automática
```

---

## 🏗️ Sintaxis Proto3

### Tipos Básicos

```protobuf
syntax = "proto3";

package auth.v1;

message User {
    // Tipos primitivos
    string id = 1;           // texto
    string email = 2;
    int32 age = 3;           // entero 32-bit
    int64 user_id = 4;       // entero 64-bit
    bool active = 5;         // boolean
    double score = 6;        // número decimal
    bytes data = 7;          // datos binarios
    
    // El número es el ID del campo (inmutable)
    // ↑ IMPORTANTE: no cambiar estos números
}
```

### Enums

```protobuf
message User {
    enum Status {
        STATUS_UNSPECIFIED = 0;  // Siempre empezar en 0
        ACTIVE = 1;
        INACTIVE = 2;
        BANNED = 3;
    }
    
    string id = 1;
    Status status = 2;  // Usar enum
}

// Uso en Go
user := &User{
    Id:     "user-123",
    Status: User_ACTIVE,
}
```

### Mensajes Anidados

```protobuf
message User {
    message Profile {
        string bio = 1;
        string avatar_url = 2;
    }
    
    string id = 1;
    Profile profile = 2;  // Anidado
}

// Uso
user := &User{
    Id: "user-123",
    Profile: &User_Profile{
        Bio: "Software developer",
    },
}
```

### Repetidos (Listas)

```protobuf
message User {
    string id = 1;
    repeated string emails = 2;  // Lista de strings
    repeated Address addresses = 3;  // Lista de mensajes
}

message Address {
    string street = 1;
    string city = 2;
}

// Uso
user := &User{
    Id: "user-123",
    Emails: []string{
        "carlos@example.com",
        "carlos.dev@example.com",
    },
    Addresses: []*Address{
        {Street: "123 Main St", City: "NYC"},
        {Street: "456 Oak Ave", City: "LA"},
    },
}
```

### Maps

```protobuf
message User {
    string id = 1;
    map<string, string> metadata = 2;  // Key-value
}

// Uso
user := &User{
    Id: "user-123",
    Metadata: map[string]string{
        "company": "Google",
        "title": "Engineer",
    },
}
```

---

## 📝 Ejemplo Completo: Auth Service

### auth.proto

```protobuf
syntax = "proto3";

package auth.v1;

option go_package = "github.com/myapp/auth-service/auth/v1;authv1";

// Enums
enum ErrorCode {
    ERROR_UNSPECIFIED = 0;
    INVALID_CREDENTIALS = 1;
    USER_NOT_FOUND = 2;
    USER_ALREADY_EXISTS = 3;
}

// Mensajes de Request/Response
message RegisterRequest {
    string email = 1;
    string password = 2;
    string full_name = 3;
}

message RegisterResponse {
    string user_id = 1;
    string email = 2;
    int64 created_at = 3;
}

message LoginRequest {
    string email = 1;
    string password = 2;
}

message LoginResponse {
    string access_token = 1;
    string refresh_token = 2;
    User user = 3;
}

// Modelos de Dominio
message User {
    enum Status {
        STATUS_UNSPECIFIED = 0;
        ACTIVE = 1;
        INACTIVE = 2;
        BANNED = 3;
    }
    
    string id = 1;
    string email = 2;
    string full_name = 3;
    Status status = 4;
    int64 created_at = 5;
    int64 updated_at = 6;
}

message Error {
    ErrorCode code = 1;
    string message = 2;
}

// Servicio gRPC
service AuthService {
    rpc Register(RegisterRequest) returns (RegisterResponse);
    rpc Login(LoginRequest) returns (LoginResponse);
    rpc GetUser(GetUserRequest) returns (User);
}

message GetUserRequest {
    string user_id = 1;
}
```

### Compilar

```bash
# En el directorio del proyecto
protoc --go_out=. --go-grpc_out=. ./auth.proto

# Genera:
# - auth.pb.go (estructuras de datos)
# - auth_grpc.pb.go (código del cliente/servidor gRPC)
```

### Usar en Go

```go
// Crear mensaje
user := &authv1.User{
    Id:       "user-123",
    Email:    "carlos@example.com",
    FullName: "Carlos Rodríguez",
    Status:   authv1.User_ACTIVE,
    CreatedAt: time.Now().Unix(),
}

// Serializar (enviar por red)
data, err := proto.Marshal(user)
if err != nil {
    log.Fatal(err)
}

// Deserializar (recibir de red)
receivedUser := &authv1.User{}
err = proto.Unmarshal(data, receivedUser)
if err != nil {
    log.Fatal(err)
}

fmt.Println(receivedUser.Email)  // "carlos@example.com"
```

---

## 🔄 Versionado y Compatibilidad

### Cambios Seguros ✅

```protobuf
// v1.0
message User {
    string id = 1;
    string email = 2;
}

// v1.1 - SEGURO: agregar campo
message User {
    string id = 1;
    string email = 2;
    string phone = 3;  // Nuevo campo
}
```

**Por qué es seguro:**
- Clientes v1.0 ignoran `phone` (campo 3)
- Servidores v1.1 usan default si falta `phone`
- **Compatible hacia atrás y adelante**

### Cambios Peligrosos ❌

```protobuf
// v1.0
message User {
    string id = 1;
    string email = 2;
}

// ❌ ROMPE: Cambiar número de campo
message User {
    string id = 1;
    string phone = 2;  // Era email! Ahora está en campo 2
    string email = 3;
}

// ❌ ROMPE: Cambiar tipo
message User {
    string id = 1;
    int32 email = 2;  // Era string!
}

// ❌ ROMPE: Eliminar campo (sin marcar deprecated)
message User {
    string id = 1;
    // email = 2; desapareció
}
```

### Evolución Correcta

```protobuf
// v1.0
message User {
    string id = 1;
    string email = 2;
}

// v2.0 - Cambios mayores
message User {
    string id = 1;
    string email = 2;
    reserved 3;  // Reservar números no usados
    string phone = 4;
    
    // Marcar deprecated
    string username = 5 [deprecated=true];
}
```

---

## ⚡ Ventajas vs JSON

| Aspecto | JSON | Protobuf |
|--------|------|----------|
| **Tamaño** | 48 bytes | 15 bytes (-68%) |
| **Velocidad** | ~1ms (parse) | ~0.1ms |
| **Schema** | Ninguno | Definido (type-safe) |
| **Legibilidad** | Legible (texto) | Binario |
| **Versionado** | Manual | Automático |
| **Lenguajes** | Cualquiera | Go, Java, Python, etc. |
| **Cambios** | Frecuentes rompen | Versionable |
| **Debugging** | Fácil (texto) | Difícil (binario) |

---

## 💻 gRPC + Protocol Buffers

```protobuf
// api.proto
syntax = "proto3";

service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (Order);
    rpc GetOrder(GetOrderRequest) returns (Order);
    rpc ListOrders(ListOrdersRequest) returns (stream Order);  // Streaming
}

message CreateOrderRequest {
    string customer_id = 1;
    repeated Item items = 2;
}

message Item {
    string product_id = 1;
    int32 quantity = 2;
}

message Order {
    string id = 1;
    repeated Item items = 2;
    double total = 3;
    int64 created_at = 4;
}

message GetOrderRequest {
    string order_id = 1;
}

message ListOrdersRequest {
    string customer_id = 1;
}
```

```bash
protoc --go_out=. --go-grpc_out=. api.proto
```

```go
// Implementar servidor
type OrderServer struct{}

func (s *OrderServer) CreateOrder(ctx context.Context, req *CreateOrderRequest) (*Order, error) {
    // Crear orden
    order := &Order{
        Id: uuid.New().String(),
        Items: req.Items,
        Total: calculateTotal(req.Items),
        CreatedAt: time.Now().Unix(),
    }
    return order, nil
}

func (s *OrderServer) ListOrders(req *ListOrdersRequest, stream grpc.ServerStream) error {
    // Obtener órdenes del cliente
    orders := getOrdersByCustomer(req.CustomerId)
    
    for _, order := range orders {
        if err := stream.Send(order); err != nil {
            return err
        }
    }
    return nil
}
```

---

## 🔗 Tu Proyecto: auth.proto

Ya tienes:

```protobuf
syntax = "proto3";

package auth;

service AuthService {
    rpc Register(RegisterRequest) returns (RegisterResponse);
    rpc VerifyEmail(VerifyEmailRequest) returns (VerifyEmailResponse);
    rpc Login(LoginRequest) returns (LoginResponse);
}
```

**Mejoras recomendadas:**

```protobuf
syntax = "proto3";

package auth.v1;

option go_package = "github.com/myapp/auth-service/auth/v1;authv1";

enum ErrorCode {
    ERROR_UNSPECIFIED = 0;
    INVALID_CREDENTIALS = 1;
    USER_NOT_FOUND = 2;
    USER_ALREADY_EXISTS = 3;
    EMAIL_NOT_VERIFIED = 4;
    TOKEN_EXPIRED = 5;
}

message User {
    string id = 1;
    string email = 2;
    bool email_verified = 3;
    int64 created_at = 4;
}

message RegisterRequest {
    string email = 1;
    string password = 2;
}

message RegisterResponse {
    User user = 1;
}

message LoginRequest {
    string email = 1;
    string password = 2;
}

message LoginResponse {
    string access_token = 1;
    string refresh_token = 2;
    User user = 3;
}

service AuthService {
    rpc Register(RegisterRequest) returns (RegisterResponse);
    rpc Login(LoginRequest) returns (LoginResponse);
}
```

---

## 📚 Recursos

### Documentación
- Protocol Buffers Official: https://protobuf.dev
- Proto3 Language Guide: https://protobuf.dev/programming-guides/proto3
- gRPC + Protobuf: https://grpc.io/docs/languages/go

### Herramientas
- `protoc`: Compilador Protocol Buffers
- `protoc-gen-go`: Plugin para generar código Go
- `buf`: Herramienta moderna para gestionar protobuf

### Tutoriales
- "Protocol Buffers Basics" - Google Developers
- "gRPC in 100 Seconds" - Fireship (YouTube)

---

## 💼 En Entrevistas

**Pregunta:** "¿Por qué usas Protocol Buffers en lugar de JSON?"

**Respuesta:**
> "Protocol Buffers es ideal para comunicación entre microservicios porque es 3-10x más rápido que JSON, genera 50-70% menos bytes en la red, y proporciona type-safety automática con schema compartido entre servicios en diferentes lenguajes. En mi proyecto, uso Protobuf con gRPC entre el gateway (que recibe JSON) y los servicios internos (que comunican con protobuf), logrando mejor performance y validación automática de tipos. Además, protobuf maneja versionado elegantemente: si agrego campos nuevos con números de campo nuevos, clientes viejos ignoran esos campos sin romper, lo que es crítico en microservicios con despliegues independientes."

---

#protobuf #grpc #serialization #schema #microservices #performance
