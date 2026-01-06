---
tags: [graceful-shutdown, lifecycle, signals, cleanup, reliability]
date: 2026-01-05
related: [Reliability, Signal Handling, Context Cancellation, Connection Pooling]
status: reference
---

# Graceful Shutdown

## 📋 ¿Qué es Graceful Shutdown?

Un proceso controlado donde una aplicación:
1. **Para de aceptar nuevas solicitudes**
2. **Completa las solicitudes en progreso**
3. **Libera recursos** (conexiones, archivos)
4. **Se detiene limpiamente**

**Analogía:** Cerrar un restaurante:
- ❌ Abruptamente: "¡Fuera! ¡Se acabó!" (clientes molestos, comida desperdiciada)
- ✅ Gracefully: "Última orden en 15 min", termine las órdenes pendientes, cierre caja

---

## 🎯 Problema que Resuelve

### Sin Graceful Shutdown

```
Kill signal → Aplicación muere INMEDIATAMENTE

Consecuencias:
❌ Solicitudes en progreso se pierden
❌ Conexiones DB no se cierran (leak)
❌ Transacciones incompletas
❌ Archivos no guardados
❌ Clientes reciben 500 error

Ejemplo:
- Usuario pagando → Signal SIGTERM → Proceso muere
- Pago quedó incompleto, dinero perdido
```

### Con Graceful Shutdown

```
Kill signal → Aplicación controla cierre:

1. Recibe SIGTERM
2. Detiene listener HTTP (no nuevas conexiones)
3. Espera a que terminen requests en progreso (con timeout)
4. Cierra DB connections
5. Libera recursos
6. Exitosamente shutdown

Consecuencias:
✅ Requests completan
✅ Recursos liberados correctamente
✅ Sin pérdida de datos
✅ Clientes satisfechos
```

---

## 🏗️ Patrones de Graceful Shutdown

### 1. **Signal Handling Básico**

```go
import (
    "os"
    "os/signal"
    "syscall"
)

func main() {
    // Crear canal para señales
    sigChan := make(chan os.Signal, 1)
    
    // Registrar señales a capturar
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
    
    // Iniciar servidor
    server := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }
    
    go func() {
        log.Printf("Starting server on %s", server.Addr)
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()
    
    // Esperar señal
    sig := <-sigChan
    log.Printf("Received signal: %v", sig)
    
    // Gracefully shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("Shutdown error: %v", err)
    }
    
    log.Println("Server shutdown successfully")
}
```

### 2. **Shutdown Manager**

```go
type GracefulShutdown struct {
    server   *http.Server
    db       *sql.DB
    timeout  time.Duration
}

func (gs *GracefulShutdown) Start(sigChan <-chan os.Signal) {
    sig := <-sigChan
    log.Printf("Received signal: %v, shutting down gracefully", sig)
    
    // Shutdown con timeout
    ctx, cancel := context.WithTimeout(context.Background(), gs.timeout)
    defer cancel()
    
    // 1. Dejar de aceptar nuevas conexiones
    if err := gs.server.Shutdown(ctx); err != nil {
        log.Printf("Server shutdown error: %v", err)
    }
    
    // 2. Cerrar conexiones DB
    if err := gs.db.Close(); err != nil {
        log.Printf("Database close error: %v", err)
    }
    
    log.Println("Graceful shutdown completed")
}

func main() {
    db, _ := sql.Open("postgres", "...")
    
    server := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }
    
    gs := &GracefulShutdown{
        server:  server,
        db:      db,
        timeout: 30 * time.Second,
    }
    
    // Capturar señales
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
    
    // Iniciar servidor
    go func() {
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()
    
    // Esperar señal y shutdown
    gs.Start(sigChan)
}
```

### 3. **Con Health Check**

```go
type HealthChecker struct {
    isHealthy bool
    mu        sync.RWMutex
}

func (hc *HealthChecker) SetUnhealthy() {
    hc.mu.Lock()
    defer hc.mu.Unlock()
    hc.isHealthy = false
}

func (hc *HealthChecker) IsHealthy() bool {
    hc.mu.RLock()
    defer hc.mu.RUnlock()
    return hc.isHealthy
}

func (hc *HealthChecker) HealthHandler(w http.ResponseWriter, r *http.Request) {
    if !hc.IsHealthy() {
        w.WriteHeader(http.StatusServiceUnavailable)
        w.Write([]byte("Service unhealthy"))
        return
    }
    
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}

// En shutdown
func (gs *GracefulShutdown) Start(sigChan <-chan os.Signal) {
    <-sigChan
    
    // 1. Marcar como unhealthy (load balancer deja de enviar traffic)
    gs.healthChecker.SetUnhealthy()
    
    // 2. Esperar a que se drenen conexiones
    time.Sleep(5 * time.Second)
    
    // 3. Shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    gs.server.Shutdown(ctx)
    gs.db.Close()
}
```

---

## 💻 Ejemplo: Auth Service con Graceful Shutdown

```go
// auth-service/cmd/server/main.go
package main

import (
    "context"
    "log"
    "net"
    "os"
    "os/signal"
    "syscall"
    "time"
    
    "google.golang.org/grpc"
    "github.com/myapp/auth-service/internal/server"
)

func main() {
    // Crear listener
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("Failed to listen: %v", err)
    }
    
    // Crear gRPC server
    grpcServer := grpc.NewServer()
    
    // Registrar servicios
    authServer := server.NewAuthServer()
    pb.RegisterAuthServiceServer(grpcServer, authServer)
    
    // Iniciar servidor en goroutine
    go func() {
        log.Printf("Starting gRPC server on :50051")
        if err := grpcServer.Serve(lis); err != nil {
            log.Fatalf("Server error: %v", err)
        }
    }()
    
    // Capturar señales
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
    
    // Esperar señal
    sig := <-sigChan
    log.Printf("Received signal: %v", sig)
    
    // Graceful shutdown
    done := make(chan struct{})
    go func() {
        log.Println("Gracefully shutting down...")
        
        // GracefulStop: espera a que terminen RPC calls en progreso
        grpcServer.GracefulStop()
        
        close(done)
    }()
    
    // Timeout: si no termina en 30s, force stop
    select {
    case <-done:
        log.Println("Graceful shutdown completed")
    case <-time.After(30 * time.Second):
        log.Println("Shutdown timeout, force stopping")
        grpcServer.Stop()
    }
}
```

---

## 🎯 Casos de Uso

### 1. **Rolling Deployment**

```
v1 Replica 1 ← Traffic
v1 Replica 2 ← Traffic
v1 Replica 3 ← Traffic

Deploy v2:
1. Start v2 Replica 1 (sin traffic)
2. Mark v1 Replica 1 unhealthy (health check)
3. Wait for active requests to complete (30s)
4. Kill v1 Replica 1
5. Send traffic to v2 Replica 1
6. Repeat para replicas 2 y 3

Resultado: Zero downtime
```

### 2. **Blue-Green Deployment**

```
Blue (v1):   Replica 1, 2, 3 ← Traffic

Deploy v2 (Green):
1. Start Green: Replica 1, 2, 3 (sin traffic)
2. Test Green (health checks, smoke tests)
3. Switch traffic: Load balancer → Green
4. Monitor Green (si falla, switch back a Blue)
5. Keep Blue por 1 hora (rollback rápido)

Graceful shutdown de Blue después del periodo
```

### 3. **Maintenance Window**

```
Recibir solicitud de mantenimiento:
1. Marcar como unhealthy
2. Load balancer deja de enviar traffic
3. Esperar a que terminen requests (max 5 min)
4. Ejecutar mantenimiento
5. Restart y volver a healthy
```

---

## 📊 Señales de Sistema

```
SIGTERM (15)
└─ Terminar (cleanly)
   └─ Graceful shutdown
   
SIGKILL (9)
└─ Kill inmediato (no se puede capturar)
   └─ ❌ Sin graceful shutdown
   
SIGINT (2)
└─ Interrupt (Ctrl+C)
   └─ Graceful shutdown
```

---

## ⚡ Best Practices

✅ **Captura SIGTERM y SIGINT**
✅ **Usa timeout (30-60s)** para shutdown
✅ **Drena conexiones** antes de matar
✅ **Cierra recursos en orden inverso** (HTTP → DB → conexiones)
✅ **Loguea shutdown** (para debugging)
✅ **Implementa health checks** (load balancer-compatible)
✅ **Testea shutdown** en tus tests

---

## ⚠️ Antipatrones

❌ Ignorar señales de shutdown
❌ No usar timeout (puede colgarse forever)
❌ Cerrar recursos en orden incorrecto
❌ No loguear shutdown
❌ No implementar health checks
❌ Shutdown inmediato (kill -9)

---

## 🔗 Kubernetes Integration

En Kubernetes, graceful shutdown es crítico:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: auth-service
spec:
  containers:
  - name: auth
    image: auth-service:latest
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # Esperar a drain
    terminationGracePeriodSeconds: 30  # Timeout
  
  # Health check
  livenessProbe:
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 10
  
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5
```

---

## 📚 Recursos

### Go
- signal package: https://pkg.go.dev/os/signal
- http.Server.Shutdown: https://pkg.go.dev/net/http#Server.Shutdown
- grpc.Server.GracefulStop: https://pkg.go.dev/google.golang.org/grpc#Server.GracefulStop

### Artículos
- "Graceful Shutdown in Go" - Medium
- Kubernetes lifecycle hooks: https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/

---

## 💼 En Entrevistas

**Pregunta:** "¿Qué pasa si matas un servidor durante una transacción de pago?"

**Respuesta:**
> "Sin graceful shutdown: el servidor muere inmediatamente, la transacción de pago se cancela a mitad de camino, dinero se pierde. Con graceful shutdown: cuando recibo SIGTERM, marco el servidor como unhealthy (load balancer deja de enviar traffic), espero a que completen las transacciones en progreso (con timeout de 30s), cierro la conexión DB limpiamente, y luego muero. El resultado: las transacciones se completan, se persisten en DB, sin pérdida de datos. En Kubernetes, configuro `terminationGracePeriodSeconds: 30` para que la plataforma respete mi timeout, e implemento un `preStop` hook que espera a que se drenen las conexiones."

---

#graceful-shutdown #lifecycle #signals #reliability #kubernetes
