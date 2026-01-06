---
tags: [kubernetes, orchestration, deployment, containerization, microservices, devops]
date: 2026-01-06
related: [Container Orchestration, Service Discovery, Docker, Deployment, Scaling]
status: reference
---

# Kubernetes

## 📋 ¿Qué es Kubernetes?

Un **sistema de orquestación de contenedores** que automatiza:
- **Despliegue** de aplicaciones containerizadas
- **Escalado** automático basado en demanda
- **Auto-healing** cuando contenedores fallan
- **Actualizaciones** sin downtime (rolling deployments)
- **Networking** entre contenedores
- **Storage** persistente

**Analogía:** Gerente de restaurante:
- ❌ Sin Kubernetes: Tú vigilas cada cocinero (CPU), cada orden (solicitud), si alguien se enferma reemplazas manualmente
- ✅ Con Kubernetes: El gerente dice "necesito 10 cocineros", Kubernetes los levanta/baja automáticamente según demanda

---

## 🎯 Problema que Resuelve

### Sin Kubernetes

```
Manual Operations:
❌ Lanzar container: docker run ...
❌ Container falla: docker run nuevamente
❌ Tráfico aumenta: ssh al servidor, docker run 5 veces más
❌ Actualización: Kill containers, esperar clientes reconecten
❌ Networking: Configurar puertos manualmente
❌ Storage: Montar volúmenes manualmente

Resultado: Trabajo de DevOps: 80% operacional, 20% innovation
```

### Con Kubernetes

```
Declarativo:
✅ Especificar: "Necesito 10 replicas de auth-service"
✅ K8s: Lanza 10, monitorea, si uno falla lo relanza
✅ Tráfico aumenta: K8s ve CPU/memoria alta → escala a 20
✅ Actualización: Rolling deployment → zero downtime
✅ Networking: Service discovery automático
✅ Storage: PersistentVolumes, PersistentVolumeClaims

Resultado: DevOps: 20% operacional, 80% innovation
```

---

## 🏗️ Conceptos Clave Kubernetes

### Pod (Smallest Unit)

```yaml
# Definición de Pod
apiVersion: v1
kind: Pod
metadata:
  name: auth-service-pod
  labels:
    app: auth-service
spec:
  containers:
  - name: auth
    image: auth-service:1.0
    ports:
    - containerPort: 50051
    env:
    - name: DATABASE_URL
      value: "postgres://db:5432/auth"
```

Un Pod = 1 o más containers (usualmente 1)
- Containers en mismo Pod comparten networking (localhost)
- Almacenamiento compartido
- Ciclo de vida común

### Deployment

```yaml
# Manage replicas, updates, rollbacks
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  replicas: 3  # Mantener 3 instancias siempre
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
        image: auth-service:1.0
        ports:
        - containerPort: 50051
        livenessProbe:  # ¿Está vivo?
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:  # ¿Listo para tráfico?
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

Deployment = Declarativo, replicas, updates automáticos

### Service (Internal Load Balancer)

```yaml
# Exponer Pods internamente
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth-service
  ports:
  - name: grpc
    port: 50051
    targetPort: 50051
  - name: http
    port: 8080
    targetPort: 8080
  type: ClusterIP  # Interno solo
```

Service = Network abstraction, load balancing, DNS

### Ingress (External Load Balancer)

```yaml
# Exponer externamente
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gateway-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gateway
            port:
              number: 8080
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
```

Ingress = External access, SSL/TLS, routing

---

## 💻 Ejemplo: Tu Proyecto en Kubernetes

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
```

### ConfigMap (Variables de ambiente)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-config
  namespace: myapp
data:
  AUTH_SERVICE_TIMEOUT: "500ms"
  LOG_LEVEL: "info"
```

### Secret (Credenciales)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: myapp
type: Opaque
data:
  DATABASE_URL: cG9zdGdyZXM6Ly91c2VyOnBhc3NAZGI6NTQzMi9hdXRo  # base64 encoded
```

### Deployment: Auth Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: myapp
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
        ports:
        - name: grpc
          containerPort: 50051
        - name: health
          containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DATABASE_URL
        - name: AUTH_SERVICE_TIMEOUT
          valueFrom:
            configMapKeyRef:
              name: auth-config
              key: AUTH_SERVICE_TIMEOUT
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
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
```

### Service: Auth Service

```yaml
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
  - name: health
    port: 8080
    targetPort: 8080
  type: ClusterIP
```

### Deployment: Gateway

```yaml
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
          value: "auth-service:50051"
        - name: AUTH_SERVICE_TIMEOUT
          valueFrom:
            configMapKeyRef:
              name: auth-config
              key: AUTH_SERVICE_TIMEOUT
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
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "250m"
```

### Service: Gateway

```yaml
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

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gateway
            port:
              number: 80
```

---

## 🚀 Operaciones Kubernetes

### Deployar

```bash
# Crear/actualizar recursos
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f auth-deployment.yaml
kubectl apply -f auth-service.yaml
kubectl apply -f gateway-deployment.yaml
kubectl apply -f gateway-service.yaml

# O un directorio
kubectl apply -f k8s/
```

### Ver estado

```bash
# Listar recursos
kubectl get pods -n myapp
kubectl get deployments -n myapp
kubectl get services -n myapp

# Ver detalles
kubectl describe pod <pod-name> -n myapp
kubectl logs <pod-name> -n myapp
kubectl logs <pod-name> -n myapp -f  # Follow

# Ver events
kubectl get events -n myapp
```

### Escalar

```bash
# Manual scaling
kubectl scale deployment auth-service --replicas=5 -n myapp

# Autoscaling (HPA)
kubectl autoscale deployment auth-service --min=2 --max=10 --cpu-percent=80 -n myapp
```

### Actualizar

```bash
# Rolling update (sin downtime)
kubectl set image deployment/auth-service auth=auth-service:2.0.0 -n myapp

# Rollback si falla
kubectl rollout undo deployment/auth-service -n myapp
```

### Debug

```bash
# Ejecutar comando en container
kubectl exec -it <pod-name> -n myapp -- /bin/sh

# Port forward (acceder localmente)
kubectl port-forward service/gateway 8080:80 -n myapp

# Acceder en browser: localhost:8080
```

---

## 📊 Kubernetes Architecture

```
Master Node (Control Plane):
├─ API Server (REST API)
├─ etcd (Key-value store)
├─ Scheduler (Coloca Pods en Nodes)
├─ Controller Manager (Mantiene estado deseado)
└─ Cloud Controller Manager

Worker Nodes:
├─ kubelet (Agent local)
├─ kube-proxy (Networking)
├─ Container Runtime (Docker, containerd)
└─ Pods (Aplicaciones)
```

---

## ⚡ Best Practices

✅ **Usa Deployments, no Pods directamente**
✅ **Define resource requests/limits**
✅ **Implementa liveness y readiness probes**
✅ **Usa namespaces para organizar**
✅ **Versionea imágenes con tags específicos**
✅ **PreStop hooks para graceful shutdown**
✅ **Monitorea con Prometheus/Grafana**

---

## ⚠️ Antipatrones

❌ Tamaño de cluster fijo (no escalable)
❌ Sin probes (pods muertos no se detectan)
❌ Latest tag en imágenes
❌ Sin resource limits (noisy neighbor)
❌ Directo en Master node
❌ Sin logging/monitoring

---

## 🔗 Ecosystem Kubernetes

```
Kubernetes Core
├─ Ingress Controller (nginx, Istio)
├─ Service Mesh (Istio, Linkerd) - [[Service Mesh]]
├─ Monitoring (Prometheus)
├─ Logging (ELK, Loki)
├─ Container Registry (Docker Hub, ECR)
├─ CI/CD (ArgoCD, Flux)
└─ Package Manager (Helm)
```

---

## 📚 Recursos

### Documentación Oficial
- kubernetes.io: https://kubernetes.io
- Interactive tutorial: https://kubernetes.io/docs/tutorials/

### Herramientas
- kubectl: CLI para K8s
- Helm: Package manager
- minikube: Local Kubernetes
- kind: Kubernetes in Docker

### Certificaciones
- CKA (Certified Kubernetes Administrator)
- CKAD (Certified Kubernetes Application Developer)

---

## 💼 En Entrevistas

**Pregunta:** "¿Cómo desplegarías tu microservicio en producción?"

**Respuesta:**
> "Usaría Kubernetes. Primero, containerizaría la aplicación con Docker (Dockerfile). Luego, escribiría manifiestos Kubernetes: un Deployment para mantener replicas (3-5 para high availability), definiendo health checks (liveness probe para detectar crashes, readiness probe para saber cuándo puede recibir traffic). Configuraría resource requests/limits para evitar 'noisy neighbor'. Para networking, crearía un Service ClusterIP interno entre auth-service y gateway, un LoadBalancer para exposición externa. Para actualizaciones: rolling deployment - Kubernetes lanza nuevas replicas con versión nueva, dirige tráfico gradualmente, mantiene las viejas como fallback. Si algo falla, rollback automático. Escalado automático con HPA basado en CPU/memoria. Monitorearía con Prometheus/Grafana, logs con Loki. Result: infraestructura declarativa, reproducible, self-healing, escalable, actualizable sin downtime. En producción, usaría managed K8s (EKS, GKE, AKS) para que cloud provider maneja control plane."

---

#kubernetes #orchestration #devops #containerization #microservices
