---
tags: [csp, concurrency, go, channels, patterns, architecture]
date: 2026-01-05
related: [Goroutines, Channels, Race Conditions, Deadlocks, Concurrency]
status: reference
---

# CSP (Communicating Sequential Processes)

## 📋 ¿Qué es CSP?

Un **modelo de concurrencia donde procesos independientes se comunican mediante canales** en lugar de compartir memoria.

**Frase clave de Go:** "Don't communicate by sharing memory; share memory by communicating"

**Analogía:** Cafetería:
- ❌ Compartir memoria: Todos acceden a la caja registradora (conflictos, locks, caos)
- ✅ CSP: Clientes forman una fila (secuencia), pasan dinero al vendedor (comunicación), vendedor procesa uno por uno

---

## 🎯 Problema que Resuelve

### Sin CSP (Shared Memory)

```go
// ❌ Múltiples goroutines compartiendo un variable
var count = 0
var mu sync.Mutex

func increment() {
    mu.Lock()
    count++
    mu.Unlock()
}

func decrement() {
    mu.Lock()
    count--
    mu.Unlock()
}

func main() {
    for i := 0; i < 1000; i++ {
        go increment()
        go decrement()
    }
    
    // Problema: necesito locks en CADA acceso
    // Fácil cometer errores (deadlocks)
    // Código difícil de entender
}
```

### Con CSP (Channels)

```go
// ✅ Procesos comunicándose via canales
func counter() <-chan int {
    out := make(chan int)
    
    go func() {
        count := 0
        for {
            select {
            case cmd := <-commandChan:
                if cmd == "inc" {
                    count++
                } else if cmd == "dec" {
                    count--
                }
            case out <- count:
            }
        }
    }()
    
    return out
}

func main() {
    // Cada goroutine es independiente
    // Se comunica via canales
    // Sin locks, sin race conditions
}
```

---

## 🏗️ Conceptos CSP

### 1. **Goroutines (Sequential Processes)**

```go
// Cada goroutine es un proceso independiente
go func() {
    // Este código ejecuta en paralelo
    fmt.Println("Proceso 1")
}()

go func() {
    // Este también
    fmt.Println("Proceso 2")
}()

// Main sigue sin esperar
```

### 2. **Canales (Communication)**

```go
// Canal para pasar integers
ch := make(chan int)

// Enviar
ch <- 42

// Recibir
value := <-ch

// Cerrar
close(ch)
```

### 3. **Select (Multiplexing)**

```go
// Esperar en múltiples canales
select {
case msg := <-ch1:
    fmt.Println("Mensaje de ch1:", msg)
case msg := <-ch2:
    fmt.Println("Mensaje de ch2:", msg)
case <-time.After(1*time.Second):
    fmt.Println("Timeout")
}
```

---

## 💻 Patrones CSP Comunes

### 1. **Producer-Consumer**

```go
func producer(out chan<- int) {
    for i := 0; i < 10; i++ {
        out <- i  // Enviar item
    }
    close(out)
}

func consumer(in <-chan int) {
    for item := range in {  // Recibir items
        fmt.Println("Recibido:", item)
    }
}

func main() {
    ch := make(chan int)
    
    go producer(ch)
    consumer(ch)
    
    // Producer y consumer corren independientemente
    // Se comunican via canal
    // Sin locks, sin race conditions
}
```

### 2. **Worker Pool**

```go
// N workers esperando tareas
func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        fmt.Printf("Worker %d procesa job %d\n", id, job)
        results <- job * 2
    }
}

func main() {
    jobs := make(chan int, 10)
    results := make(chan int)
    
    // Crear 3 workers
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }
    
    // Enviar jobs
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)
    
    // Recibir resultados
    for a := 1; a <= 5; a++ {
        <-results
    }
}

// Ventajas:
// - Workers reutilizables
// - Load balancing automático
// - Fácil de escalar
```

### 3. **Fan-Out / Fan-In**

```go
// Fan-Out: Un input, múltiples outputs
func fanOut(in <-chan int) (chan int, chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    
    go func() {
        for item := range in {
            out1 <- item
            out2 <- item
        }
        close(out1)
        close(out2)
    }()
    
    return out1, out2
}

// Fan-In: Múltiples inputs, un output
func fanIn(in1 <-chan int, in2 <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        for {
            select {
            case item := <-in1:
                out <- item
            case item := <-in2:
                out <- item
            }
        }
    }()
    
    return out
}

func main() {
    // Crear pipeline
    in := make(chan int)
    out1, out2 := fanOut(in)
    out := fanIn(out1, out2)
    
    go func() {
        for i := 0; i < 5; i++ {
            in <- i
        }
        close(in)
    }()
    
    // Consumir
    for item := range out {
        fmt.Println(item)
    }
}
```

### 4. **Pipeline**

```go
// Etapa 1: Generar números
func generate(count int) <-chan int {
    out := make(chan int)
    go func() {
        for i := 0; i < count; i++ {
            out <- i
        }
        close(out)
    }()
    return out
}

// Etapa 2: Filtrar pares
func filter(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for item := range in {
            if item%2 == 0 {
                out <- item
            }
        }
        close(out)
    }()
    return out
}

// Etapa 3: Duplicar
func duplicate(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for item := range in {
            out <- item * 2
        }
        close(out)
    }()
    return out
}

func main() {
    // Pipeline: generate -> filter -> duplicate
    nums := generate(10)
    evens := filter(nums)
    results := duplicate(evens)
    
    for result := range results {
        fmt.Println(result)
    }
}

// Ventajas:
// - Modular
// - Fácil de entender
// - Cada etapa independiente
// - Composable
```

---

## 🎯 Tu Proyecto: CSP en Auth Service

### Message Queue Pattern

```go
// auth-service/internal/service/auth_service.go
type AuthService struct {
    userRepo         UserRepository
    verificationChan chan<- VerificationTask
}

type VerificationTask struct {
    Email string
    Code  string
}

// Inicializar con worker goroutines
func NewAuthService(userRepo UserRepository) *AuthService {
    verificationChan := make(chan VerificationTask, 100)
    
    // N workers procesando verificaciones
    for i := 0; i < 5; i++ {
        go s.verificationWorker(verificationChan)
    }
    
    return &AuthService{
        userRepo:         userRepo,
        verificationChan: verificationChan,
    }
}

// Register: envía tarea, no espera respuesta
func (s *AuthService) Register(ctx context.Context, email, password string) (*User, error) {
    // Crear usuario
    user, err := s.userRepo.Create(ctx, email, password)
    if err != nil {
        return nil, err
    }
    
    // Enviar tarea de verificación (no-blocking)
    select {
    case s.verificationChan <- VerificationTask{
        Email: email,
        Code:  generateCode(),
    }:
        // Tarea enqueued
    case <-ctx.Done():
        return nil, ctx.Err()
    }
    
    return user, nil
}

// Worker: procesa verificaciones
func (s *AuthService) verificationWorker(in <-chan VerificationTask) {
    for task := range in {
        // Procesar verificación (enviar email, guardar código)
        s.sendVerificationEmail(task.Email, task.Code)
    }
}
```

### Multiple Channels

```go
// gateway/internal/handlers/auth.go
type RequestTracker struct {
    requests   <-chan Request
    responses  chan<- Response
    errors     chan<- error
}

func (rt *RequestTracker) ProcessRequests() {
    for {
        select {
        case req := <-rt.requests:
            // Procesar request
            resp, err := s.processRequest(req)
            
            if err != nil {
                rt.errors <- err
            } else {
                rt.responses <- resp
            }
            
        case <-time.After(30*time.Second):
            rt.errors <- errors.New("timeout processing requests")
        }
    }
}
```

---

## 📊 Ventajas de CSP vs Shared Memory

```
Shared Memory (Locks):
├─ Necesito sync.Mutex en cada variable
├─ Riesgo de deadlocks
├─ Difícil de razonar
├─ Errores sutiles
└─ Performance: Lock contention

CSP (Channels):
├─ Comunicación explícita
├─ Evita race conditions naturalmente
├─ Código claro y composable
├─ Más fácil de debuggear
└─ Performance: Lock-free (en mayoría casos)
```

---

## ⚡ Best Practices

✅ **Usa canales para comunicación entre goroutines**
✅ **Cierra canales desde sender, no receiver**
✅ **Usa `<-chan` y `chan<-` para claridad**
✅ **Evita compartir memory entre goroutines**
✅ **Usa buffered channels con cuidado**
✅ **Implementa timeouts en select**

---

## ⚠️ Antipatrones

❌ Compartir structs via memoria entre goroutines
❌ No cerrar canales
❌ Receivers cerrando canales
❌ Send en canal después de close
❌ Buffered channels innecesarios
❌ Ignorar context en select

---

## 🔗 Relación con otros patrones

```
CSP
├─ [[Race Conditions]] ← evita
├─ [[Deadlocks]] ← puede introducir
├─ [[Request Cancellation]] ← integrar con context
├─ [[Graceful Shutdown]] ← usar para draining
└─ Worker Pool ← implementación
```

---

## 📚 Recursos

### Go Documentation
- Channels: https://go.dev/tour/concurrency/2
- Select: https://go.dev/tour/concurrency/5
- Pipelines: https://go.dev/blog/pipelines

### Libros
- "The Go Programming Language" - Capítulo Concurrency
- "Concurrency in Go" - Katherine Cox-Buday

### Artículos
- "Pipelines" - Go Blog
- "Concurrency is not parallelism" - Rob Pike

---

## 💼 En Entrevistas

**Pregunta:** "¿Cuál es la diferencia entre CSP y Shared Memory para concurrencia?"

**Respuesta:**
> "CSP (Communicating Sequential Processes) es el modelo que Go usa por defecto. En lugar de múltiples goroutines compartiendo variables con locks, cada goroutine es independiente y se comunica con otras via canales. Ejemplo: si tengo 10 goroutines que necesitan actualizar un contador, con shared memory necesito un mutex, 10 goroutines compitiendo por el lock, deadlock risk. Con CSP: creo una goroutine 'counter-service' que mantiene el estado, otros goroutines le envían 'increment' y 'decrement' comandos via canal. El counter-service procesa secuencialmente, sin locks, sin race conditions. La frase de Go es: 'Don't communicate by sharing memory; share memory by communicating'. CSP escala mejor porque evita lock contention, es más fácil razonar (comunicación explícita), y menos propenso a bugs. En mi auth-service implementé worker pools: múltiples goroutines recibiendo tareas de un canal, procesando independientemente. Load balancing automático sin sincronización compleja."

---

#csp #concurrency #channels #go #architecture
