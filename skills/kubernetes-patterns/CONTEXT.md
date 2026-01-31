# Kubernetes Patterns

Essential Kubernetes patterns for deployments, services, configuration, and troubleshooting.

## When to Apply

- Deploying applications to Kubernetes
- Configuring resources and limits
- Setting up networking and services
- Managing configuration and secrets
- Troubleshooting pod issues
- Implementing zero-downtime deployments

## Deployment Patterns

### Basic Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
```

### Rolling Update Strategy
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods over desired
      maxUnavailable: 0  # Zero downtime
```

### Blue-Green Deployment
```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
---
# Green deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
---
# Service points to one version
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue  # Switch to green for cutover
  ports:
  - port: 80
    targetPort: 8080
```

### Canary Deployment
```yaml
# Stable deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
---
# Canary deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
---
# Service routes to both
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # Matches both stable and canary
  ports:
  - port: 80
    targetPort: 8080
```

## Services

### ClusterIP (Internal)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP  # Default
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

### LoadBalancer (External)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    # Cloud-specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

### Headless Service (StatefulSet)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  clusterIP: None  # Headless
  selector:
    app: mydb
  ports:
  - port: 5432
```

## Ingress

### Basic Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

### TLS Termination
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

## Configuration

### ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  config.yaml: |
    server:
      port: 8080
    database:
      host: postgres
---
# Usage in Deployment
spec:
  containers:
  - name: myapp
    volumeMounts:
    - name: config
      mountPath: /app/config
  volumes:
  - name: config
    configMap:
      name: myapp-config
```

### Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  database-url: "postgres://user:pass@host/db"
  api-key: "secret123"
---
# Usage as environment variables
spec:
  containers:
  - name: myapp
    envFrom:
    - secretRef:
        name: myapp-secrets
    # Or individual keys
    env:
    - name: DB_URL
      valueFrom:
        secretKeyRef:
          name: myapp-secrets
          key: database-url
```

### External Secrets (AWS Secrets Manager)
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: SecretStore
    name: aws-secrets
  target:
    name: myapp-secrets
  data:
  - secretKey: database-url
    remoteRef:
      key: myapp/database
      property: url
```

## Resource Management

### Resource Requests & Limits
```yaml
resources:
  requests:
    cpu: 100m      # 0.1 CPU cores
    memory: 128Mi  # 128 MiB
  limits:
    cpu: 500m      # 0.5 CPU cores
    memory: 512Mi  # 512 MiB
```

### Horizontal Pod Autoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Pod Disruption Budget
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2  # Or maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

## Health Checks

### Probes
```yaml
spec:
  containers:
  - name: myapp
    # Readiness: Is the pod ready to receive traffic?
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
    
    # Liveness: Is the pod alive? (Restart if fails)
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
      failureThreshold: 3
    
    # Startup: Is the pod starting up? (Disables other probes)
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

## Jobs & CronJobs

### One-time Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:v1.0.0
        command: ["./migrate", "up"]
      restartPolicy: OnFailure
```

### CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["./backup.sh"]
          restartPolicy: OnFailure
```

## Troubleshooting

### Pod Debugging
```bash
# Check pod status
kubectl get pods -l app=myapp
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>  # Multi-container
kubectl logs <pod-name> --previous  # Previous instance

# Interactive shell
kubectl exec -it <pod-name> -- /bin/sh

# Port forward for local access
kubectl port-forward <pod-name> 8080:8080

# Copy files
kubectl cp <pod-name>:/path/to/file ./local-file
```

### Common Issues

**Pod stuck in Pending:**
```bash
kubectl describe pod <pod-name>
# Look for:
# - Insufficient resources
# - Node selector/affinity issues
# - PVC not bound
```

**Pod in CrashLoopBackOff:**
```bash
kubectl logs <pod-name> --previous
# Check:
# - Application startup errors
# - Missing config/secrets
# - Health check failures
```

**Service not reachable:**
```bash
# Check endpoints
kubectl get endpoints <service-name>

# Test from another pod
kubectl run debug --rm -it --image=busybox -- wget -qO- http://service-name
```

### Resource Usage
```bash
# Node resources
kubectl top nodes

# Pod resources
kubectl top pods

# Detailed resource usage
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
```

## Common Patterns

### Init Containers
```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
  containers:
  - name: myapp
    image: myapp:v1.0.0
```

### Sidecar Container
```yaml
spec:
  containers:
  - name: myapp
    image: myapp:v1.0.0
  - name: log-shipper
    image: fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir: {}
```

### Pod Affinity/Anti-Affinity
```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname
```

## kubectl Cheatsheet

```bash
# Apply manifests
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/

# Get resources
kubectl get pods,svc,deploy
kubectl get all -n namespace

# Watch changes
kubectl get pods -w

# Scale deployment
kubectl scale deployment myapp --replicas=5

# Rollout
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp
kubectl rollout restart deployment/myapp

# Delete resources
kubectl delete -f deployment.yaml
kubectl delete pod <pod-name>

# Context switching
kubectl config get-contexts
kubectl config use-context <context-name>
```

## References

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubectl Cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Patterns Book](https://k8spatterns.io/)
- [Production Best Practices](https://learnk8s.io/production-best-practices)
