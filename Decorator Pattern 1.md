---
tags: [design-patterns, oop, structural, middleware, composition]
date: 2026-01-05
related: [Chain of Responsibility, Wrapper Pattern, Middleware, Composition]
status: reference
---

# Decorator Pattern

## 📋 ¿Qué es el Decorator Pattern?

Un **patrón de diseño estructural** que permite agregar responsabilidades (comportamiento) a objetos dinámicamente, envolviéndolos en nuevos objetos que mantienen la misma interfaz.

**Analogía:** Un café:
- Café base: $2
- Agregar leche: +$0.50
- Agregar caramelo: +$0.50
- Agregar crema: +$0.50

Cada "decorator" agrega responsabilidad sin cambiar el café original.

---

## 🎯 Problema que Resuelve

### Sin Decorator (Explosión de Clases)

```go
// Tienes un componente: Component
type Component interface {
    Operation() string
}

// Implementación base
type ConcreteComponent struct{}

func (c *ConcreteComponent) Operation() string {
    return "Base Operation"
}

// Ahora necesitas agregar funcionalidades:
// - Logging
// - Caching
// - Validación
// - Timeout
// - Retry
// - Encryption

// ❌ Solución sin Decorator: Crear una clase por cada combinación
type LoggingComponent struct{ c Component }
type CachingComponent struct{ c Component }
type LoggingCachingComponent struct{ c Component }  // Combinación
type TimeoutLoggingComponent struct{ c Component }  // Otra combinación
type TimeoutCachingComponent struct{ c Component }  // Otra más
type TimeoutLoggingCachingComponent struct{ c Component }  // Explosión!

// Si tienes 5 comportamientos = 2^5 = 32 clases diferentes
```

### Con Decorator (Composición Flexible)

```go
type Component interface {
    Operation() string
}

type ConcreteComponent struct{}

func (c *ConcreteComponent) Operation() string {
    return "Base"
}

// Decoradores reutilizables
type LoggingDecorator struct {
    component Component
}

func (ld *LoggingDecorator) Operation() string {
    log.Println("Before operation")
    result := ld.component.Operation()
    log.Println("After operation")
    return result
}

type CachingDecorator struct {
    component Component
    cache     map[string]string
}

func (cd *CachingDecorator) Operation() string {
    if val, ok := cd.cache["op"]; ok {
        return val
    }
    result := cd.component.Operation()
    cd.cache["op"] = result
    return result
}

// ✅ Combinar dinámicamente
base := &ConcreteComponent{}
withLogging := &LoggingDecorator{base}
withLoggingAndCaching := &CachingDecorator{withLogging, make(map[string]string)}

result := withLoggingAndCaching.Operation()
// ✅ Reutilizable, sin explosión de clases
```

---

## 🏗️ Estructura del Pattern

```
┌────────────────────────────────────┐
│          Component                 │
│   (interfaz/contrato)              │
└────────────────────────────────────┘
         ↑                    ↑
         │                    │
    ┌────┴────┐          ┌────┴────────┐
    │ Concrete│          │  Decorator  │
    │Component│          │             │
    └─────────┘          └────┬────────┘
                               ↑
                    ┌──────────┴──────────┐
                    │                     │
              ┌─────┴──────┐       ┌─────┴──────┐
              │ Concrete   │       │ Concrete   │
              │Decorator A │       │Decorator B │
              └────────────┘       └────────────┘
```

**Clave:** Todos implementan la misma interfaz `Component`, permitiendo composición.

---

## 💻 Ejemplo 1: Handler HTTP con Decoradores

```go
package main

import (
    "log"
    "net/http"
    "time"
)

// Interfaz del componente
type Handler interface {
    Handle(w http.ResponseWriter, r *http.Request)
}

// Implementación base
type BasicHandler struct{}

func (h *BasicHandler) Handle(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("Hello, World!"))
}

// Decorator 1: Logging
type LoggingDecorator struct {
    handler Handler
}

func (ld *LoggingDecorator) Handle(w http.ResponseWriter, r *http.Request) {
    log.Printf("[%s] %s %s", time.Now().Format("15:04:05"), r.Method, r.URL.Path)
    ld.handler.Handle(w, r)
    log.Printf("[%s] Response sent", time.Now().Format("15:04:05"))
}

// Decorator 2: Authentication
type AuthDecorator struct {
    handler Handler
}

func (ad *AuthDecorator) Handle(w http.ResponseWriter, r *http.Request) {
    token := r.Header.Get("Authorization")
    if token == "" {
        w.WriteHeader(http.StatusUnauthorized)
        w.Write([]byte("Unauthorized"))
        return
    }
    ad.handler.Handle(w, r)
}

// Decorator 3: Timing
type TimingDecorator struct {
    handler Handler
}

func (td *TimingDecorator) Handle(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    td.handler.Handle(w, r)
    duration := time.Since(start)
    log.Printf("Request took %v", duration)
}

// Uso
func main() {
    base := &BasicHandler{}
    
    // Combinar decoradores en diferentes órdenes
    handler1 := &LoggingDecorator{
        handler: &AuthDecorator{
            handler: &TimingDecorator{
                handler: base,
            },
        },
    }
    
    http.HandleFunc("/", handler1.Handle)
    http.ListenAndServe(":8080", nil)
}
```

**Flujo de ejecución:**
```
GET /api/user (con token)
    ↓
LoggingDecorator: Log "[15:30:45] GET /api/user"
    ↓
AuthDecorator: Verifica token ✅
    ↓
TimingDecorator: Inicia contador
    ↓
BasicHandler: Ejecuta lógica
    ↓
TimingDecorator: Log "Request took 50ms"
    ↓
AuthDecorator: (sin acción adicional)
    ↓
LoggingDecorator: Log "[15:30:45] Response sent"
```

---

## 💻 Ejemplo 2: Data Processing Pipeline

```go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
)

// Interfaz para procesadores de datos
type DataProcessor interface {
    Process(data string) string
}

// Procesador base: parsea JSON
type JSONParser struct{}

func (jp *JSONParser) Process(data string) string {
    var result map[string]interface{}
    json.Unmarshal([]byte(data), &result)
    return fmt.Sprintf("%v", result)
}

// Decorator: Trim whitespace
type TrimDecorator struct {
    processor DataProcessor
}

func (td *TrimDecorator) Process(data string) string {
    trimmed := strings.TrimSpace(data)
    return td.processor.Process(trimmed)
}

// Decorator: Convert to uppercase
type UppercaseDecorator struct {
    processor DataProcessor
}

func (ud *UppercaseDecorator) Process(data string) string {
    upper := strings.ToUpper(data)
    return ud.processor.Process(upper)
}

// Decorator: Logging
type LoggingDecorator struct {
    processor DataProcessor
}

func (ld *LoggingDecorator) Process(data string) string {
    fmt.Printf("Processing: %s\n", data)
    result := ld.processor.Process(data)
    fmt.Printf("Result: %s\n", result)
    return result
}

// Uso
func main() {
    input := "  {\"name\": \"carlos\"} "
    
    // Pipeline: Trim → Log → Parse
    processor := &LoggingDecorator{
        processor: &TrimDecorator{
            processor: &JSONParser{},
        },
    }
    
    result := processor.Process(input)
    fmt.Println(result)
}
```

---

## 🎯 Tu Proyecto: Decoradores en Handler

Ya tienes un ejemplo perfecto en tu código - **timeoutWriter**:

```go
// gateway/internal/http/middleware/timeout.go

type timeoutWriter struct {
    w           http.ResponseWriter
    wroteHeader bool
    mu          sync.Mutex
}

func (tw *timeoutWriter) Write(b []byte) (int, error) {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    
    if tw.wroteHeader {
        return tw.w.Write(b)
    }
    return len(b), nil
}

// timeoutWriter es un DECORATOR de http.ResponseWriter
// Agrega funcionalidad (mutex protection) sin modificar el original
```

### Expandir el Pattern

```go
// Aplicar decoradores a handlers HTTP
type Handler func(http.ResponseWriter, *http.Request)

// Decorator: Timeout
func WithTimeout(duration time.Duration) func(Handler) Handler {
    return func(next Handler) Handler {
        return func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), duration)
            defer cancel()
            
            r = r.WithContext(ctx)
            next(w, r)
        }
    }
}

// Decorator: Logging
func WithLogging(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Printf("[%s] %s %s", time.Now(), r.Method, r.URL.Path)
        next(w, r)
    }
}

// Decorator: Auth
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

// Uso: Componer decoradores
handler := WithLogging(
    WithAuth(
        WithTimeout(500*time.Millisecond)(
            func(w http.ResponseWriter, r *http.Request) {
                w.Write([]byte("Hello"))
            },
        ),
    ),
)
```

---

## ⚡ Ventajas del Decorator Pattern

✅ **Flexibilidad:** Combina comportamientos dinámicamente  
✅ **Single Responsibility:** Cada decorator tiene una responsabilidad  
✅ **Open/Closed Principle:** Abierto a extensión, cerrado a modificación  
✅ **Reutilizable:** Los mismos decoradores funcionan con diferentes componentes  
✅ **Composición sobre Herencia:** Evita jerarquías complejas  

---

## ⚠️ Desventajas

❌ **Orden importa:** `Auth → Timeout` es diferente a `Timeout → Auth`  
❌ **Debugging:** Stack de decoradores dificulta el debugging  
❌ **Performance:** Cada decorator agrega overhead (aunque mínimo)  

---

## 🔗 Decorator vs Otras Alternativas

| Pattern | Uso | Pros | Contras |
|---------|-----|------|---------|
| **Decorator** | Agregar funcionalidad dinámicamente | Flexible, composición | Orden importa |
| **Inheritance** | Extender clase | Simple | Explosión de clases |
| **Strategy** | Cambiar algoritmo | Flexible | Requiere inyección |
| **Proxy** | Controlar acceso | Control fino | Más overhead |

---

## 📚 Recursos

### Go específico
- `chi/middleware`: Decoradores de middleware
- Patrón de composición en Go: https://golang.org/doc/effective_go#embedding

### Tutoriales
- "Decorator Pattern in Go" - Go Design Patterns
- "Middleware and Decorators" - YouTube

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo agregarías múltiples comportamientos a una función sin modificar el código original?"

**Respuesta:**
> "Usaría el Decorator Pattern. En Go, puedo crear una interfaz `Handler` y luego hacer decoradores que envuelven otros handlers, cada uno agregando responsabilidad. Por ejemplo, si tengo un handler básico y necesito agregar logging, autenticación y timeout, creo tres decoradores diferentes que implementan la misma interfaz. Luego los compongo: `WithLogging(WithAuth(WithTimeout(baseHandler)))`. Esto permite agregar funcionalidad sin modificar el código original, sigue single responsibility (cada decorator hace una cosa), y es reutilizable: los mismos decoradores funcionan con cualquier handler. Es mucho mejor que crear clases como `LoggingAuthTimeoutHandler` para cada combinación."

---

#decorator-pattern #design-patterns #middleware #composition #golang
