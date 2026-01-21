# Kubernetes (2025)

> **Last updated**: January 2026
> **Versions covered**: Kubernetes 1.29+
> **Purpose**: Container orchestration with security and scalability

---

## Philosophy (2025-2026)

Kubernetes is now the **default platform** — 87% of organizations deploy in hybrid-cloud environments (CNCF). In 2025, the focus is on **security**, **cost optimization**, and **GitOps**. 60% of containers live less than one minute.

**Key shifts:**
- **Policy engines from day one** — OPA Gatekeeper or Kyverno
- **Gateway API over Ingress** — More powerful routing
- **External Secrets Operators** — No secrets in Git
- **Right-sizing** — 40% CPU, 57% memory over-provisioned on average
- **DevSecOps integration** — Security in CI/CD, not afterthought

---

## TL;DR

- Use namespaces for isolation
- Define resource requests AND limits
- Enable RBAC with least privilege
- Use Network Policies (Calico/Cilium)
- Scan images in CI/CD (Trivy)
- Use policy engines (Kyverno/OPA)
- Externalize secrets (External Secrets Operator)
- Use Gateway API for ingress
- Monitor with Prometheus + Grafana

---

## Best Practices

### Project Structure (GitOps)

```
k8s/
├── base/                       # Base configurations
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── kustomization.yaml
├── overlays/                   # Environment-specific
│   ├── development/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   └── production/
│       ├── kustomization.yaml
│       ├── patches/
│       └── replicas-patch.yaml
├── policies/                   # Security policies
│   ├── network-policies/
│   ├── kyverno/
│   └── resource-quotas/
└── helm/                       # Helm charts
    └── my-app/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

### Secure Deployment Manifest

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app.kubernetes.io/name: my-app
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: my-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      # Security context at pod level
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      # Service account
      serviceAccountName: my-app
      automountServiceAccountToken: false

      containers:
        - name: my-app
          image: ghcr.io/myorg/my-app:1.0.0@sha256:abc123...
          imagePullPolicy: Always

          # Container security context
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP

          # Resource management
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi

          # Health checks
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 30

          # Environment variables
          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "8080"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name

          envFrom:
            - configMapRef:
                name: my-app-config
            - secretRef:
                name: my-app-secrets

          # Writable directories
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /app/.cache

      volumes:
        - name: tmp
          emptyDir: {}
        - name: cache
          emptyDir: {}

      # Scheduling
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: my-app
```

### Service

```yaml
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app.kubernetes.io/name: my-app
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
    - name: metrics
      port: 9090
      targetPort: metrics
      protocol: TCP
  selector:
    app.kubernetes.io/name: my-app
```

### Gateway API (Modern Ingress)

```yaml
# Gateway (cluster-level)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: gateway-system
spec:
  gatewayClassName: cilium
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-cert
      allowedRoutes:
        namespaces:
          from: All

---
# HTTPRoute (per-application)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: my-app
spec:
  parentRefs:
    - name: main-gateway
      namespace: gateway-system
  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/v1
      backendRefs:
        - name: my-app
          port: 80
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: X-Request-ID
                value: "%REQ_ID%"
```

### Horizontal Pod Autoscaler

```yaml
# base/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 20
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
```

### Network Policy

```yaml
# policies/network-policies/my-app.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-app
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: my-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow from Gateway
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: gateway-system
      ports:
        - protocol: TCP
          port: 8080
    # Allow Prometheus scraping
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9090
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # Allow database access
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: database
          podSelector:
            matchLabels:
              app.kubernetes.io/name: postgres
      ports:
        - protocol: TCP
          port: 5432
```

### Kyverno Policies

```yaml
# policies/kyverno/require-labels.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-app-labels
      match:
        resources:
          kinds:
            - Deployment
            - StatefulSet
      validate:
        message: "Labels 'app.kubernetes.io/name' and 'app.kubernetes.io/version' are required"
        pattern:
          metadata:
            labels:
              app.kubernetes.io/name: "?*"
              app.kubernetes.io/version: "?*"

---
# policies/kyverno/disallow-privileged.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-privileged-containers
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Privileged containers are not allowed"
        pattern:
          spec:
            containers:
              - securityContext:
                  privileged: "false"

---
# policies/kyverno/require-requests-limits.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-requests-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-cpu-memory
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "CPU and memory requests/limits are required"
        pattern:
          spec:
            containers:
              - resources:
                  requests:
                    memory: "?*"
                    cpu: "?*"
                  limits:
                    memory: "?*"
```

### External Secrets

```yaml
# External Secrets Operator configuration
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: my-app-secrets
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: my-app/production
        property: database_url
    - secretKey: API_KEY
      remoteRef:
        key: my-app/production
        property: api_key
```

### RBAC

```yaml
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: my-app

---
# Role for app-specific permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app
  namespace: my-app
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["my-app-secrets"]
    verbs: ["get"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app
  namespace: my-app
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: my-app
roleRef:
  kind: Role
  name: my-app
  apiGroup: rbac.authorization.k8s.io
```

### Resource Quotas

```yaml
# policies/resource-quotas/my-app.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: my-app
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    persistentvolumeclaims: "10"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: my-app
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: 2
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi
```

### Kustomization

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - hpa.yaml
commonLabels:
  app.kubernetes.io/managed-by: kustomize

---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: my-app-prod
namePrefix: prod-
patches:
  - path: replicas-patch.yaml
  - path: resources-patch.yaml
configMapGenerator:
  - name: my-app-config
    literals:
      - LOG_LEVEL=info
      - CACHE_TTL=3600
images:
  - name: ghcr.io/myorg/my-app
    newTag: v1.2.3
```

### Helm Values

```yaml
# helm/my-app/values.yaml
replicaCount: 3

image:
  repository: ghcr.io/myorg/my-app
  tag: "1.0.0"
  pullPolicy: Always

serviceAccount:
  create: true
  name: my-app

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

nodeSelector: {}

tolerations: []

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: my-app
          topologyKey: kubernetes.io/hostname

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

---

## Anti-Patterns

### ❌ No Resource Limits

```yaml
# ❌ DON'T — No limits
containers:
  - name: app
    image: my-app:latest

# ✅ DO — Always set requests and limits
containers:
  - name: app
    image: my-app:1.0.0
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

### ❌ Running as Root

```yaml
# ❌ DON'T — Default runs as root
containers:
  - name: app
    image: my-app

# ✅ DO — Non-root with security context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
containers:
  - name: app
    image: my-app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
```

### ❌ Using :latest Tag

```yaml
# ❌ DON'T — Mutable, unpredictable
image: my-app:latest

# ✅ DO — Immutable with digest
image: my-app:1.0.0@sha256:abc123...
```

### ❌ Secrets in Git

```yaml
# ❌ DON'T — Secrets in manifests
apiVersion: v1
kind: Secret
data:
  password: cGFzc3dvcmQxMjM=  # Base64 is not encryption!

# ✅ DO — External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
spec:
  secretStoreRef:
    name: vault
```

### ❌ No Network Policies

```yaml
# ❌ DON'T — All pods can talk to all pods

# ✅ DO — Explicit network policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
spec:
  policyTypes: [Ingress, Egress]
  # Define allowed traffic explicitly
```

---

## Quick Reference

| Resource | Purpose |
|----------|---------|
| Deployment | Stateless apps |
| StatefulSet | Stateful apps |
| DaemonSet | Per-node workloads |
| Job/CronJob | Batch workloads |
| ConfigMap | Configuration |
| Secret | Sensitive data |
| Service | Internal networking |
| Ingress/Gateway | External access |
| NetworkPolicy | Traffic control |
| HPA | Auto-scaling |

| Command | Purpose |
|---------|---------|
| `kubectl apply -k overlays/prod/` | Apply Kustomize |
| `kubectl get pods -n my-app` | List pods |
| `kubectl logs -f deploy/my-app` | Stream logs |
| `kubectl exec -it pod/my-app -- sh` | Shell access |
| `kubectl top pods -n my-app` | Resource usage |
| `kubectl rollout restart deploy/my-app` | Rolling restart |

| Tool | Purpose |
|------|---------|
| Kustomize | Configuration management |
| Helm | Package management |
| ArgoCD/Flux | GitOps |
| Kyverno/OPA | Policy engine |
| Trivy | Image scanning |
| Prometheus | Monitoring |
| Cilium/Calico | CNI + Network Policy |

---

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/overview/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [Kyverno Policies](https://kyverno.io/policies/)
- [External Secrets Operator](https://external-secrets.io/)
- [Kubernetes Best Practices 2025](https://komodor.com/learn/14-kubernetes-best-practices-you-must-know-in-2025/)
