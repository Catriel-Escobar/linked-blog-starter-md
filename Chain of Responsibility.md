---
tags: [design-patterns, behavioral, middleware, chain, request-handling]
date: 2026-01-05
related: [Decorator Pattern, Command Pattern, Responsibility, Handler]
status: reference
---

# Chain of Responsibility Pattern

## 📋 ¿Qué es Chain of Responsibility?

Un **patrón de diseño comportamental** que permite pasar una solicitud a lo largo de una cadena de manejadores, donde cada manejador decide si procesa la solicitud o la pasa al siguiente en la cadena.

**Analogía:** Un ticket de soporte en una empresa:
1. **Nivel 1 (Support):** ¿Puedo resolverlo? No → Pasar al siguiente
2. **Nivel 2 (Technical):** ¿Puedo resolverlo? No → Pasar al siguiente
3. **Nivel 3 (Manager):** ¿Puedo resolverlo? Sí → Resolver

Cada nivel intenta manejar el ticket, si no puede, lo pasa al siguiente.

---

## 🎯 Problema que Resuelve

### Sin Chain of Responsibility

```go
// Procesar un request de usuario
func ProcessUserRequest(req *UserRequest) error {
    // Validar
    if err := validateRequest(req); err != nil {
        return err
    }
    
    // Autenticar
    user, err := authenticateUser(req)
    if err != nil {
        return err
    }
    
    // Autorizar
    if !authorizeUser(user) {
        return ErrUnauthorized
    }
    
    // Rate limiting
    if !checkRateLimit(user) {
        return ErrRateLimited
    }
    
    // Logging
    logRequest(req)
    
    // Procesar
    return handleUserRequest(req)
}

// ❌ Problemas:
// - Función gigante (God Function)
// - Difícil de mantener (agregar nuevo paso es complicado)
// - Difícil de testear (todos los pasos juntos)
// - No reutilizable (la cadena está hardcodeada)
```

### Con Chain of Responsibility

```go
type RequestHandler interface {
    Handle(req *UserRequest) error
    SetNext(handler RequestHandler) RequestHandler
}

type BaseHandler struct {
    next RequestHandler
}

func (h *BaseHandler) SetNext(handler RequestHandler) RequestHandler {
    h.next = handler
    return handler
}

type ValidationHandler struct {
    BaseHandler
}

func (h *ValidationHandler) Handle(req *UserRequest) error {
    if err := validateRequest(req); err != nil {
        return err
    }
    
    if h.next != nil {
        return h.next.Handle(req)
    }
    return nil
}

type AuthenticationHandler struct {
    BaseHandler
}

func (h *AuthenticationHandler) Handle(req *UserRequest) error {
    user, err := authenticateUser(req)
    if err != nil {
        return err
    }
    
    req.User = user
    
    if h.next != nil {
        return h.next.Handle(req)
    }
    return nil
}

// ✅ Ventajas:
// - Separación de responsabilidades
// - Fácil agregar/remover pasos
// - Reutilizable
// - Testeable
```

---

## 🏗️ Estructura del Pattern

```
Request → Handler1 → Handler2 → Handler3 → Handler4
            ↓ ✗        ↓ ✗        ↓ ✓ (procesa)
          Pass      Pass      Procesa
          next      next        ↓
                             Retorna resultado
```

**Flujo:**
```
1. Handler1 recibe request
2. Si no puede manejar: pasa al siguiente
3. Handler2 recibe request
4. Si no puede manejar: pasa al siguiente
5. Handler3 recibe request
6. Puede manejar: procesa y retorna
```

---

## 💻 Ejemplo 1: Middleware Chain

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

type RequestHandler interface {
    Handle(w http.ResponseWriter, r *http.Request) bool
    SetNext(h RequestHandler) RequestHandler
}

type BaseHandler struct {
    next RequestHandler
}

func (bh *BaseHandler) SetNext(h RequestHandler) RequestHandler {
    bh.next = h
    return h
}

func (bh *BaseHandler) NextHandle(w http.ResponseWriter, r *http.Request) bool {
    if bh.next != nil {
        return bh.next.Handle(w, r)
    }
    return true
}

// Handler 1: Logging
type LoggingHandler struct {
    BaseHandler
}

func (h *LoggingHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    log.Printf("[%s] %s %s", time.Now().Format("15:04:05"), r.Method, r.URL.Path)
    return h.NextHandle(w, r)
}

// Handler 2: Authentication
type AuthenticationHandler struct {
    BaseHandler
}

func (h *AuthenticationHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    token := r.Header.Get("Authorization")
    if token == "" {
        w.WriteHeader(http.StatusUnauthorized)
        w.Write([]byte("Unauthorized"))
        return false
    }
    
    r.Header.Set("X-User-ID", "user-123")
    return h.NextHandle(w, r)
}

// Handler 3: Rate Limiting
type RateLimitHandler struct {
    BaseHandler
    requestCount map[string]int
}

func (h *RateLimitHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    ip := r.RemoteAddr
    h.requestCount[ip]++
    
    if h.requestCount[ip] > 100 {
        w.WriteHeader(http.StatusTooManyRequests)
        w.Write([]byte("Rate limit exceeded"))
        return false
    }
    
    return h.NextHandle(w, r)
}

// Handler 4: Business Logic
type BusinessLogicHandler struct {
    BaseHandler
}

func (h *BusinessLogicHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    userID := r.Header.Get("X-User-ID")
    w.WriteHeader(http.StatusOK)
    fmt.Fprintf(w, "Hello, user %s!", userID)
    return true
}

// Configurar la cadena
func setupChain() RequestHandler {
    logging := &LoggingHandler{}
    auth := &AuthenticationHandler{}
    rateLimit := &RateLimitHandler{requestCount: make(map[string]int)}
    business := &BusinessLogicHandler{}
    
    logging.SetNext(auth).SetNext(rateLimit).SetNext(business)
    
    return logging
}

func main() {
    chain := setupChain()
    
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        chain.Handle(w, r)
    })
    
    http.ListenAndServe(":8080", nil)
}
```

**Flujo para cada request:**
```
GET /api/user (sin token)
    ↓
LoggingHandler: Log "[15:30:45] GET /api/user"
    ↓ return h.NextHandle()
AuthenticationHandler: ❌ Token vacío
    ↓ return false (detiene la cadena)
Response: 401 Unauthorized
```

---

## 💻 Ejemplo 2: Procesamiento de Pedidos

```go
package main

import (
    "fmt"
    "log"
)

type Order struct {
    ID       string
    Items    []string
    Total    float64
    Status   string
}

type OrderHandler interface {
    Handle(order *Order) error
    SetNext(h OrderHandler) OrderHandler
}

type BaseOrderHandler struct {
    next OrderHandler
}

func (boh *BaseOrderHandler) SetNext(h OrderHandler) OrderHandler {
    boh.next = h
    return h
}

func (boh *BaseOrderHandler) NextHandle(order *Order) error {
    if boh.next != nil {
        return boh.next.Handle(order)
    }
    return nil
}

// Handler 1: Validar Inventario
type InventoryHandler struct {
    BaseOrderHandler
}

func (h *InventoryHandler) Handle(order *Order) error {
    for _, item := range order.Items {
        if !hasInStock(item) {
            return fmt.Errorf("item %s out of stock", item)
        }
    }
    
    log.Println("✓ Inventory check passed")
    order.Status = "inventory_checked"
    return h.NextHandle(order)
}

// Handler 2: Validar Pago
type PaymentHandler struct {
    BaseOrderHandler
}

func (h *PaymentHandler) Handle(order *Order) error {
    if !processPayment(order.Total) {
        return fmt.Errorf("payment failed for order %s", order.ID)
    }
    
    log.Println("✓ Payment processed")
    order.Status = "payment_processed"
    return h.NextHandle(order)
}

// Handler 3: Crear Envío
type ShippingHandler struct {
    BaseOrderHandler
}

func (h *ShippingHandler) Handle(order *Order) error {
    trackingNumber := createShipping(order)
    
    log.Printf("✓ Shipping created: %s\n", trackingNumber)
    order.Status = "shipped"
    return h.NextHandle(order)
}

// Handler 4: Notificar Cliente
type NotificationHandler struct {
    BaseOrderHandler
}

func (h *NotificationHandler) Handle(order *Order) error {
    sendNotification(order.ID, "Your order has been processed")
    
    log.Println("✓ Customer notified")
    order.Status = "completed"
    return h.NextHandle(order)
}

// Funciones auxiliares
func hasInStock(item string) bool {
    return item != "out-of-stock-item"
}

func processPayment(amount float64) bool {
    return amount > 0
}

func createShipping(order *Order) string {
    return "TRACKING-" + order.ID
}

func sendNotification(orderID, msg string) {
    fmt.Printf("📧 Notification for %s: %s\n", orderID, msg)
}

// Uso
func main() {
    // Crear cadena
    inventory := &InventoryHandler{}
    payment := &PaymentHandler{}
    shipping := &ShippingHandler{}
    notification := &NotificationHandler{}
    
    inventory.SetNext(payment).SetNext(shipping).SetNext(notification)
    
    // Procesar orden
    order := &Order{
        ID:     "ORD-001",
        Items:  []string{"laptop", "mouse"},
        Total:  1500.00,
    }
    
    if err := inventory.Handle(order); err != nil {
        log.Printf("❌ Error: %v", err)
        return
    }
    
    fmt.Printf("✅ Order %s completed with status: %s\n", order.ID, order.Status)
}
```

**Ejecución:**
```
✓ Inventory check passed
✓ Payment processed
✓ Shipping created: TRACKING-ORD-001
📧 Notification for ORD-001: Your order has been processed
✓ Customer notified
✅ Order ORD-001 completed with status: completed
```

---

## 🎯 Tu Proyecto: Aplicar en Gateway

Podrías aplicar Chain of Responsibility en tus handlers:

```go
// gateway/internal/handlers/chain.go

type AuthRequest struct {
    Email    string
    Password string
    User     *pb.User
}

type RequestHandler interface {
    Handle(req *AuthRequest) error
    SetNext(h RequestHandler) RequestHandler
}

type BaseHandler struct {
    next RequestHandler
}

func (bh *BaseHandler) SetNext(h RequestHandler) RequestHandler {
    bh.next = h
    return h
}

func (bh *BaseHandler) PassToNext(req *AuthRequest) error {
    if bh.next != nil {
        return bh.next.Handle(req)
    }
    return nil
}

// Handler 1: Validar entrada
type ValidationHandler struct {
    BaseHandler
}

func (h *ValidationHandler) Handle(req *AuthRequest) error {
    if req.Email == "" || req.Password == "" {
        return ErrInvalidInput
    }
    return h.PassToNext(req)
}

// Handler 2: Llamar servicio de auth
type AuthServiceHandler struct {
    BaseHandler
    client pb.AuthServiceClient
}

func (h *AuthServiceHandler) Handle(req *AuthRequest) error {
    user, err := h.client.Login(context.Background(), &pb.LoginRequest{
        Email:    req.Email,
        Password: req.Password,
    })
    if err != nil {
        return err
    }
    
    req.User = user
    return h.PassToNext(req)
}

// Handler 3: Logging
type LoggingHandler struct {
    BaseHandler
}

func (h *LoggingHandler) Handle(req *AuthRequest) error {
    log.Printf("Login attempt for %s", req.Email)
    err := h.PassToNext(req)
    log.Printf("Login result: %v", err)
    return err
}
```

---

## ⚡ Ventajas

✅ **Desacoplamiento:** Cada handler no conoce a los demás  
✅ **Flexibilidad:** Fácil agregar/remover handlers  
✅ **Single Responsibility:** Cada handler hace una cosa  
✅ **Reutilizable:** Los mismos handlers en diferentes cadenas  
✅ **Dinámico:** Construir cadenas en runtime  

---

## ⚠️ Desventajas

❌ **Debugging:** Difícil seguir flujo de solicitud  
❌ **Orden importante:** Cambiar orden puede romper lógica  
❌ **No garantizado:** Si nadie maneja, no hay error  
❌ **Performance:** Overhead de múltiples llamadas  

---

## 🔗 Comparación: Chain of Responsibility vs Decorator

| Aspecto | Chain of Responsibility | Decorator |
|---------|------------------------|-----------|
| **Propósito** | Pasar solicitud hasta encontrar manejador | Agregar funcionalidad |
| **Decisión** | Manejador decide si continuar | Siempre continúa |
| **Parada** | Puede detener la cadena | No detiene |
| **Orden** | Importante | Importante |
| **Uso** | Solicitudes, eventos | Comportamiento, funcionalidad |

---

## 📚 Recursos

### Documentación
- Design Patterns: Elements of Reusable Object-Oriented Software
- Refactoring.Guru: Chain of Responsibility

### Go ejemplos
- chi middleware
- HTTP handlers en net/http

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo manejarías múltiples validaciones en una solicitud?"

**Respuesta:**
> "Usaría Chain of Responsibility Pattern. Cada validación (email válido, contraseña fuerte, usuario no existe, etc.) es un handler separado. Formo una cadena donde cada handler intenta procesar la solicitud y, si pasa, la pasa al siguiente. Si alguno falla, detiene la cadena y retorna error. Esto es mejor que hacer todas las validaciones en una función gigante porque: cada handler es pequeño y testeable, puedo cambiar el orden dinámicamente, y puedo reutilizar handlers en diferentes contextos. Por ejemplo, la validación de email podría usarse en registration, login y password recovery sin modificar el código."

---

#chain-of-responsibility #design-patterns #behavioral #middleware #request-handling
