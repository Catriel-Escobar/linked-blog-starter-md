---
tags: [race-conditions, concurrency, debugging, data-races, synchronization]
date: 2026-01-05
related: [CSP, Deadlocks, Goroutines, Synchronization, Mutex]
status: reference
---

# Race Conditions

## 📋 ¿Qué es una Race Condition?

Cuando **múltiples goroutines acceden simultáneamente a la misma memoria sin sincronización**, resultando en comportamiento impredecible.

**Analogía:** Dos personas editando el mismo documento:
- Persona A: Lee count=5, agrega 1, escribe count=6
- Persona B: Lee count=5 (no vió cambio A), agrega 1, escribe count=6
- Resultado: count debería ser 7, pero es 6

---

## 🎯 Problema que Resuelve

### Sin Sincronización (Race Condition)

```go
// ❌ Race condition
var count = 0

func increment() {
    count++  // OPERACIÓN NO ATÓMICA
}

func main() {
    for i := 0; i < 1000; i++ {
        go increment()
    }
    
    time.Sleep(1 * time.Second)
    fmt.Println(count)  // Esperado: 1000, Actual: 834 (varía cada ejecución)
}

// Detrás de escenas:
// count++ es en realidad:
// 1. Load count (lee de memoria)
// 2. Add 1 (suma en CPU)
// 3. Store count (escribe a memoria)

// Si dos goroutines hacen esto simultáneamente:
// G1: Load count (5) → Add 1 → Store (6)
// G2: Load count (5) → Add 1 → Store (6)
// Resultado: 6 (perdimos uno)
```

### Con Sincronización (Sin Race Condition)

```go
// ✅ Sin race condition
var count = 0
var mu sync.Mutex

func increment() {
    mu.Lock()
    count++
    mu.Unlock()
}

func main() {
    for i := 0; i < 1000; i++ {
        go increment()
    }
    
    time.Sleep(1 * time.Second)
    fmt.Println(count)  // Siempre: 1000
}

// Garantías:
// - Solo una goroutine en la sección crítica
// - Lectura y escritura se ven mutuamente
// - Sin pérdida de datos
```

---

## 🏗️ Formas Comunes de Race Conditions

### 1. **Unsafe Read-Modify-Write**

```go
// ❌ Race condition
type Counter struct {
    value int
}

func (c *Counter) Increment() {
    c.value++  // Read-modify-write sin sincronización
}

// ✅ Con sincronización
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    c.value++
    c.mu.Unlock()
}
```

### 2. **Unsafe Map Access**

```go
// ❌ Race condition
var userMap = make(map[string]*User)

func getUser(id string) *User {
    return userMap[id]  // Read sin sincronización
}

func setUser(id string, user *User) {
    userMap[id] = user  // Write sin sincronización
}

// Si mientras lees, otro goroutine escribe → crash

// ✅ Con sincronización
var (
    mu      sync.RWMutex
    userMap = make(map[string]*User)
)

func getUser(id string) *User {
    mu.RLock()
    defer mu.RUnlock()
    return userMap[id]
}

func setUser(id string, user *User) {
    mu.Lock()
    defer mu.Unlock()
    userMap[id] = user
}
```

### 3. **Unsafe Struct Field**

```go
// ❌ Race condition
type User struct {
    ID    string
    Email string
    Age   int
}

func (u *User) Update(email string, age int) {
    u.Email = email  // Write
    u.Age = age      // Write
}

func (u *User) GetAge() int {
    return u.Age  // Read
}

// Si reader lee mientras writer modifica → inconsistencia

// ✅ Con sincronización
type User struct {
    mu    sync.RWMutex
    ID    string
    Email string
    Age   int
}

func (u *User) Update(email string, age int) {
    u.mu.Lock()
    defer u.mu.Unlock()
    u.Email = email
    u.Age = age
}

func (u *User) GetAge() int {
    u.mu.RLock()
    defer u.mu.RUnlock()
    return u.Age
}
```

### 4. **Unsafe Slice**

```go
// ❌ Race condition
var items []string

func append(item string) {
    items = append(items, item)  // Realloc no es sincronizado
}

func list() []string {
    return items  // Read
}

// ✅ Con sincronización
var (
    mu    sync.Mutex
    items []string
)

func append(item string) {
    mu.Lock()
    defer mu.Unlock()
    items = append(items, item)
}

func list() []string {
    mu.Lock()
    defer mu.Unlock()
    return slices.Clone(items)
}
```

---

## 🔍 Detectar Race Conditions

### Go Race Detector

```bash
# Compilar con -race
go run -race main.go

# Salida si hay race:
# ==================
# WARNING: DATA RACE
# Write at 0x00c0001a0000 by goroutine 7:
#     main.main.func2()
#         /path/to/main.go:22 +0x44
#
# Previous write at 0x00c0001a0000 by goroutine 6:
#     main.main.func1()
#         /path/to/main.go:18 +0x44
```

### Tests con Race Detector

```go
func TestRaceCondition(t *testing.T) {
    var count = 0
    
    // Ejecutar con: go test -race
    for i := 0; i < 100; i++ {
        go func() {
            count++
        }()
    }
    
    time.Sleep(100 * time.Millisecond)
    
    if count != 100 {
        t.Errorf("Expected 100, got %d", count)
    }
}
```

---

## 💻 Ejemplos en Tu Proyecto

### Auth Service: User Repository

```go
// ❌ Race condition (sin mutex)
type UserRepository struct {
    users map[string]*domain.User
}

func (r *UserRepository) Create(ctx context.Context, user *domain.User) error {
    r.users[user.ID] = user  // Race condition
    return nil
}

func (r *UserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    return r.users[id], nil  // Race condition
}

// ✅ Sin race condition (con mutex)
type UserRepository struct {
    mu    sync.RWMutex
    users map[string]*domain.User
}

func (r *UserRepository) Create(ctx context.Context, user *domain.User) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    r.users[user.ID] = user
    return nil
}

func (r *UserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    return r.users[id], nil
}
```

### Gateway: In-Memory Cache

```go
// ❌ Race condition
type Cache struct {
    data map[string]interface{}
}

func (c *Cache) Get(key string) interface{} {
    return c.data[key]
}

func (c *Cache) Set(key string, value interface{}) {
    c.data[key] = value
}

// ✅ Sin race condition
type Cache struct {
    mu   sync.RWMutex
    data map[string]interface{}
}

func (c *Cache) Get(key string) interface{} {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    return c.data[key]
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    c.data[key] = value
}
```

---

## 📊 Sincronización Primitivas

```
sync.Mutex
├─ Lock()/Unlock()
├─ Uso: Secciones críticas
└─ Performance: Más lento

sync.RWMutex
├─ Lock()/Unlock() (writer)
├─ RLock()/RUnlock() (readers)
├─ Uso: Múltiples readers, pocos writers
└─ Performance: Mejor para reads

sync.Atomic
├─ LoadInt32, StoreInt32, AddInt32
├─ Uso: Variables simples (counters)
└─ Performance: Muy rápido

sync.Once
├─ Do(func())
├─ Uso: Inicializar una sola vez
└─ Garantía: Se ejecuta exactamente una vez

Channels
├─ Enviar/Recibir
├─ Uso: Comunicación entre goroutines
└─ Performance: Mejor para flujo de trabajo
```

---

## ⚡ Best Practices

✅ **Sempre que compartas data entre goroutines, sincroniza**
✅ **Usa `go run -race` en tests**
✅ **RWMutex para múltiples readers**
✅ **Atomic para variables simples**
✅ **Channels para comunicación**
✅ **Evita compartir memoria si es posible (usa CSP)**

---

## ⚠️ Antipatrones

❌ Compartir memoria sin sincronización
❌ Lock granular muy pequeño (contention)
❌ Hold locks por mucho tiempo
❌ Nested locks (riesgo de deadlock)
❌ Ignoring race detector warnings
❌ Asumir que is "probably thread-safe"

---

## 🔗 Go Race Detector Internals

```
Race Detector:
└─ Instrumenta el código
   ├─ Registra accesos a memoria
   ├─ Detecta simultaneous access sin sincronización
   └─ Genera WARNING

Overhead:
├─ Tiempo: ~2-5x más lento
├─ Memoria: ~2-3x más uso
└─ Recomendación: Usar solo en testing/debugging
```

---

## 📚 Recursos

### Go Documentation
- Race Detector: https://go.dev/doc/articles/race_detector
- sync package: https://pkg.go.dev/sync

### Herramientas
- Go Race Detector: Incluido en Go
- ThreadSanitizer: Backend del race detector

### Artículos
- "Introducing the Go Race Detector" - Go Blog
- "Synchronization" - Effective Go

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo evitas race conditions en Go?"

**Respuesta:**
> "Go ofrece excelentes herramientas. Primero, el principio: 'share memory by communicating' - uso channels para comunicación entre goroutines en lugar de compartir memoria. Cuando necesito compartir estado mutable, uso sincronización primitivas: sync.Mutex para secciones críticas, sync.RWMutex si tengo múltiples readers y pocos writers (mejor performance), sync.Atomic para contadores simples. Segundo, testing: Always ejecuto `go test -race` - Go's race detector instrumenta el código, detecta accesos simultáneos sin sincronización. Ejemplo: si dos goroutines escriben en un map sin mutex, race detector lo reporta inmediatamente. En mi auth-service, el user repository accesa maps, envuelvo todo con RWMutex - readers usan RLock (muchas operaciones concurrentes), writers usan Lock (exclusivo). Tercero, design: intento evitar shared state cuando puedo - inmutabilidad o CSP patterns. El race detector es oro: catch bugs que causarían intermittent failures en production."

---

#race-conditions #concurrency #synchronization #debugging #go
