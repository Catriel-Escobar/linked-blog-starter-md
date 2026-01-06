---
tags: [http, networking, protocol, performance, multiplexing]
date: 2026-01-05
related: [gRPC, Protocol Buffers, Streaming APIs, HTTP/3]
status: reference
---

# HTTP/2

## 📋 ¿Qué es HTTP/2?

La **segunda versión del protocolo HTTP**, introducida en 2015, que mejora significativamente la velocidad y eficiencia respecto a HTTP/1.1.

**Analogía:** 
- HTTP/1.1: Una sola caja de correos, un cartero, un paquete a la vez
- HTTP/2: Una sola caja, un cartero, múltiples paquetes simultáneamente

---

## 🎯 Problemas de HTTP/1.1 que Resuelve

### HTTP/1.1: Conexión por Request

```
Request 1: HTML        ════════════════════ 100ms
Request 2: CSS                              ════════════════ 80ms
Request 3: JS                                                ════ 50ms
Request 4: Image                                                   ════════ 120ms
                                                                    
Total: ~350ms (secuencial)
```

```
┌──────────────────────────────────────────┐
│ Cliente                 Servidor        │
├──────────────────────────────────────────┤
│ Abre conexión TCP      ←───────────────→ │
│ Envía GET /index.html  ───────────────→  │
│                        ←─────────────── HTML
│ Cierra conexión                          │
│                                          │
│ Abre nueva conexión    ←───────────────→ │
│ Envía GET /style.css   ───────────────→  │
│                        ←─────────────── CSS
│ Cierra conexión                          │
│                                          │
│ (Repite para cada recurso...)            │
└──────────────────────────────────────────┘
```

**Problemas:**
- ❌ Overhead: Abrir/cerrar conexiones TCP es lento
- ❌ Secuencial: No puedes enviar requests simultáneamente
- ❌ Head-of-line blocking: Si request 1 es lenta, bloquea request 2

### HTTP/2: Multiplexing

```
Request 1 (HTML)   ════════════════════
Request 2 (CSS)         ════════════════
Request 3 (JS)              ════
Request 4 (Image)              ════════

Total: ~120ms (paralelo en la misma conexión)
```

```
┌──────────────────────────────────────────┐
│ Cliente                 Servidor        │
├──────────────────────────────────────────┤
│ Abre conexión TCP      ←───────────────→ │
│ Envía GET /index.html  ───────────────→  │
│ Envía GET /style.css   ───────────────→  │
│ Envía GET /script.js   ───────────────→  │
│ Envía GET /image.png   ───────────────→  │
│                        ←──────────────── HTML
│                        ←──────────────── CSS
│                        ←──────────────── JS
│                        ←──────────────── Image
└──────────────────────────────────────────┘
```

**Mejoras:**
- ✅ Una sola conexión TCP
- ✅ Multiplexing: múltiples requests/responses simultáneamente
- ✅ Sin head-of-line blocking

---

## 🏗️ Características Principales

### 1. **Binary Framing Layer**

HTTP/2 divide los datos en **frames** binarios (vs texto en HTTP/1.1).

```
HTTP/1.1:
GET /api/user HTTP/1.1
Host: example.com
Content-Type: application/json
Content-Length: 15

{"name":"Carlos"}

Texto → más bytes

HTTP/2:
[Frame Type: HEADERS]
[Frame Type: DATA]
[Stream ID: 1]

Binario → más eficiente
```

**Ventajas:**
- ✅ Parsing más rápido (binario vs parsear texto)
- ✅ Tamaño más compacto
- ✅ Más eficiente para máquinas

### 2. **Multiplexing y Streams**

Múltiples streams en una sola conexión TCP.

```
Connection TCP (1)
├── Stream 1: GET /user (Request ID 1)
│   ├── HEADERS frame
│   ├── DATA frame
│   └── ← HEADERS + DATA response
│
├── Stream 2: GET /orders (Request ID 2)
│   ├── HEADERS frame
│   └── ← HEADERS + DATA response (más rápida)
│
└── Stream 3: POST /checkout (Request ID 3)
    ├── HEADERS frame
    ├── DATA frame (body)
    └── ← HEADERS + DATA response
```

**Beneficio:** Todos en paralelo, sin bloqueos.

### 3. **Server Push**

El servidor puede **anticipar** qué recursos necesita el cliente y enviarlos sin esperar.

```
HTTP/1.1:
Client: GET /index.html
Server: ← index.html
Client: (analizando HTML) GET /style.css
Server: ← style.css
Client: GET /script.js
Server: ← script.js

HTTP/2 con Server Push:
Client: GET /index.html
Server: ← index.html
Server: (anticipando) PUSH /style.css
Server: (anticipando) PUSH /script.js
Client: (recibe todo junto)
```

### 4. **Header Compression (HPACK)**

Comprime headers HTTP usando tabla estática/dinámica.

```
Request 1:
GET /user HTTP/2
Accept: application/json
User-Agent: Chrome

Headers ~300 bytes

Request 2 (al mismo servidor):
GET /orders HTTP/2
Accept: application/json
User-Agent: Chrome

HPACK: Solo envía diferencias
Headers ~20 bytes (93% más pequeño)
```

### 5. **Flow Control**

Ambos lados pueden controlar cuántos datos recibir (evitar desbordamientos).

```
Servidor → Cliente: "Puedo recibir hasta 64KB"
Cliente envía 64KB
Cliente: "Esperando confirmación del servidor..."
Servidor: "OK, ahora puedo recibir 64KB más"
Cliente envía 64KB más
```

---

## ⚡ HTTP/1.1 vs HTTP/2

| Feature | HTTP/1.1 | HTTP/2 |
|---------|----------|--------|
| **Conexiones** | Múltiples (6-8) | Una (multiplexing) |
| **Request/Response** | Secuencial | Paralelo |
| **Formato** | Texto | Binario |
| **Headers** | No comprimidos | HPACK comprimido |
| **Server Push** | No | Sí |
| **Latencia** | ~350ms (4 recursos) | ~120ms (4 recursos) |
| **Throughput** | 100 Mbps | 300+ Mbps |

### Medición Real

```
Descargar 100 archivos de 1KB cada uno:

HTTP/1.1 (Keep-Alive):
├── Conexión 1: 50 archivos secuencial = 1000ms
├── Conexión 2: 50 archivos secuencial = 1000ms
└── Total: ~2000ms

HTTP/2:
└── Conexión 1: 100 archivos paralelo = ~200ms
```

---

## 🔧 HTTP/2 en Go

### Cliente HTTP/2

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    // Go 1.6+ usa HTTP/2 por defecto con HTTPS
    client := &http.Client{}
    
    // Request
    resp, err := client.Get("https://example.com/api/user")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
    
    fmt.Printf("Protocol: %s\n", resp.Proto)  // "HTTP/2.0"
    fmt.Printf("Status: %d\n", resp.StatusCode)
}
```

### Servidor HTTP/2

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    // HTTP/2 se habilita automáticamente con TLS
    http.HandleFunc("/api/user", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "HTTP/2 Protocol: %s\n", r.Proto)
        fmt.Fprintf(w, "Remote Address: %s\n", r.RemoteAddr)
    })
    
    // TLS habilitado = HTTP/2 automático
    log.Fatal(http.ListenAndServeTLS(":443", "cert.pem", "key.pem", nil))
}
```

### Verificar HTTP/2

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    client := &http.Client{}
    resp, _ := client.Get("https://google.com")
    
    // HTTP/2 si r.Proto es "HTTP/2.0"
    if resp.Proto == "HTTP/2.0" {
        fmt.Println("✅ HTTP/2 activo")
    } else {
        fmt.Printf("❌ %s\n", resp.Proto)
    }
}
```

---

## 🔗 gRPC + HTTP/2

gRPC usa **HTTP/2 obligatoriamente**, permitiendo:

### 1. **Multiplexing de Streams gRPC**

```protobuf
service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (Order);
    rpc ListOrders(ListOrdersRequest) returns (stream Order);  // Streaming
}
```

```go
// Múltiples llamadas gRPC en paralelo sobre HTTP/2
client := pb.NewOrderServiceClient(conn)

// Request 1
go func() {
    order, _ := client.CreateOrder(ctx, &CreateOrderRequest{...})
    fmt.Println(order)
}()

// Request 2
go func() {
    stream, _ := client.ListOrders(ctx, &ListOrdersRequest{...})
    for {
        order, _ := stream.Recv()
        fmt.Println(order)
    }
}()

// Ambas en paralelo sobre la misma conexión HTTP/2
```

### 2. **Bidirectional Streaming**

```protobuf
service ChatService {
    rpc Chat(stream Message) returns (stream Message);
}
```

```go
// Cliente y servidor intercambian mensajes simultáneamente
// Posible porque HTTP/2 permite multiplexing real
stream, _ := client.Chat(ctx)

go func() {
    for {
        msg, _ := stream.Recv()  // Recibir del servidor
        fmt.Println(msg)
    }
}()

stream.Send(&Message{Text: "Hola"})  // Enviar al servidor
stream.Send(&Message{Text: "¿Cómo estás?"})
```

---

## 📊 HTTP/2 en Microservicios

### Arquitectura con HTTP/2

```
Client (HTTPS/HTTP/2)
    ↓ (multiplexing)
Gateway (HTTP/2)
    ├→ Auth Service (gRPC/HTTP/2) [multiplexing]
    ├→ Order Service (gRPC/HTTP/2) [multiplexing]
    └→ Payment Service (gRPC/HTTP/2) [multiplexing]
        ↓ (todas en paralelo)
    PostgreSQL
```

**Beneficio:** Una sola conexión TCP del gateway a cada servicio, múltiples requests/responses paralelos.

### Comparación

```
HTTP/1.1 approach:
Gateway → Auth Service: Abre conexión para request, cierra
Gateway → Order Service: Abre conexión para request, cierra
Gateway → Payment Service: Abre conexión para request, cierra
Overhead TCP: 3 handshakes + 3 closes

HTTP/2 approach:
Gateway → Auth Service: Una conexión, multiplexing
    ├─ Request 1 (Create order)
    ├─ Request 2 (Verify payment)
    └─ Request 3 (Update inventory)
    (todas paralelo)
Overhead TCP: 1 handshake
```

---

## ⚠️ Consideraciones HTTP/2

### Cuándo Ayuda ✅
- ✅ **Múltiples requests** simultáneamente
- ✅ **Streaming** (bidireccional)
- ✅ **Muchos clientes** (conexión por cliente optimizada)
- ✅ **Latencia importante** (reducir round-trips)

### Cuándo NO Importa ❌
- ❌ **Pocos requests** por conexión
- ❌ **Requests grandes** (el tamaño es lo que importa, no protocol)
- ❌ **Conexiones cortas** (TLS handshake domina)

### Problemas Potenciales ⚠️
- ⚠️ **Server Push puede ser contraproducente** (si cliente cachea)
- ⚠️ **Debugging más difícil** (binario vs texto)
- ⚠️ **TLS obligatorio** (overhead inicial)

---

## 🔗 Tu Proyecto

En tu arquitectura:

```go
// Gateway (HTTPS/HTTP/2)
http.ListenAndServeTLS(":8443", "cert.pem", "key.pem", router)

// Gateway → Auth Service (gRPC/HTTP/2)
conn, _ := grpc.NewClient(
    "auth-service:50051",
    grpc.WithTransportCredentials(...),  // TLS
)
client := pb.NewAuthServiceClient(conn)

// Múltiples llamadas en paralelo
client.Login(ctx, ...)    // Stream 1
client.Logout(ctx, ...)   // Stream 2 (simultáneo)
client.Register(ctx, ...) // Stream 3 (simultáneo)
```

---

## 📚 Recursos

### Documentación
- HTTP/2 RFC 7540: https://tools.ietf.org/html/rfc7540
- Google HTTP/2 Explained: https://http2.github.io
- gRPC Performance Best Practices: https://grpc.io/docs/guides/performance

### Herramientas
- `h2load`: Benchmark HTTP/2
- `nghttp2`: Cliente/servidor HTTP/2
- `curl`: Con flag `--http2`

### Tutoriales
- "HTTP/2 from High Performance Browser Networking" - Ilya Grigorik
- "gRPC and HTTP/2" - Udemy

---

## 💼 En Entrevistas

**Pregunta:** "¿Cuál es la ventaja principal de HTTP/2 sobre HTTP/1.1?"

**Respuesta:**
> "HTTP/2 introduce multiplexing, permitiendo múltiples requests/responses en paralelo sobre una sola conexión TCP, en lugar de la naturaleza secuencial de HTTP/1.1. En HTTP/1.1 necesitabas 6-8 conexiones abiertas para paralelismo. HTTP/2 también comprime headers (HPACK) y usa framing binario en lugar de texto, reduciendo latencia y bytes en la red. En microservicios, esto es crucial: con gRPC (que usa HTTP/2), el gateway puede hacer múltiples llamadas a servicios internos en paralelo sobre la misma conexión, reduciendo latencia y consumo de recursos significativamente."

---

#http2 #networking #performance #protocol #grpc #microservices
