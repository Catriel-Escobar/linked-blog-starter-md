---
tags: [container-orchestration, docker, kubernetes, microservices, deployment, scaling]
date: 2026-01-06
related: [Kubernetes, Docker, Container Registries, Service Discovery, Networking]
status: reference
---

# Container Orchestration

## 📋 ¿Qué es Container Orchestration?

Un **sistema que automatiza el despliegue, scaling y manejo de contenedores** en un cluster.

**Analogía:** Director de orquesta:
- ❌ Sin orquestación: Cada músico decide cuándo tocar, si se cansa se va, nadie sabe cuándo terminar
- ✅ Con orquestación: Director dice "violas, 3 copias", "si una se cansa, reemplazarla", "cuando alguien se equivoca, repetir", sincronización perfecta

---

## 🎯 Problemas que Resuelve

### Sin Container Orchestration

```
Manual:
❌ Deployar: docker run en cada servidor
❌ Scaling: SSH a servidor, docker run múltiples veces
❌ Fallos: Monitorear cada contenedor, relanzar manualmente
❌ Networking: Configurar puertos, networking entre máquinas
❌ Storage: Montar volúmenes manualmente
❌ Actualizaciones: Matar containers, esperar reconexiones
❌ Logs: Ssh a cada servidor para ver logs

Resultado: DevOps es 100% operacional, 0% innovation
```

### Con Container Orchestration

```
Declarativo:
✅ Deployar: kubectl apply -f deployment.yaml
✅ Scaling: kubectl scale replicas=10
✅ Fallos: Auto-restart de containers
✅ Networking: Automático entre contenedores
✅ Storage: PersistentVolumes, automático
✅ Actualizaciones: Rolling deployment, zero downtime
✅ Logs: kubectl logs, agregados

Resultado: DevOps es 20% operacional, 80% innovation
```

---

## 🏗️ Componentes de Orchestration

### 1. **Scheduling**

```
Problema: "Tengo 100 containers, 10 servidores, ¿dónde pongo cada uno?"

Orquestador:
├─ Analiza recursos cada servidor
├─ Analiza requirements de container
├─ Calcula mejor placement
└─ Asigna container a servidor

Ejemplo:
┌─ Servidor 1: CPU=50%, Memory=60%
├─ Servidor 2: CPU=30%, Memory=40%
├─ Servidor 3: CPU=80%, Memory=90%
│
├─ Container app-web (CPU=20%, RAM=512MB)
│  └─ Placement: Servidor 2 (mejor fit)
│
└─ Container db (CPU=40%, RAM=4GB)
   └─ Placement: Servidor 1 (más recursos)
```

### 2. **Replication & Scaling**

```
Usuario declara:
"Necesito 5 instancias de auth-service siempre"

Orquestador:
├─ Ve: 3 instancias corriendo
├─ Acción: Levanta 2 más
├─ Resultado: 5 instancias

Cuando tráfico sube:
├─ Monitorea CPU/Memory
├─ Ve: CPU > 80%
├─ Auto-scales: Levanta a 8 instancias
├─ Resultado: CPU normaliza a 60%
```

### 3. **Health Management**

```
Orquestador continuamente:
├─ Revisa: ¿Container sigue vivo? (liveness probe)
├─ Revisa: ¿Listo para tráfico? (readiness probe)
├─ Si muere: Levanta automáticamente
├─ Si timeout health check: Mata y reinicia

Ejemplo timeline:
10:00 - auth-service-1 se levanta
10:05 - Health check: OK
10:10 - Health check: OK
10:15 - Database falla, auth-service se congela
10:16 - Health check: FAIL, timeout
10:17 - Orquestador mata container y relanza
10:22 - Database vuelve, nuevo container conecta
10:30 - Servicio disponible sin intervención manual
```

### 4. **Networking**

```
Orquestador:
├─ Service Discovery: ¿Dónde está el servicio?
├─ Load Balancing: Distribuye tráfico
├─ Network Policies: Quién puede hablar con quién
└─ Ingress: Acceso externo

Ejemplo:
gateway (Pod 1) necesita auth-service
├─ Pregunta: "¿Dónde auth-service?"
├─ Orquestador: "10.0.0.5:50051, 10.0.0.6:50051, 10.0.0.7:50051"
├─ gateway: Conecta a 10.0.0.5
├─ 10.0.0.5 muere, siguiente request va a 10.0.0.6
└─ Auto load-balanced
```

### 5. **Storage**

```
Orquestador maneja:
├─ Ephemeral: Datos temporales del container
├─ Persistent: Datos que surviven container muertes
│  ├─ PersistentVolumes: Storage pool
│  ├─ PersistentVolumeClaims: Solicitud de storage
│  └─ Binding automático
│
└─ ConfigMaps/Secrets: Configuración e secretos

Ejemplo:
DB container necesita /data/postgres
├─ Declara: "Necesito 100GB PersistentVolume"
├─ Orquestador: Busca PV disponible
├─ Monta automáticamente
├─ Container muere, PV persiste
├─ Nuevo container monta mismo PV
└─ Datos preserved
```

### 6. **Update & Rollback**

```
Deployment de v1.0 → v2.0:

Sin orquestación (manual):
├─ Kill v1 containers
├─ Start v2 containers
├─ Downtime: 30s-2min

Con orquestación (rolling):
├─ Start v2 container #1
├─ Verify health: OK
├─ Kill v1 container #1
├─ Start v2 container #2
├─ Verify health: OK
├─ Kill v1 container #2
├─ ... repeat
└─ Zero downtime, gradual rollout

Si v2 tiene bug:
├─ Rollback a v1 con 1 comando
└─ Sistema preserva estado de v2 (para debugging)
```

---

## 💻 Principales Orquestadores

### 1. **Kubernetes**

```yaml
# Declarativo, state-driven
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth
        image: auth-service:1.0.0
```

**Ventajas:**
- ✅ Más poderoso
- ✅ Comunidad enorme
- ✅ Estándar de industria
- ✅ Todos los cloud providers soportan

**Desventajas:**
- ❌ Curva aprendizaje pronunciada
- ❌ Complexity overhead
- ❌ Overkill para aplicaciones simples

### 2. **Docker Swarm**

```bash
# Declarativo, más simple que K8s
docker service create \
  --name auth-service \
  --replicas 3 \
  --publish 50051:50051 \
  auth-service:1.0.0
```

**Ventajas:**
- ✅ Mucho más simple que K8s
- ✅ Built-in en Docker
- ✅ Menos recursos

**Desventajas:**
- ❌ Menos features que K8s
- ❌ Comunidad menor
- ❌ Menos adoption

### 3. **Nomad (HashiCorp)**

```hcl
job "auth-service" {
  datacenters = ["dc1"]
  type        = "service"

  group "api" {
    count = 3

    task "auth" {
      driver = "docker"

      config {
        image = "auth-service:1.0.0"
        ports = ["grpc"]
      }

      resources {
        cpu    = 500
        memory = 256
      }
    }
  }
}
```

**Ventajas:**
- ✅ Multi-cloud nativo
- ✅ Flexible (containers, VMs, bare metal)
- ✅ Integración con Consul

**Desventajas:**
- ❌ Menos adoption que K8s
- ❌ Comunidad menor

### 4. **Docker Compose (Local)**

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: password

  auth-service:
    build: ./auth-service
    ports:
      - "50051:50051"
    environment:
      DATABASE_URL: postgres://postgres:password@postgres:5432/auth
    depends_on:
      - postgres

  gateway:
    build: ./gateway
    ports:
      - "8080:8080"
    environment:
      AUTH_SERVICE_HOST: auth-service:50051
    depends_on:
      - auth-service
```

**Ventajas:**
- ✅ Perfecto para desarrollo local
- ✅ Simple
- ✅ Multi-container fácil

**Desventajas:**
- ❌ Solo local, no production-ready
- ❌ Sin scaling automático
- ❌ Sin self-healing

---

## 🎯 Tu Proyecto: Container Orchestration

### Development: Docker Compose

```bash
# Tu setup actual
docker-compose up

# Automáticamente:
├─ Levanta postgres
├─ Levanta auth-service (dependencia)
└─ Levanta gateway (dependencia)

# Networking automático:
├─ gateway ↔ auth-service: auth-service:50051
├─ auth-service ↔ postgres: postgres:5432
└─ Todo en red `myapp_default`
```

### Production: Kubernetes

```yaml
# kube/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp

---
# kube/postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc

---
# kube/auth-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth
        image: auth-service:1.0.0
        ports:
        - name: grpc
          containerPort: 50051
        - name: health
          containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: database-url
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
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"

---
# kube/auth-service-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: myapp
spec:
  selector:
    app: auth-service
  ports:
  - name: grpc
    port: 50051
    targetPort: 50051
  type: ClusterIP

---
# kube/gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      containers:
      - name: gateway
        image: gateway:1.0.0
        ports:
        - name: http
          containerPort: 8080
        env:
        - name: AUTH_SERVICE_HOST
          value: "auth-service.myapp.svc.cluster.local:50051"
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

---
# kube/gateway-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: gateway
  namespace: myapp
spec:
  selector:
    app: gateway
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: LoadBalancer
```

---

## 📊 Comparativa Orquestadores

```
Feature           | K8s    | Swarm  | Nomad  | Compose
─────────────────────────────────────────────────
Complexity        | High   | Low    | Medium | Very Low
Learning Curve    | Hard   | Easy   | Medium | Easy
Scaling Auto      | Yes    | Yes    | Yes    | No
Multi-cloud       | Yes    | Limited| Yes    | No
StatefulSets      | Yes    | No     | Yes    | No
Networking        | Advanced| Simple | Medium | Simple
Adoption          | Huge   | Small  | Medium | Dev Only
Production Ready  | Yes    | Yes    | Yes    | No
Community         | Huge   | Small  | Medium | Large
Cost              | High   | Low    | Medium | Free
```

---

## ⚡ Best Practices

✅ **Conoce herramientas de tu orquestador**
✅ **Define resource requests/limits**
✅ **Implementa health checks**
✅ **Monitorea cluster**
✅ **Automatiza deployments (CI/CD)**
✅ **Prepara disaster recovery**
✅ **Documenta runbooks**

---

## ⚠️ Antipatrones

❌ Seleccionar orquestador sin evaluar necesidades
❌ Sin planning de recursos
❌ Sin backup/DR plan
❌ Manual deployments
❌ Sin monitoring/alerting
❌ Ignorar security

---

## 🔗 Ecosystem Orquestación

```
Container Orchestration
├─ [[Kubernetes]] ← Estándar
├─ [[Docker]] ← Runtime
├─ [[Service Discovery]] ← Networking
├─ [[Container Registries]] ← Image storage
├─ [[CI/CD]] ← Deployments
├─ [[Monitoring]] ← Prometheus/Grafana
└─ [[Logging]] ← ELK/Loki
```

---

## 📚 Recursos

### Kubernetes
- Official Docs: https://kubernetes.io/docs/
- Interactive Tutorial: https://kubernetes.io/docs/tutorials/

### Docker
- Swarm: https://docs.docker.com/engine/swarm/
- Compose: https://docs.docker.com/compose/

### Nomad
- Docs: https://www.nomadproject.io/docs
- Job Spec: https://www.nomadproject.io/docs/job-specification

### Comparativas
- CNCF Landscape: https://landscape.cncf.io/
- Container Orchestration Comparison: Medium articles

---

## 💼 En Entrevistas

**Pregunta:** "¿Qué herramienta de orquestación usarías y por qué?"

**Respuesta:**
> "Depende del contexto. Para un startup, Docker Compose localmente + AWS ECS o Heroku en production. Simple, mantenible, low ops overhead. Para una empresa con múltiples servicios, Kubernetes. Es el estándar de industria, todos los cloud providers lo soportan (EKS, GKE, AKS), comunidad enorme, muy flexible. En mi auth-service + gateway, usaría Kubernetes: defino Deployments para cada servicio (3 replicas auth-service, 2 replicas gateway), Services para networking (auth-service:50051 automáticamente balancea entre replicas), Ingress para exposición externa con TLS. Orquestador maneja: scheduling (¿dónde pongo cada container?), scaling (si CPU > 80%, agregar replicas), health checks (si container falla, relanzar), rolling updates (v1.0 → v2.0 sin downtime), storage persistente (postgres data persiste si pod muere). Sin orquestador, todo manual. Con orquestador, es declarativo: digo 'necesito 3 auth-services', K8s lo mantiene. Superior."

---

#container-orchestration #kubernetes #docker #deployment #microservices
