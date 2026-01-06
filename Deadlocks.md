---
tags: [deadlocks, concurrency, debugging, synchronization, circular-wait]
date: 2026-01-05
related: [Race Conditions, CSP, Mutex, Context Cancellation, Channels]
status: reference
---

# Deadlocks

## 📋 ¿Qué es un Deadlock?

Cuando **múltiples goroutines se bloquean esperándose mutuamente**, ninguna puede avanzar.

**Analogía:** Intersección de tráfico:
- Coche A espera a que Coche B se mueva
- Coche B espera a que Coche C se mueva
- Coche C espera a que Coche A se mueva
- Resultado: Ninguno se mueve (deadlock) 🚗➡️🚗➡️🚗➡️

---

## 🎯 Condiciones para Deadlock

**Cuatro condiciones DEBEN cumplirse simultáneamente:**

1. **Mutual Exclusion** - Recurso no compartible (mutex)
2. **Hold and Wait** - Goroutine mantiene recurso while esperando otro
3. **No Preemption** - Recurso no puede ser forzadamente tomado
4. **Circular Wait** - Cadena circular de goroutines esperándose

**Para prevenir deadlock, rompe ANY una de estas condiciones.**

---

## 💥 Ejemplos de Deadlock

### 1. **Nested Locks (Circular Wait)**

```go
// ❌ DEADLOCK
var mu1 sync.Mutex
var mu2 sync.Mutex

func goroutine1() {
    mu1.Lock()
    defer mu1.Unlock()
    
    time.Sleep(10 * time.Millisecond)
    
    mu2.Lock()  // Espera mu2
    defer mu2.Unlock()
    
    fmt.Println("G1 done")
}

func goroutine2() {
    mu2.Lock()
    defer mu2.Unlock()
    
    time.Sleep(10 * time.Millisecond)
    
    mu1.Lock()  // Espera mu1
    defer mu1.Unlock()
    
    fmt.Println("G2 done")
}

func main() {
    go goroutine1()
    go goroutine2()
    
    time.Sleep(1 * time.Second)
    // DEADLOCK: G1 espera mu2 (G2 tiene), G2 espera mu1 (G1 tiene)
}

// Sequencia fatal:
// G1: Lock(mu1) ✓
// G2: Lock(mu2) ✓
// G1: Lock(mu2) ✗ (esperando)
// G2: Lock(mu1) ✗ (esperando)
// DEADLOCK!
```

### 2. **Self-Lock**

```go
// ❌ DEADLOCK (simple)
type SafeCounter struct {
    mu    sync.Mutex
    value int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    c.value++
    c.IncrementTwice()  // Llama función que también lockea
}

func (c *SafeCounter) IncrementTwice() {
    c.mu.Lock()  // ❌ DEADLOCK: Ya tengo el lock
    defer c.mu.Unlock()
    
    c.value++
}

// ✅ Solución: RWMutex no-reentrant
// Usa sync.RWMutex con cuidado, o refactoriza
```

### 3. **Channel Deadlock**

```go
// ❌ DEADLOCK
func main() {
    ch := make(chan int)  // Unbuffered
    
    ch <- 1  // Enviar sin receiver
    
    value := <-ch  // Esperar
}

// ch <- 1 se bloquea esperando receiver
// Pero receiver está después (nunca se ejecuta)
// DEADLOCK!

// ✅ Soluciones:

// 1. Buffered channel
ch := make(chan int, 1)
ch <- 1
value := <-ch

// 2. Goroutine separada
ch := make(chan int)
go func() {
    ch <- 1
}()
value := <-ch

// 3. Timeout con select
select {
case ch <- 1:
case <-time.After(1*time.Second):
    fmt.Println("Timeout")
}
```

### 4. **Channel Send en Closed Channel**

```go
// ❌ PANIC (no deadlock, pero similar)
ch := make(chan int)
close(ch)

ch <- 1  // ❌ PANIC: send on closed channel

// ✅ Verificar antes
select {
case ch <- 1:
case <-time.After(1*time.Second):
    fmt.Println("Channel closed or full")
}
```

---

## 🏗️ Patrones de Deadlock

### Patrón 1: Resource Ordering

```go
// ❌ Puede causar deadlock
type Account struct {
    mu      sync.Mutex
    balance int
}

func transfer(from, to *Account, amount int) {
    from.mu.Lock()
    defer from.mu.Unlock()
    
    to.mu.Lock()
    defer to.mu.Unlock()
    
    from.balance -= amount
    to.balance += amount
}

// Si dos goroutines hacen: transfer(A, B) y transfer(B, A)
// DEADLOCK!

// ✅ Solución: Order recursos
func transfer(from, to *Account, amount int) {
    // Siempre lockea en orden (by ID)
    first, second := from, to
    if from.ID > to.ID {
        first, second = to, from
    }
    
    first.mu.Lock()
    defer first.mu.Unlock()
    
    second.mu.Lock()
    defer second.mu.Unlock()
    
    from.balance -= amount
    to.balance += amount
}
```

### Patrón 2: Timeout Prevention

```go
// ✅ Usa timeout para prevenir deadlock permanente
func withTimeout(mu *sync.Mutex, duration time.Duration, fn func()) bool {
    done := make(chan bool, 1)
    
    go func() {
        mu.Lock()
        defer mu.Unlock()
        fn()
        done <- true
    }()
    
    select {
    case <-done:
        return true
    case <-time.After(duration):
        return false  // Timeout
    }
}
```

### Patrón 3: Channels con Timeout

```go
// ✅ Prevenir deadlock en channel operations
func safeSend(ch chan int, value int, timeout time.Duration) bool {
    select {
    case ch <- value:
        return true
    case <-time.After(timeout):
        fmt.Println("Send timeout")
        return false
    }
}

func safeReceive(ch chan int, timeout time.Duration) (int, bool) {
    select {
    case value := <-ch:
        return value, true
    case <-time.After(timeout):
        fmt.Println("Receive timeout")
        return 0, false
    }
}
```

---

## 💻 Detectar Deadlocks

### 1. **Visual Inspection**

```go
// Revisar código buscando:
// - Nested locks en orden diferente
// - Channel send/receive sin timeout
// - Circular dependencies

// Herramienta: Go's deadlock detector en tests
```

### 2. **Testing**

```go
func TestDeadlock(t *testing.T) {
    done := make(chan bool, 1)
    
    go func() {
        myFunction()
        done <- true
    }()
    
    select {
    case <-done:
        // OK
    case <-time.After(5 * time.Second):
        t.Fatal("Deadlock detected")
    }
}
```

### 3. **Runtime Detection**

```go
// Go's scheduler detecta deadlocks si TODOS goroutines están bloqueadas
// Panica: "fatal error: all goroutines are asleep - deadlock!"

// Pero: Si main está en timeout esperando, Go no lo detecta
```

---

## 🎯 Tu Proyecto: Deadlock Prevention

### Auth Service: Transactional Operations

```go
// ❌ Puede causar deadlock con nested locks
type UserRepository struct {
    mu sync.Mutex
}

func (r *UserRepository) CreateWithToken(user *User, token *RefreshToken) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    // Crear usuario
    if err := r.createUser(user); err != nil {
        return err
    }
    
    // Crear token (también intenta lock)
    return r.createToken(token)  // ❌ DEADLOCK
}

func (r *UserRepository) createUser(user *User) error {
    r.mu.Lock()  // ❌ Ya tengo lock
    defer r.mu.Unlock()
    return nil
}

// ✅ Solución 1: Refactorizar
func (r *UserRepository) CreateWithToken(user *User, token *RefreshToken) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    // Hacer todo dentro del lock
    if err := r.db.CreateUser(user); err != nil {
        return err
    }
    
    return r.db.CreateToken(token)
}

// ✅ Solución 2: Usar transacciones
func (r *UserRepository) CreateWithToken(user *User, token *RefreshToken) error {
    tx := r.db.BeginTx(context.Background(), nil)
    defer tx.Rollback()
    
    if err := tx.CreateUser(user); err != nil {
        return err
    }
    
    if err := tx.CreateToken(token); err != nil {
        return err
    }
    
    return tx.Commit().Error
}
```

### Gateway: Multiple Service Calls

```go
// ❌ Puede causar deadlock si servicios se esperan mutuamente
func (h *Handler) ProcessOrder(order *Order) error {
    // Llamar auth service
    user, err := h.authService.GetUser(order.UserID)
    if err != nil {
        return err
    }
    
    // Llamar order service
    result, err := h.orderService.Create(order)
    if err != nil {
        return err
    }
    
    // Llamar payment service
    return h.paymentService.Charge(order.Amount)
}

// ✅ Con timeout y context cancellation
func (h *Handler) ProcessOrder(ctx context.Context, order *Order) error {
    // Timeout total: 30s
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    // Cada call respeta el timeout
    user, err := h.authService.GetUser(ctx, order.UserID)
    if err != nil {
        return err
    }
    
    result, err := h.orderService.Create(ctx, order)
    if err != nil {
        return err
    }
    
    return h.paymentService.Charge(ctx, order.Amount)
}
```

---

## ⚡ Best Practices para Prevenir Deadlock

✅ **Evita nested locks** (o ordena recursos consistentemente)
✅ **Usa timeouts** en todas las operaciones blocking
✅ **Prefiere CSP y channels** sobre locks
✅ **Hold locks por el MENOR tiempo posible**
✅ **Usa context.WithTimeout** para RPC calls
✅ **Test con -race flag**

---

## ⚠️ Antipatrones

❌ Nested locks sin ordering
❌ Channel operations sin timeout
❌ Llamadas síncronas en handlers (use goroutines)
❌ Circular service dependencies sin timeout
❌ Ignorar context deadlines
❌ Locks while esperando I/O

---

## 📊 Prevención Strategies

```
Deadlock Prevention Strategies:

1. Resource Ordering
   └─ Siempre toma recursos en el mismo orden

2. Timeout
   └─ Operaciones bloqueantes con timeout

3. Avoid Hold-and-Wait
   └─ No esperes mientras tienes recursos

4. Circular Dependency Breaking
   └─ Refactoriza flujo para evitar ciclos

5. CSP / Channels
   └─ No compartir estado mutable
```

---

## 📚 Recursos

### Go Documentation
- sync package: https://pkg.go.dev/sync
- context package: https://pkg.go.dev/context

### Libros
- "The Go Programming Language" - Capítulo Concurrency
- "Concurrency in Go" - Katherine Cox-Buday

### Herramientas
- go run -race: Detecta data races
- pprof: Profiling de goroutines
- Delve debugger: Step through goroutines

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo debuggearías un deadlock en producción?"

**Respuesta:**
> "Deadlocks son frustantes porque puedes no verlos en testing. Primero, prevencion: evito nested locks, siempre uso timeouts en operaciones bloqueantes, prefiero CSP patterns. En testing, executo `go test -race` para data races (que pueden llevar a deadlocks). Si veo que un servicio está hung en producción, uso pprof goroutine dump: `curl http://localhost:6060/debug/pprof/goroutine > goroutines.txt`. Aquí veo exactamente dónde cada goroutine está bloqueada - 'syscall.select' (esperando I/O), 'sync.(*Mutex).Lock' (esperando lock), 'channel receive' (esperando dato). Típicamente veo patrones: goroutine A esperando lock de B, B esperando lock de A (circular). Solución: refactorizar para romper ciclo - o usar `context.WithTimeout` para break el wait, o redesign las dependencias de servicio. En mi auth-service, uso context cancellation: cada RPC call respeta el context deadline del request HTTP, así si algo se cuelga, timeout lo mata en lugar de dejar goroutines bloqueadas forever."

---

#deadlocks #concurrency #debugging #synchronization #go
