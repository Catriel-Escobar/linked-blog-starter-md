---
tags: [architecture, async, messaging, events, kafka, rabbitmq]
date: 2026-01-05
related: [Microservices, Message Queue, Event Sourcing, CQRS]
status: reference
---

# Event-Driven Architecture (EDA)

## 📋 ¿Qué es Event-Driven Architecture?

Un **paradigma arquitectónico** donde los componentes del sistema se comunican mediante **eventos** (notificaciones de que algo pasó) en lugar de llamadas directas, permitiendo comunicación asíncrona y desacoplamiento.

**Analogía:** Como un sistema de notificaciones push:
- 📱 Subes una foto a Instagram (evento)
- 🔔 Tus seguidores reciben notificación automáticamente
- 👥 No necesitas llamar a cada seguidor individualmente

---

## 🎯 Problema que Resuelve

### Arquitectura Síncrona (Request/Response)

```
User Registration Flow:

Client → Gateway → Auth Service (crear usuario)
                   ↓
                   ✅ Usuario creado
                   ↓
              Email Service (enviar welcome email)
                   ↓ (5s esperando)
              Analytics (registrar evento)
                   ↓ (2s esperando)
              CRM Service (crear contacto)
                   ↓ (3s esperando)
              ← Response: "User created"

Total: 10+ segundos ❌
```

**Problemas:**
- ❌ Lento (espera a todos los servicios)
- ❌ Si Email Service falla → todo falla
- ❌ Acoplamiento (Auth Service conoce todos los servicios)
- ❌ Difícil agregar nuevos servicios

### Arquitectura Event-Driven

```
User Registration Flow:

Client → Gateway → Auth Service (crear usuario)
                   ↓
                   ✅ Usuario creado
                   ↓
              Publica: "UserRegistered" event
                   ↓ (instantáneo)
              ← Response: "User created"

Total: <1 segundo ✅

Mientras tanto (asíncrono):
    Event Bus (Kafka/RabbitMQ)
        ↓
    ├→ Email Service (escucha "UserRegistered")
    ├→ Analytics (escucha "UserRegistered")
    ├→ CRM Service (escucha "UserRegistered")
    └→ Notification Service (escucha "UserRegistered")
```

**Ventajas:**
- ✅ Rápido (no espera a otros servicios)
- ✅ Resiliente (si Email falla, otros siguen)
- ✅ Desacoplado (Auth no conoce a los consumidores)
- ✅ Fácil agregar servicios (solo subscribirse)

---

## 🏗️ Componentes Principales

### 1. **Eventos (Events)**

Un **registro inmutable** de algo que pasó.

```go
type Event struct {
    ID        string    `json:"id"`         // UUID único
    Type      string    `json:"type"`       // "UserRegistered"
    Timestamp time.Time `json:"timestamp"`  // Cuándo ocurrió
    Data      any       `json:"data"`       // Payload del evento
    Version   string    `json:"version"`    // "v1" (versionado)
}

type UserRegisteredEvent struct {
    UserID    string `json:"user_id"`
    Email     string `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}
```

**Características:**
- ✅ Inmutables (nunca cambian)
- ✅ Pasado (UserRegistered, no RegisterUser)
- ✅ Self-contained (toda la info necesaria)

### 2. **Event Producer (Publicador)**

Servicio que **publica** eventos.

```go
type EventPublisher interface {
    Publish(ctx context.Context, event Event) error
}

// Auth Service (productor)
func (s *AuthService) Register(ctx context.Context, email, password string) error {
    // 1. Crear usuario
    user, err := s.repo.Create(ctx, email, password)
    if err != nil {
        return err
    }
    
    // 2. Publicar evento
    event := Event{
        ID:        uuid.New().String(),
        Type:      "UserRegistered",
        Timestamp: time.Now(),
        Data: UserRegisteredEvent{
            UserID:    user.ID,
            Email:     user.Email,
            CreatedAt: user.CreatedAt,
        },
    }
    
    return s.publisher.Publish(ctx, event)
}
```

### 3. **Event Bus / Message Broker**

Sistema que **transporta** eventos entre servicios.

**Opciones populares:**
- **Kafka**: High-throughput, persistente, streaming
- **RabbitMQ**: AMQP, flexible, fácil de usar
- **NATS**: Ligero, rápido
- **Redis Streams**: Simple, in-memory
- **AWS SQS/SNS**: Managed, cloud-native

### 4. **Event Consumer (Suscriptor)**

Servicio que **escucha** y **procesa** eventos.

```go
type EventConsumer interface {
    Subscribe(ctx context.Context, eventType string, handler func(Event) error) error
}

// Email Service (consumidor)
func (s *EmailService) Start(ctx context.Context) error {
    return s.consumer.Subscribe(ctx, "UserRegistered", func(event Event) error {
        var data UserRegisteredEvent
        json.Unmarshal(event.Data, &data)
        
        // Enviar welcome email
        return s.sendWelcomeEmail(data.Email)
    })
}

// Analytics Service (otro consumidor)
func (s *AnalyticsService) Start(ctx context.Context) error {
    return s.consumer.Subscribe(ctx, "UserRegistered", func(event Event) error {
        var data UserRegisteredEvent
        json.Unmarshal(event.Data, &data)
        
        // Registrar en analytics
        return s.trackUserSignup(data.UserID)
    })
}
```

---

## 📊 Patrones de Event-Driven

### 1. **Pub/Sub (Publish/Subscribe)**

Múltiples consumidores reciben el **mismo** evento.

```
Auth Service
     ↓ Publica "UserRegistered"
Event Bus (Kafka Topic: user-events)
     ↓ Fan-out a todos los suscriptores
     ├→ Email Service
     ├→ Analytics Service
     └→ CRM Service
```

**Uso:** Notificaciones, broadcasting.

### 2. **Event Streaming**

Eventos se guardan en un **log ordenado** y pueden ser re-procesados.

```
Kafka Topic: user-events
┌─────────────────────────────────────┐
│ [1] UserRegistered (user-123)       │
│ [2] EmailVerified (user-123)        │
│ [3] UserRegistered (user-456)       │
│ [4] ProfileUpdated (user-123)       │
└─────────────────────────────────────┘
         ↓
Consumer puede leer desde offset 1, 2, etc.
```

**Uso:** Event sourcing, replay, audit log.

### 3. **CQRS (Command Query Responsibility Segregation)**

Separar operaciones de **escritura** (commands) de **lectura** (queries).

```
Write Side (Commands):
Client → CreateUser Command
         ↓
    Auth Service (write DB)
         ↓
    Publica UserCreated Event
    
Read Side (Queries):
    UserCreated Event
         ↓
    Read Model Service
         ↓
    Actualiza Read DB (optimizada para queries)
         ↓
    Client → Query User (rápido)
```

**Ventajas:**
- ✅ Escalado independiente (lecturas vs escrituras)
- ✅ Optimización específica (escritura vs lectura)

### 4. **Saga Pattern**

Transacciones distribuidas usando eventos.

```
Order Saga:

1. OrderService → CreateOrder
   ↓ Publica "OrderCreated"

2. PaymentService escucha → ProcessPayment
   ✅ Success → Publica "PaymentCompleted"
   ❌ Failure → Publica "PaymentFailed"
   
3. InventoryService escucha "PaymentCompleted" → ReserveInventory
   ✅ Success → Publica "InventoryReserved"
   ❌ Failure → Publica "InventoryFailed"
   
4. Si "InventoryFailed":
   PaymentService escucha → RefundPayment (compensación)
```

---

## 💻 Implementación con Kafka

### Setup

```bash
# Docker Compose
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
```

### Producer (Auth Service)

```go
package events

import (
    "context"
    "encoding/json"
    
    "github.com/segmentio/kafka-go"
)

type KafkaPublisher struct {
    writer *kafka.Writer
}

func NewKafkaPublisher(brokers []string, topic string) *KafkaPublisher {
    return &KafkaPublisher{
        writer: &kafka.Writer{
            Addr:     kafka.TCP(brokers...),
            Topic:    topic,
            Balancer: &kafka.LeastBytes{},
        },
    }
}

func (p *KafkaPublisher) Publish(ctx context.Context, event Event) error {
    data, err := json.Marshal(event)
    if err != nil {
        return err
    }
    
    return p.writer.WriteMessages(ctx, kafka.Message{
        Key:   []byte(event.ID),
        Value: data,
    })
}

// Uso en Auth Service
func (s *AuthService) Register(ctx context.Context, email, password string) error {
    user, err := s.repo.Create(ctx, email, password)
    if err != nil {
        return err
    }
    
    event := Event{
        ID:        uuid.New().String(),
        Type:      "UserRegistered",
        Timestamp: time.Now(),
        Data: UserRegisteredEvent{
            UserID: user.ID,
            Email:  user.Email,
        },
    }
    
    return s.publisher.Publish(ctx, event)
}
```

### Consumer (Email Service)

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    
    "github.com/segmentio/kafka-go"
)

type EmailService struct {
    reader *kafka.Reader
}

func NewEmailService(brokers []string, topic, groupID string) *EmailService {
    return &EmailService{
        reader: kafka.NewReader(kafka.ReaderConfig{
            Brokers: brokers,
            Topic:   topic,
            GroupID: groupID,  // Consumer group (balanceo automático)
        }),
    }
}

func (s *EmailService) Start(ctx context.Context) error {
    for {
        msg, err := s.reader.ReadMessage(ctx)
        if err != nil {
            return err
        }
        
        var event Event
        if err := json.Unmarshal(msg.Value, &event); err != nil {
            log.Printf("Error unmarshaling event: %v", err)
            continue
        }
        
        if event.Type == "UserRegistered" {
            var data UserRegisteredEvent
            json.Unmarshal(event.Data, &data)
            
            if err := s.sendWelcomeEmail(data.Email); err != nil {
                log.Printf("Error sending email: %v", err)
                // Retry logic aquí
            }
        }
    }
}

func (s *EmailService) sendWelcomeEmail(email string) error {
    log.Printf("Sending welcome email to %s", email)
    // Implementación real de envío
    return nil
}
```

---

## 🔄 Outbox Pattern (Garantizar Consistencia)

**Problema:** ¿Qué pasa si guardas el usuario en DB pero falla publicar el evento?

```go
// ❌ NO ATÓMICO
func (s *AuthService) Register(ctx context.Context, email, password string) error {
    user, err := s.repo.Create(ctx, email, password)  // DB commit
    if err != nil {
        return err
    }
    
    // Si esto falla, usuario creado pero sin evento 💥
    return s.publisher.Publish(ctx, event)
}
```

**Solución: Outbox Pattern**

```go
// 1. Guardar evento en la misma transacción
func (s *AuthService) Register(ctx context.Context, email, password string) error {
    return s.txManager.WithTransaction(ctx, func(tx Transaction) error {
        // Crear usuario
        user, err := s.repo.CreateTx(tx, email, password)
        if err != nil {
            return err
        }
        
        // Guardar evento en tabla "outbox" (misma transacción)
        event := OutboxEvent{
            ID:        uuid.New().String(),
            Type:      "UserRegistered",
            Payload:   marshalEvent(user),
            CreatedAt: time.Now(),
            Processed: false,
        }
        
        return s.outboxRepo.SaveTx(tx, event)
    })
}

// 2. Background worker publica eventos pendientes
func (w *OutboxWorker) Run(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    
    for {
        select {
        case <-ticker.C:
            events, _ := w.outboxRepo.GetPending(ctx)
            
            for _, event := range events {
                if err := w.publisher.Publish(ctx, event); err == nil {
                    w.outboxRepo.MarkProcessed(ctx, event.ID)
                }
            }
        case <-ctx.Done():
            return
        }
    }
}
```

**Garantía:** Si el usuario se guarda, el evento **eventualmente** se publicará.

---

## 📈 Ventajas y Desventajas

### Ventajas ✅

- **Desacoplamiento**: Servicios no se conocen entre sí
- **Escalabilidad**: Consumidores independientes
- **Resilencia**: Si un consumidor falla, otros continúan
- **Flexibilidad**: Agregar consumidores sin modificar productores
- **Async**: No bloquea el flow principal
- **Audit log**: Eventos = historial de todo lo que pasó

### Desventajas ❌

- **Complejidad**: Más componentes (message broker)
- **Debugging**: Difícil rastrear flujo completo
- **Eventual consistency**: Datos no actualizados instantáneamente
- **Message ordering**: Garantizar orden es complejo
- **Duplicación**: Manejar eventos duplicados (idempotencia)
- **Monitoreo**: Necesitas observabilidad avanzada

---

## 🎯 Cuándo Usar Event-Driven

### ✅ Usar EDA si:
- Necesitas **desacoplamiento** entre servicios
- Tienes **workflows asíncronos** (emails, notificaciones)
- Requieres **alta escalabilidad**
- Múltiples servicios reaccionan al **mismo evento**
- Necesitas **audit trail** completo
- Tienes **microservicios complejos**

### ❌ NO usar EDA si:
- Aplicación **simple** (CRUD básico)
- Necesitas **consistencia inmediata**
- Equipo **pequeño** (overhead operacional)
- **Debugging** es crítico (más difícil con eventos)
- No tienes **infraestructura** para message brokers

---

## 🔗 Integración con tu Proyecto

### Cómo agregarías EDA a tu Auth Service

```go
// 1. Definir eventos
type UserRegisteredEvent struct {
    UserID    string
    Email     string
    Timestamp time.Time
}

// 2. Publicar en Auth Service
func (s *AuthService) Register(ctx context.Context, email, password string) error {
    // Crear usuario + guardar evento en outbox (misma transacción)
    return s.txManager.WithTransaction(ctx, func(tx Transaction) error {
        user, err := s.userRepo.CreateTx(tx, email, password)
        if err != nil {
            return err
        }
        
        event := OutboxEvent{
            Type: "UserRegistered",
            Data: UserRegisteredEvent{
                UserID:    user.ID,
                Email:     user.Email,
                Timestamp: time.Now(),
            },
        }
        
        return s.outboxRepo.SaveTx(tx, event)
    })
}

// 3. Background worker publica a Kafka
// (Ya implementado en tu código con outbox_pg.go)

// 4. Crear Email Service (nuevo microservicio)
type EmailService struct {
    kafkaReader *kafka.Reader
}

func (s *EmailService) Start() {
    for {
        msg, _ := s.kafkaReader.ReadMessage(context.Background())
        
        var event UserRegisteredEvent
        json.Unmarshal(msg.Value, &event)
        
        s.sendWelcomeEmail(event.Email)
    }
}
```

### Docker Compose actualizado

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9092:9092"
  
  auth-service:
    environment:
      KAFKA_BROKERS: "kafka:9092"
  
  email-service:  # Nuevo servicio
    build: ./email-service
    environment:
      KAFKA_BROKERS: "kafka:9092"
      KAFKA_TOPIC: "user-events"
```

---

## 📚 Recursos

### Librerías Go
- **Kafka**: github.com/segmentio/kafka-go
- **RabbitMQ**: github.com/streadway/amqp
- **NATS**: github.com/nats-io/nats.go
- **Watermill**: github.com/ThreeDotsLabs/watermill (framework EDA)

### Libros
- "Building Event-Driven Microservices" - Adam Bellemare
- "Designing Event-Driven Systems" - Ben Stopford

### Artículos
- "What is Event-Driven Architecture?" - AWS
- "Event-Driven Architecture Patterns" - Martin Fowler

---

## 💼 En Entrevistas

**Pregunta:** "¿Cuál es la diferencia entre arquitectura síncrona y event-driven?"

**Respuesta:**
> "En arquitectura síncrona, los servicios se comunican mediante request/response directo, esperando la respuesta. Es simple pero crea acoplamiento y el caller debe esperar. En event-driven, los servicios publican eventos de lo que pasó (por ejemplo, 'UserRegistered') a un message broker como Kafka, y otros servicios se suscriben y reaccionan de forma asíncrona. Esto desacopla servicios, mejora escalabilidad, y permite agregar nuevos consumidores sin modificar el productor. Por ejemplo, al registrar un usuario, el auth-service publica el evento y múltiples servicios (email, analytics, CRM) pueden procesarlo independientemente sin bloquear la respuesta al cliente. Usaría EDA cuando necesito workflows asíncronos, desacoplamiento, o múltiples servicios que reaccionan al mismo evento."

---

#event-driven #kafka #async #microservices #messaging #architecture
