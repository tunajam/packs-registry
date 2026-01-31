# tailscale-kubernetes

Tailscale Kubernetes operator, ingress, sidecars, and cluster networking

## Source

Tailscale Kubernetes documentation: https://tailscale.com/kb/1185/kubernetes

## Overview

Tailscale integrates with Kubernetes in several ways:
- **Kubernetes Operator** - Managed ingress, egress, and API server access
- **Sidecar** - Per-pod Tailscale connectivity
- **Proxy** - Expose services to tailnet
- **Subnet Router** - Access entire cluster network

## Kubernetes Operator

The recommended approach for production Kubernetes deployments.

### Installation

```bash
# Add Tailscale Helm repo
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update

# Install operator
helm upgrade --install tailscale-operator tailscale/tailscale-operator \
  --namespace tailscale \
  --create-namespace \
  --set oauth.clientId="<oauth-client-id>" \
  --set oauth.clientSecret="<oauth-client-secret>"
```

### OAuth Client Setup
1. Go to Admin Console → Settings → OAuth Clients
2. Create new OAuth client with scopes: `devices`, `routes`
3. Use client ID and secret in Helm install

### Ingress (Expose Services)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    # Use Tailscale as ingress
    kubernetes.io/ingress.class: tailscale
spec:
  rules:
    - host: my-app  # becomes my-app.tail12345.ts.net
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

### Egress (Access Tailnet from Pods)

```yaml
apiVersion: tailscale.com/v1alpha1
kind: Connector
metadata:
  name: my-connector
spec:
  tags:
    - tag:k8s
  hostname: k8s-cluster
  routes:
    - 100.64.0.0/10  # Tailnet CGNAT range
```

### API Server Proxy

Access kube-apiserver over Tailscale:

```yaml
apiVersion: tailscale.com/v1alpha1
kind: ProxyClass
metadata:
  name: api-server
spec:
  statefulSet:
    pod:
      annotations:
        tailscale.com/expose: "true"
```

## Sidecar Pattern

Add Tailscale to individual pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-tailscale
spec:
  containers:
    - name: nginx
      image: nginx
    - name: tailscale
      image: tailscale/tailscale:stable
      securityContext:
        capabilities:
          add:
            - NET_ADMIN
      env:
        - name: TS_AUTHKEY
          valueFrom:
            secretKeyRef:
              name: tailscale-auth
              key: TS_AUTHKEY
        - name: TS_KUBE_SECRET
          value: "tailscale-state"
        - name: TS_HOSTNAME
          value: "nginx-pod"
        - name: TS_USERSPACE
          value: "false"
      volumeMounts:
        - name: tun
          mountPath: /dev/net/tun
  volumes:
    - name: tun
      hostPath:
        path: /dev/net/tun
```

### Userspace Sidecar (No Privileges)

```yaml
containers:
  - name: tailscale
    image: tailscale/tailscale:stable
    env:
      - name: TS_AUTHKEY
        valueFrom:
          secretKeyRef:
            name: tailscale-auth
            key: TS_AUTHKEY
      - name: TS_USERSPACE
        value: "true"
      - name: TS_SOCKS5_SERVER
        value: ":1055"
    # No special capabilities needed
```

App uses SOCKS5 proxy at `localhost:1055`.

## Service Proxy

Expose a Kubernetes Service to tailnet:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    tailscale.com/expose: "true"
    tailscale.com/hostname: "my-k8s-service"
spec:
  selector:
    app: my-app
  ports:
    - port: 80
```

## Subnet Router

Access entire cluster network over Tailscale:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tailscale-router
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tailscale-router
  template:
    metadata:
      labels:
        app: tailscale-router
    spec:
      containers:
        - name: tailscale
          image: tailscale/tailscale:stable
          securityContext:
            capabilities:
              add:
                - NET_ADMIN
          env:
            - name: TS_AUTHKEY
              valueFrom:
                secretKeyRef:
                  name: tailscale-auth
                  key: TS_AUTHKEY
            - name: TS_ROUTES
              value: "10.96.0.0/12,10.244.0.0/16"  # Service + Pod CIDRs
            - name: TS_EXTRA_ARGS
              value: "--advertise-tags=tag:k8s-router"
            - name: TS_KUBE_SECRET
              value: "tailscale-state"
          volumeMounts:
            - name: tun
              mountPath: /dev/net/tun
      volumes:
        - name: tun
          hostPath:
            path: /dev/net/tun
```

Find your cluster CIDRs:
```bash
# Pod CIDR
kubectl cluster-info dump | grep -m 1 cluster-cidr

# Service CIDR  
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range
```

## RBAC for State

Tailscale stores state in Kubernetes secrets:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tailscale
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tailscale
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create", "get", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tailscale
subjects:
  - kind: ServiceAccount
    name: tailscale
roleRef:
  kind: Role
  name: tailscale
  apiGroup: rbac.authorization.k8s.io
```

## Auth Key Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-auth
type: Opaque
stringData:
  TS_AUTHKEY: "tskey-auth-xxxxx"
```

For ephemeral pods:
```yaml
stringData:
  TS_AUTHKEY: "tskey-auth-xxxxx?ephemeral=true"
```

## MagicDNS in Pods

Enable DNS resolution for tailnet names:

```yaml
env:
  - name: TS_ACCEPT_DNS
    value: "true"
```

## Common Patterns

### Internal Service Access
```yaml
# Expose internal service to tailnet
apiVersion: v1
kind: Service
metadata:
  name: internal-api
  annotations:
    tailscale.com/expose: "true"
    tailscale.com/hostname: "k8s-api"
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - port: 8080
```

### Secure Database Access
```yaml
# Tailscale sidecar for database pod
# Only tailnet members can connect
```

### CI/CD Access
```yaml
env:
  - name: TS_AUTHKEY
    value: "tskey-auth-xxx?ephemeral=true"
# Pod auto-removed from tailnet when deleted
```

## Operator CRDs

### Connector
Access tailnet resources from cluster:
```yaml
apiVersion: tailscale.com/v1alpha1
kind: Connector
metadata:
  name: tailnet-access
spec:
  hostname: k8s-egress
  tags:
    - tag:k8s
```

### ProxyClass
Customize how services are exposed:
```yaml
apiVersion: tailscale.com/v1alpha1
kind: ProxyClass
metadata:
  name: high-memory
spec:
  statefulSet:
    pod:
      resources:
        limits:
          memory: 512Mi
```

## Troubleshooting

### Pod Can't Authenticate
```bash
# Check secret exists
kubectl get secret tailscale-auth -o yaml

# Check auth key validity (keys expire)
# Generate new key in Admin Console
```

### Routes Not Working
```bash
# Check routes are approved in Admin Console
# Verify ACLs allow access to routes
# Check operator logs
kubectl logs -n tailscale -l app=operator
```

### Can't Reach Services
```bash
# Check service annotation
kubectl get svc <name> -o yaml | grep tailscale

# Check tailscale pod status
kubectl exec <tailscale-pod> -- tailscale status
```

## Best Practices

1. **Use Operator for production** - More maintainable than manual sidecars
2. **Ephemeral keys for CI/CD** - Auto-cleanup of temporary pods
3. **Tag your pods** - Use `--advertise-tags` for ACL management
4. **Secure auth keys** - Store in Secrets, consider external secrets
5. **Monitor state** - Ensure state Secret persists
6. **Plan CIDRs** - Document Pod/Service CIDRs for subnet routing

## Reference

- Kubernetes Operator: https://tailscale.com/kb/1236/kubernetes-operator
- Ingress: https://tailscale.com/kb/1439/kubernetes-operator-cluster-ingress
- Subnet routing: https://tailscale.com/kb/1441/kubernetes-operator-connector
