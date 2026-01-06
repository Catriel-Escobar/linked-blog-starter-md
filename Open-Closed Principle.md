---
tags: [solid-principles, design-principles, architecture, extensibility]
date: 2026-01-05
related: [SOLID Principles, Single Responsibility, Liskov Substitution, Design Patterns]
status: reference
---

# Open/Closed Principle (OCP)

## 📋 ¿Qué es el Open/Closed Principle?

El **segundo principio SOLID** que establece:

> **Software entities (clases, módulos, funciones) deben estar ABIERTOS para extensión pero CERRADOS para modificación.**

**En otras palabras:**
- ✅ Agregar nueva funcionalidad sin cambiar código existente
- ❌ NO modificar código que ya funciona

**Analogía:** Un enchufe eléctrico:
- Abierto para extensión: Puedo conectar diferentes dispositivos (lámpara, cargador, TV)
- Cerrado para modificación: No necesito abrir el enchufe para agregarlos

---

## 🎯 Problema que Resuelve

### ❌ Violar OCP (Cerrado para Extensión)

```go
type PaymentProcessor struct{}

func (p *PaymentProcessor) ProcessPayment(paymentType string, amount float64) bool {
    if paymentType == "credit_card" {
        // Procesar con tarjeta de crédito
        return processCreditCard(amount)
    } else if paymentType == "paypal" {
        // Procesar con PayPal
        return processPayPal(amount)
    } else if paymentType == "stripe" {
        // Procesar con Stripe
        return processStripe(amount)
    }
    
    return false
}

// ❌ Problema: Cada vez que agrego nuevo método de pago,
// ❌ DEBO MODIFICAR PaymentProcessor.ProcessPayment()
// ❌ Violo: "closed for modification"

// Para agregar Google Pay:
// 1. Abrir PaymentProcessor
// 2. Agregar else if paymentType == "google_pay"
// 3. Recompilar
// 4. Testear TODO

// Riesgo: Podría romper casos existentes
```

### ✅ Cumplir OCP (Abierto para Extensión)

```go
// Definir interfaz (contrato)
type PaymentMethod interface {
    Process(amount float64) bool
    Validate() error
}

// Implementaciones específicas
type CreditCardPayment struct {
    CardNumber string
}

func (cc *CreditCardPayment) Process(amount float64) bool {
    return processCreditCard(cc.CardNumber, amount)
}

func (cc *CreditCardPayment) Validate() error {
    if !isValidCardNumber(cc.CardNumber) {
        return ErrInvalidCard
    }
    return nil
}

type PayPalPayment struct {
    Email string
}

func (pp *PayPalPayment) Process(amount float64) bool {
    return processPayPal(pp.Email, amount)
}

func (pp *PayPalPayment) Validate() error {
    if !isValidEmail(pp.Email) {
        return ErrInvalidEmail
    }
    return nil
}

type StripePayment struct {
    Token string
}

func (sp *StripePayment) Process(amount float64) bool {
    return processStripe(sp.Token, amount)
}

func (sp *StripePayment) Validate() error {
    if !isValidToken(sp.Token) {
        return ErrInvalidToken
    }
    return nil
}

// Processor refactorizado
type PaymentProcessor struct{}

func (p *PaymentProcessor) ProcessPayment(method PaymentMethod, amount float64) error {
    // NO toca la implementación: solo usa la interfaz
    if err := method.Validate(); err != nil {
        return err
    }
    
    if !method.Process(amount) {
        return ErrProcessingFailed
    }
    
    return nil
}

// ✅ Ventaja: Para agregar Google Pay:
// 1. Crear GooglePayPayment struct
// 2. Implementar PaymentMethod interface
// 3. Usar con PaymentProcessor (sin cambiar nada)

type GooglePayPayment struct {
    Token string
}

func (gp *GooglePayPayment) Process(amount float64) bool {
    return processGooglePay(gp.Token, amount)
}

func (gp *GooglePayPayment) Validate() error {
    return nil  // Validación
}

// Listo! Funciona sin tocar PaymentProcessor
processor := &PaymentProcessor{}
processor.ProcessPayment(&GooglePayPayment{...}, 100.0)
```

---

## 🏗️ Cómo Cumplir OCP

### 1. **Usar Abstracciones (Interfaces)**

```go
// ❌ Acoplado a implementaciones concretas
type OrderService struct {
    db *PostgresDB  // Acoplado
}

func (s *OrderService) GetOrder(id string) (*Order, error) {
    return s.db.GetOrder(id)  // Solo funciona con PostgreSQL
}

// ✅ Desacoplado mediante interfaz
type OrderRepository interface {
    GetOrder(ctx context.Context, id string) (*Order, error)
}

type OrderService struct {
    repo OrderRepository  // Abierto: puede ser cualquier implementación
}

func (s *OrderService) GetOrder(ctx context.Context, id string) (*Order, error) {
    return s.repo.GetOrder(ctx, id)
}

// Puedo cambiar implementación sin tocar OrderService:
service := &OrderService{repo: &PostgresOrderRepository{}}  // PostgreSQL
service := &OrderService{repo: &MongoOrderRepository{}}     // MongoDB
service := &OrderService{repo: &RedisOrderRepository{}}     // Redis (cache)
```

### 2. **Herencia y Métodos Template**

```go
type ReportGenerator interface {
    Generate(data interface{}) string
}

type BaseReport struct{}

func (br *BaseReport) Format(content string) string {
    return fmt.Sprintf("=== REPORT ===\n%s\n=============", content)
}

type PDFReport struct {
    BaseReport
}

func (pr *PDFReport) Generate(data interface{}) string {
    content := pr.buildContent(data)
    return pr.Format(content)
}

func (pr *PDFReport) buildContent(data interface{}) string {
    return "PDF content here"
}

type HTMLReport struct {
    BaseReport
}

func (hr *HTMLReport) Generate(data interface{}) string {
    content := hr.buildContent(data)
    return hr.Format(content)
}

func (hr *HTMLReport) buildContent(data interface{}) string {
    return "<html>HTML content here</html>"
}

// ✅ Agregar JSONReport sin modificar BaseReport
type JSONReport struct {
    BaseReport
}

func (jr *JSONReport) Generate(data interface{}) string {
    content := jr.buildContent(data)
    return jr.Format(content)
}

func (jr *JSONReport) buildContent(data interface{}) string {
    b, _ := json.Marshal(data)
    return string(b)
}
```

### 3. **Estrategia Pattern**

```go
// ❌ Violar OCP: Agregar validación requiere modificar función
func validateUser(user *User, validationType string) error {
    if validationType == "strict" {
        if user.Email == "" {
            return ErrInvalidEmail
        }
        if user.Name == "" {
            return ErrInvalidName
        }
    } else if validationType == "basic" {
        if user.Email == "" {
            return ErrInvalidEmail
        }
    }
    return nil
}

// ✅ Cumplir OCP: Usar estrategias
type ValidationStrategy interface {
    Validate(user *User) error
}

type StrictValidation struct{}

func (sv *StrictValidation) Validate(user *User) error {
    if user.Email == "" {
        return ErrInvalidEmail
    }
    if user.Name == "" {
        return ErrInvalidName
    }
    return nil
}

type BasicValidation struct{}

func (bv *BasicValidation) Validate(user *User) error {
    if user.Email == "" {
        return ErrInvalidEmail
    }
    return nil
}

type UserValidator struct {
    strategy ValidationStrategy
}

func (uv *UserValidator) Validate(user *User) error {
    return uv.strategy.Validate(user)
}

// ✅ Agregar nueva estrategia sin modificar nada
type CustomValidation struct{}

func (cv *CustomValidation) Validate(user *User) error {
    // Lógica personalizada
    return nil
}

validator := &UserValidator{strategy: &CustomValidation{}}
validator.Validate(user)
```

---

## 💻 Tu Proyecto: Aplicar OCP

### ❌ Violación Actual (Ejemplo)

```go
// gateway/internal/client/auth_client.go

func (c *AuthClient) Login(ctx context.Context, email, password string) (*pb.LoginResponse, error) {
    // Si quiero agregar mocking para tests:
    // Tengo que modificar este método (violación de OCP)
    
    resp, err := c.client.Login(ctx, &pb.LoginRequest{...})
    return resp, err
}
```

### ✅ Cumplir OCP

```go
// Definir interfaz para el cliente
type AuthClient interface {
    Login(ctx context.Context, email, password string) (*pb.LoginResponse, error)
    Register(ctx context.Context, email, password string) (*pb.RegisterResponse, error)
}

// Implementación real
type RealAuthClient struct {
    conn pb.AuthServiceClient
}

func (rac *RealAuthClient) Login(ctx context.Context, email, password string) (*pb.LoginResponse, error) {
    return rac.conn.Login(ctx, &pb.LoginRequest{
        Email:    email,
        Password: password,
    })
}

// Implementación mock para tests
type MockAuthClient struct {
    loginFunc func(ctx context.Context, email, password string) (*pb.LoginResponse, error)
}

func (mac *MockAuthClient) Login(ctx context.Context, email, password string) (*pb.LoginResponse, error) {
    return mac.loginFunc(ctx, email, password)
}

// Servicio que usa la interfaz
type AuthService struct {
    client AuthClient  // Abierto a extensión: puede ser Real o Mock
}

func (as *AuthService) Authenticate(ctx context.Context, email, password string) (*User, error) {
    // No toca la implementación
    return as.client.Login(ctx, email, password)
}

// ✅ Testing sin modificar código
func TestLogin(t *testing.T) {
    mockClient := &MockAuthClient{
        loginFunc: func(ctx context.Context, email, password string) (*pb.LoginResponse, error) {
            return &pb.LoginResponse{User: &pb.User{Id: "123"}}, nil
        },
    }
    
    service := &AuthService{client: mockClient}
    user, _ := service.Authenticate(context.Background(), "test@test.com", "password")
    
    assert.Equal(t, "123", user.Id)
}
```

---

## 🏗️ Patrones que Implementan OCP

| Patrón | Cómo Implementa OCP | Ejemplo |
|--------|-------------------|---------|
| **Strategy** | Encapsula algoritmos en interfaces | PaymentMethod interface |
| **Decorator** | Extiende sin modificar original | CachedRepository wrapper |
| **Template Method** | Define estructura, permite sobrescribir | BaseReport.Generate() |
| **Factory** | Crea objetos sin especificar clases | PaymentFactory |
| **Observer** | Agrega listeners sin modificar originales | Event subscribers |
| **State** | Cambia comportamiento según estado | StateMachine |

---

## 📊 Niveles de Cumplimiento OCP

### Nivel 1: Rígido (Mala práctica)

```go
func ProcessOrder(orderType string, order *Order) {
    if orderType == "digital" {
        // Procesar digital
    } else if orderType == "physical" {
        // Procesar físico
    }
    // Modificar para cada tipo es violación OCP
}
```

### Nivel 2: Con Interfaces (Bueno)

```go
type OrderProcessor interface {
    Process(order *Order) error
}

// Implementaciones específicas
type DigitalOrderProcessor struct{}
type PhysicalOrderProcessor struct{}

// No necesitas modificar nada para agregar nuevo tipo
```

### Nivel 3: Con Abstracciones (Excelente)

```go
type Order interface {
    GetProcessor() OrderProcessor
    Validate() error
}

type DigitalOrder struct{}
func (do *DigitalOrder) GetProcessor() OrderProcessor {
    return &DigitalOrderProcessor{}
}

// El cliente no conoce tipos específicos
func processAnyOrder(order Order) error {
    return order.GetProcessor().Process(order)
}
```

---

## ⚡ Beneficios de OCP

✅ **Reducir cambios:** Agregar funcionalidad sin tocar código existente  
✅ **Reducir bugs:** No modificas código que ya funciona  
✅ **Testing:** Fácil crear mocks/stubs  
✅ **Reutilización:** Componentes reutilizables  
✅ **Mantenibilidad:** Cambios localizados  

---

## ⚠️ Riesgos de NO Cumplir OCP

❌ Cada cambio requiere modificación de código existente  
❌ Riesgo de romper funcionalidad actual  
❌ Código poco flexible  
❌ Testing difícil  
❌ Deuda técnica crece  

---

## 📚 Recursos

### Libros
- "Design Principles and Design Patterns" - Robert C. Martin
- "Clean Code" - Robert C. Martin

### Artículos
- SOLID Principles: https://en.wikipedia.org/wiki/SOLID
- OCP by Example: https://www.baeldung.com/solid-principles

### Go
- Effective Go: https://golang.org/doc/effective_go#interfaces

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo diseñarías un sistema que fácilmente soporte nuevos métodos de pago?"

**Respuesta:**
> "Usaría el Open/Closed Principle. Definiría una interfaz PaymentMethod que todos los métodos de pago implementan (CreditCard, PayPal, Stripe, etc.). El procesador de pagos usa solo la interfaz, no conoce tipos específicos. Así, para agregar Google Pay, solo creo GooglePayPayment que implementa PaymentMethod - sin tocar el procesador. El código está cerrado para modificación (procesador no cambia) pero abierto para extensión (nuevos métodos de pago). Ventajas: bajo riesgo de romper código, fácil testear (puedo hacer MockPaymentMethod), y nuevas funcionalidades se agregan sin cambios existentes. En Go, los interfaces pequeños hacen esto muy natural."

---

#open-closed-principle #solid-principles #architecture #design-patterns #extensibility
