---
tags: [design-principles, oop, architecture, composition, inheritance]
date: 2026-01-05
related: [Decorator Pattern, Embedding, Interface Segregation, Polymorphism]
status: reference
---

# Composition Over Inheritance

## 📋 ¿Qué es Composition Over Inheritance?

Un **principio de diseño** que promueve usar **composición** (crear objetos que contienen otros objetos) en lugar de **herencia** (crear jerarquías de clases padre-hijo).

**Analogía:**
- **Herencia:** Es un pájaro → Vuela, come, respira
- **Composición:** Tengo alas (puedo volar), tengo pico (puedo comer), tengo pulmones (puedo respirar)

Composición es más flexible: puedo tener objetos sin alas que no vuelan, o sin pico que no comen.

---

## 🎯 Problema que Resuelve

### Herencia (Problema)

```go
// ❌ Jerarquía de herencia rígida

type Animal struct {
    Name string
    Age  int
}

type Dog struct {
    Animal          // Herencia
    Breed  string
}

func (d *Dog) Bark() {
    fmt.Println("Woof!")
}

type Bird struct {
    Animal          // Herencia
    WingSpan float64
}

func (b *Bird) Fly() {
    fmt.Println("Flying!")
}

type Penguin struct {
    Bird            // Hereda de Bird
    // Pero... pingüinos NO vuelan!
    // ❌ Problema: heredó Fly() aunque no lo necesita
}

// ¿Qué pasa con un animal que nada?
// ¿Necesito crear una clase para cada combinación?
// - Perro que nada (Dog + Swimming)
// - Pájaro que nada (Bird + Swimming)
// - Pez que nada (Fish + Swimming)
// - Pez que vuela (Flying Fish)
// - Pájaro que nada (Duck) - ¿Hereda de Bird o de Swimming?

// ❌ EXPLOSIÓN: 2^n combinaciones para n comportamientos
```

### Composición (Solución)

```go
// ✅ Composición flexible

// Interfaces pequeñas y específicas
type Walker interface {
    Walk()
}

type Swimmer interface {
    Swim()
}

type Flyer interface {
    Fly()
}

// Tipos base
type Animal struct {
    Name string
    Age  int
}

// Componer comportamientos
type Dog struct {
    Animal
    walker Walker  // Composición
}

func (d *Dog) Walk() {
    d.walker.Walk()
}

type Bird struct {
    Animal
    flyer Flyer  // Composición
}

func (b *Bird) Fly() {
    b.flyer.Fly()
}

type Penguin struct {
    Animal
    swimmer Swimmer  // Composición
    walker  Walker   // Puede tener múltiples comportamientos
}

func (p *Penguin) Swim() {
    p.swimmer.Swim()
}

func (p *Penguin) Walk() {
    p.walker.Walk()
}

// ✅ Ventajas:
// - Pingüino no hereda Fly() innecesariamente
// - Fácil combinar comportamientos
// - Sin explosión de clases
// - Más flexible
```

---

## 🏗️ Herencia vs Composición

### Herencia: "Es Un"

```
Animal
  ├─ Dog (es un animal)
  ├─ Cat (es un animal)
  └─ Bird (es un animal)
       ├─ Eagle (es un pájaro)
       └─ Penguin (es un pájaro)

Problema: Cuando Penguin hereda Bird.Fly(), pero no puede volar
```

### Composición: "Tiene Un"

```
Dog (tiene un Animal, tiene movimiento Walking)
Cat (tiene un Animal, tiene movimiento Walking)
Bird (tiene un Animal, tiene movimiento Flying)
Penguin (tiene un Animal, tiene movimiento Swimming y Walking)

Ventaja: Cada objeto tiene exactamente lo que necesita
```

---

## 💻 Ejemplo 1: Vehículos

### ❌ Herencia (Problema)

```go
type Vehicle struct {
    Brand string
    Year  int
}

type Car struct {
    Vehicle
}

func (c *Car) Drive() {
    fmt.Println("Car driving...")
}

type Airplane struct {
    Vehicle
}

func (a *Airplane) Fly() {
    fmt.Println("Plane flying...")
}

// ¿Qué pasa con un anfibio (vehículo terrestre + acuático)?
type Amphibian struct {
    Vehicle
    // ¿Hereda de Car o Truck? No hay "múltiple herencia" en Go
}

// Necesitaría: AmphibianCar, AmphibianTruck, AmphibianBoat, etc.
// ❌ Explosión combinatoria
```

### ✅ Composición (Solución)

```go
// Interfaces para capacidades
type Engine interface {
    Start() string
    Stop() string
}

type Propeller interface {
    Thrust() string
}

type Wheel interface {
    Roll() string
}

// Implementaciones
type GasEngine struct{}

func (e *GasEngine) Start() string {
    return "Engine started with gas"
}

func (e *GasEngine) Stop() string {
    return "Engine stopped"
}

type JetEngine struct{}

func (je *JetEngine) Start() string {
    return "Jet engine ignited"
}

func (je *JetEngine) Stop() string {
    return "Jet engine shut down"
}

// Vehículo con composición
type Vehicle struct {
    Brand  string
    Year   int
    engine Engine
}

type Car struct {
    vehicle Vehicle
    wheels  []Wheel
}

func (c *Car) Drive() string {
    start := c.vehicle.engine.Start()
    return fmt.Sprintf("%s, Car rolling", start)
}

type Airplane struct {
    vehicle   Vehicle
    propeller Propeller
}

func (a *Airplane) Fly() string {
    start := a.vehicle.engine.Start()
    return fmt.Sprintf("%s, Plane flying", start)
}

type Amphibian struct {
    vehicle   Vehicle
    wheels    []Wheel
    propeller Propeller
}

func (a *Amphibian) Drive() string {
    start := a.vehicle.engine.Start()
    return fmt.Sprintf("%s, Amphibian rolling", start)
}

func (a *Amphibian) Swim() string {
    return fmt.Sprintf("%s, Amphibian swimming", a.propeller.Thrust())
}

// ✅ Ventajas:
// - Sin explosión de clases
// - Reutilizable: GasEngine funciona en Car, Amphibian, etc.
// - Flexible: cambiar engine en runtime
```

---

## 💻 Ejemplo 2: Servicio de Base de Datos

### ❌ Herencia (Problema)

```go
type Repository struct {
    db *sql.DB
}

func (r *Repository) Create(data interface{}) error {
    // Implementación
}

func (r *Repository) Read(id string) (interface{}, error) {
    // Implementación
}

type UserRepository struct {
    Repository
}

type OrderRepository struct {
    Repository
}

type ProductRepository struct {
    Repository
}

// Si necesito cachear usuarios pero no órdenes:
type CachedUserRepository struct {
    UserRepository
    cache Cache
}

// Problema: Código duplicado, jerarquía profunda
```

### ✅ Composición (Solución)

```go
// Interfaces
type Storage interface {
    Create(data interface{}) error
    Read(id string) (interface{}, error)
    Update(id string, data interface{}) error
    Delete(id string) error
}

type Cache interface {
    Get(key string) (interface{}, error)
    Set(key string, value interface{}, duration time.Duration)
}

// Implementación de storage
type PostgresStorage struct {
    db *sql.DB
}

func (ps *PostgresStorage) Create(data interface{}) error {
    // PostgreSQL logic
}

type RedisStorage struct {
    client *redis.Client
}

func (rs *RedisStorage) Create(data interface{}) error {
    // Redis logic
}

// Decorador: Añadir caching a cualquier storage
type CachedStorage struct {
    storage Storage
    cache   Cache
}

func (cs *CachedStorage) Read(id string) (interface{}, error) {
    // Intenta obtener del cache
    if val, err := cs.cache.Get(id); err == nil {
        return val, nil
    }
    
    // Si no está en cache, obtener del storage
    data, err := cs.storage.Read(id)
    if err != nil {
        return nil, err
    }
    
    // Guardar en cache
    cs.cache.Set(id, data, 1*time.Hour)
    return data, nil
}

func (cs *CachedStorage) Create(data interface{}) error {
    return cs.storage.Create(data)
}

// Uso: Composición flexible
pgStorage := &PostgresStorage{db: db}
cachedStorage := &CachedStorage{
    storage: pgStorage,
    cache:   redisCache,
}

// O sin cache:
directStorage := &PostgresStorage{db: db}

// ✅ Ventajas:
// - Una sola clase CachedStorage que funciona con cualquier storage
// - Fácil cambiar storage (PostgreSQL, Redis, MongoDB)
// - Sin jerarquía profunda
// - Reutilizable
```

---

## 💻 Ejemplo 3: Tu Proyecto - Auth Service

### Aplicar Composición

```go
// Interfaces pequeñas
type PasswordHasher interface {
    Hash(password string) (string, error)
    Verify(hash, password string) bool
}

type TokenGenerator interface {
    Generate(userID string) (string, error)
    Validate(token string) (string, error)
}

type UserRepository interface {
    GetByEmail(ctx context.Context, email string) (*User, error)
    Create(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
}

type EmailSender interface {
    Send(to, subject, body string) error
}

// Servicio de autenticación con composición
type AuthService struct {
    userRepo      UserRepository      // Composición
    passwordHash  PasswordHasher      // Composición
    tokenGen      TokenGenerator      // Composición
    emailSender   EmailSender         // Composición
}

func (s *AuthService) Register(ctx context.Context, email, password string) (*User, error) {
    // Hash password
    hash, _ := s.passwordHash.Hash(password)
    
    // Crear usuario
    user := &User{Email: email, Password: hash}
    s.userRepo.Create(ctx, user)
    
    // Enviar email
    s.emailSender.Send(email, "Welcome", "...")
    
    // Generar token
    token, _ := s.tokenGen.Generate(user.ID)
    
    return user, nil
}

// ✅ Ventajas:
// - Fácil testear (mock los interfaces)
// - Cambiar implementación sin cambiar AuthService
// - Reutilizar componentes en otros servicios
```

---

## 📊 Cuándo Usar Cada Una

### Usar Herencia si:
- Relación verdadera "es un"
- Jerarquía simple (max 2-3 niveles)
- Compartir implementación común
- (En Go casi nunca: prefiere composición)

### Usar Composición si:
- ✅ Comportamientos independientes
- ✅ Múltiples comportamientos juntos
- ✅ Necesitas cambiar en runtime
- ✅ Evitar jerarquías profundas
- ✅ Testing (fácil mockear)

---

## 🔗 Go: Embedding (Composición)

Go promueve composición:

```go
// Embedding (composición ligera)
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter struct {
    Reader
    Writer
}

// Uso
var rw ReadWriter = &bufio.ReadWriter{...}
rw.Read(...)   // Llama Reader.Read()
rw.Write(...)  // Llama Writer.Write()
```

---

## ⚡ Ventajas de Composición

✅ **Flexibilidad:** Cambiar comportamientos en runtime  
✅ **Reutilización:** Componentes funcionan en múltiples contextos  
✅ **Testabilidad:** Fácil mockear interfaces  
✅ **Simplicidad:** Evitar jerarquías complejas  
✅ **Mantenibilidad:** Cambios localizados  

---

## ⚠️ Desventajas de Composición

❌ **Verbosidad:** Más código (delegación)  
❌ **Indirección:** No tan intuitivo como herencia  
❌ **Overhead:** Pequeño performance hit  

---

## 📚 Recursos

### Go
- "Composition vs Inheritance in Go" - Medium
- Effective Go: https://golang.org/doc/effective_go#embedding

### Principios
- SOLID Principles
- Composition Over Inheritance (Gang of Four)

---

## 💼 En Entrevistas

**Pregunta:** "¿Cuándo usarías herencia vs composición?"

**Respuesta:**
> "Prefiero composición en casi todos los casos. Herencia crea jerarquías rígidas que son difíciles de mantener: si necesito un objeto con múltiples comportamientos, herencia produce explosión de clases. Con composición, defino interfaces pequeñas (PasswordHasher, TokenGenerator) y compongo un servicio con ellas. Ventajas: puedo cambiar implementaciones sin modificar el servicio, es fácil testear (mock los interfaces), y los componentes se reutilizan en otros servicios. Go promociona composición mediante embedding. Por ejemplo, en mi auth-service uso composición: AuthService contiene UserRepository, PasswordHasher, TokenGenerator, EmailSender. Cada uno tiene una responsabilidad clara y pueden ser intercambiados."

---

#composition #inheritance #design-principles #oop #golang
