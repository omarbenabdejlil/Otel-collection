# OpenTelemetry sur Kubernetes — Guide de déploiement complet

> Stack : OTel Operator + Collector (Agent DaemonSet + Gateway Deployment) + Prometheus + Loki + Jaeger + Grafana
>
> Testé sur : Docker Desktop Kubernetes, OTel Operator 0.146.0
>
> Date : Mars 2026

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  App A (ns: app-frontend)   App B (ns: app-backend)   App C (ns: app-worker)  │
│  annotation: inject-nodejs  annotation: inject-java   annotation: inject-python │
│         │ OTel SDK               │ OTel SDK                │ OTel SDK           │
│         └──── OTLP/gRPC ────────┼──── OTLP/gRPC ──────────┘                    │
│                                  ▼                                               │
│                    ┌─────────────────────────┐                                   │
│                    │  OTel Agent (DaemonSet)  │  ← 1 par nœud                    │
│                    │  + k8sattributes         │  ← enrichit namespace/pod/node   │
│                    │  + filelog receiver       │  ← collecte logs containers     │
│                    └────────┬────────────────┘                                   │
│                             │ OTLP/gRPC                                          │
│                             ▼                                                    │
│              ┌──────────────────────────────┐                                    │
│              │  OTel Gateway (Deployment)    │  ← centralisé, ns: observability  │
│              │  + resource processor         │                                    │
│              │  + batch processor            │                                    │
│              └───┬──────────┬───────────┬───┘                                    │
│                  │          │           │                                         │
│                  ▼          ▼           ▼                                         │
│           Prometheus     Jaeger       Loki                                       │
│           (remote_write) (OTLP/gRPC)  (OTLP/HTTP)                               │
│                  └──────────┼───────────┘                                        │
│                             ▼                                                    │
│                          Grafana                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Prérequis

- Kubernetes cluster fonctionnel
- `kubectl` et `helm` installés
- Accès admin au cluster

---

## Étape 1 — Créer les namespaces

```bash
kubectl create namespace observability
kubectl create namespace app-frontend
kubectl create namespace app-backend
kubectl create namespace app-worker
```

---

## Étape 2 — Installer l'opérateur OpenTelemetry

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace observability \
  --set admissionWebhooks.certManager.enabled=false \
  --set manager.collectorImage.repository=otel/opentelemetry-collector-contrib
```

Vérifier :

```bash
kubectl get pods -n observability
# Attendre que opentelemetry-operator soit 2/2 Running
```

---

## Étape 3 — Créer le RBAC (ServiceAccount + ClusterRole)

> **IMPORTANT** : Le ServiceAccount doit être créé AVANT tout déploiement de Collector.
> L'opérateur peut purger les ServiceAccounts qu'il considère "unmanaged".
> En le créant manuellement ici et en le référençant dans les manifests, on évite les erreurs
> `FailedCreate: serviceaccount "otel-collector" not found`.

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector
  namespace: observability
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces", "nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["replicasets", "deployments"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector
subjects:
  - kind: ServiceAccount
    name: otel-collector
    namespace: observability
EOF
```

Vérifier :

```bash
kubectl get sa -n observability | grep otel-collector
```

---

## Étape 4 — Déployer le Collector Agent (DaemonSet via opérateur)

L'Agent tourne sur chaque nœud. Il reçoit les signaux OTLP des apps et collecte les logs des containers via `filelog`.

```bash
kubectl apply -f - <<'EOF'
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-agent
  namespace: observability
spec:
  mode: daemonset
  serviceAccount: otel-collector
  hostNetwork: true
  volumes:
    - name: varlogpods
      hostPath:
        path: /var/log/pods
  volumeMounts:
    - name: varlogpods
      mountPath: /var/log/pods
      readOnly: true
  env:
    - name: K8S_NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      filelog:
        include:
          - /var/log/pods/*/*/*.log
        exclude:
          - /var/log/pods/observability_otel-*/*/*.log
        operators:
          - type: router
            routes:
              - output: parse_json
                expr: 'body matches "^\\{"'
              - output: parse_raw
                expr: "true"
          - id: parse_json
            type: json_parser
            timestamp:
              parse_from: attributes.time
              layout: "%Y-%m-%dT%H:%M:%S.%LZ"
          - id: parse_raw
            type: move
            from: body
            to: attributes.log
    processors:
      batch:
        timeout: 5s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_mib: 400
        spike_limit_mib: 100
      k8sattributes:
        auth_type: "serviceAccount"
        passthrough: false
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.pod.name
            - k8s.pod.uid
            - k8s.deployment.name
            - k8s.node.name
          labels:
            - tag_name: app.label
              key: app
              from: pod
        pod_association:
          - sources:
              - from: resource_attribute
                name: k8s.pod.ip
    exporters:
      otlp/gateway:
        endpoint: "otel-gateway.observability.svc.cluster.local:4317"
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [otlp/gateway]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [otlp/gateway]
        logs:
          receivers: [otlp, filelog]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [otlp/gateway]
EOF
```

Vérifier :

```bash
kubectl get pods -n observability -l app.kubernetes.io/name=otel-agent-collector
# Doit être 1/1 Running
```

---

## Étape 5 — Déployer le Gateway Collector (Deployment manuel)

> **IMPORTANT** : On déploie le Gateway comme un Deployment Kubernetes classique
> (ConfigMap + Deployment + Service) plutôt que via la CRD `OpenTelemetryCollector`.
> L'opérateur a tendance à purger les ressources qu'il considère "unmanaged"
> (notamment le ServiceAccount), ce qui empêche la création des pods.
> Un Deployment standard est plus fiable et plus simple à debugger.

> **NOTE** : L'exporter `loki` n'existe pas dans le collector-contrib.
> Pour envoyer des logs à Loki, utiliser `otlphttp` avec l'endpoint `/otlp`
> (Loki supporte l'ingestion OTLP nativement depuis v2.9+).

On commence en mode `debug` (les backends ne sont pas encore déployés) :

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-gateway-config
  namespace: observability
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch:
        timeout: 5s
        send_batch_size: 1024
      resource:
        attributes:
          - key: cluster.name
            value: "valentine-prod"
            action: upsert
    exporters:
      debug:
        verbosity: basic
    service:
      pipelines:
        metrics:
          receivers: [otlp]
          processors: [resource, batch]
          exporters: [debug]
        traces:
          receivers: [otlp]
          processors: [resource, batch]
          exporters: [debug]
        logs:
          receivers: [otlp]
          processors: [resource, batch]
          exporters: [debug]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-gateway
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-gateway
  template:
    metadata:
      labels:
        app: otel-gateway
    spec:
      serviceAccountName: otel-collector
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:0.146.0
          args: ["--config=/conf/config.yaml"]
          ports:
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
          volumeMounts:
            - name: config
              mountPath: /conf
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: config
          configMap:
            name: otel-gateway-config
---
apiVersion: v1
kind: Service
metadata:
  name: otel-gateway
  namespace: observability
spec:
  selector:
    app: otel-gateway
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
EOF
```

Vérifier :

```bash
kubectl get pods -n observability -l app=otel-gateway
# Doit être 1/1 Running

kubectl logs -n observability deployment/otel-gateway --tail=5
# Doit afficher "Everything is ready. Begin running and processing data."
```

---

## Étape 6 — Installer les backends (Prometheus, Loki, Jaeger, Grafana)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

# Prometheus (sans node-exporter — crash sur Docker Desktop)
helm install prometheus prometheus-community/prometheus \
  -n observability \
  --set server.enableRemoteWriteReceiver=true \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.enabled=false \
  --set prometheus-node-exporter.enabled=false

# Loki (mode single binary pour dev/test)
helm install loki grafana/loki \
  -n observability \
  --set loki.auth_enabled=false \
  --set loki.commonConfig.replication_factor=1 \
  --set singleBinary.replicas=1 \
  --set loki.storage.type=filesystem \
  --set singleBinary.persistence.enabled=false \
  --set deploymentMode=SingleBinary \
  --set chunksCache.enabled=false \
  --set resultsCache.enabled=false \
  --set gateway.enabled=false \
  --set read.replicas=0 \
  --set write.replicas=0 \
  --set backend.replicas=0

# Jaeger (all-in-one en mémoire pour dev/test)
helm install jaeger jaegertracing/jaeger \
  -n observability \
  --set allInOne.enabled=true \
  --set provisionDataStore.cassandra=false \
  --set storage.type=memory \
  --set collector.enabled=false \
  --set query.enabled=false \
  --set agent.enabled=false

# Grafana
helm install grafana grafana/grafana \
  -n observability \
  --set adminPassword=admin \
  --set persistence.enabled=false
```

Vérifier :

```bash
kubectl get pods -n observability
# Attendre que prometheus-server, loki, jaeger et grafana soient Running
```

---

## Étape 7 — Brancher le Gateway sur les vrais backends

Maintenant que les backends sont up, on met à jour le ConfigMap du Gateway :

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-gateway-config
  namespace: observability
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch:
        timeout: 5s
        send_batch_size: 1024
      resource:
        attributes:
          - key: cluster.name
            value: "valentine-prod"
            action: upsert
    exporters:
      debug:
        verbosity: basic
      prometheusremotewrite:
        endpoint: "http://prometheus-server.observability.svc:9090/api/v1/write"
        resource_to_telemetry_conversion:
          enabled: true
      otlp/jaeger:
        endpoint: "jaeger.observability.svc:4317"
        tls:
          insecure: true
      otlphttp/loki:
        endpoint: "http://loki.observability.svc:3100/otlp"
    service:
      pipelines:
        metrics:
          receivers: [otlp]
          processors: [resource, batch]
          exporters: [prometheusremotewrite, debug]
        traces:
          receivers: [otlp]
          processors: [resource, batch]
          exporters: [otlp/jaeger, debug]
        logs:
          receivers: [otlp]
          processors: [resource, batch]
          exporters: [otlphttp/loki, debug]
EOF

# Redémarrer pour prendre la nouvelle config
kubectl rollout restart deployment/otel-gateway -n observability
```

---

## Étape 8 — Créer les Instrumentations (auto-injection par namespace)

Une `Instrumentation` par namespace. L'opérateur injectera automatiquement le SDK OTel dans les pods annotés.

```bash
kubectl apply -f - <<'EOF'
# App Frontend — Node.js
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation
  namespace: app-frontend
spec:
  exporter:
    endpoint: http://otel-agent-collector.observability.svc:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "1"
  nodejs:
    env:
      - name: OTEL_SERVICE_NAME
        value: "app-frontend"
      - name: OTEL_LOGS_EXPORTER
        value: "otlp"
---
# App Backend — Java
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation
  namespace: app-backend
spec:
  exporter:
    endpoint: http://otel-agent-collector.observability.svc:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "0.5"
  java:
    env:
      - name: OTEL_SERVICE_NAME
        value: "app-backend"
      - name: OTEL_LOGS_EXPORTER
        value: "otlp"
---
# App Worker — Python
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation
  namespace: app-worker
spec:
  exporter:
    endpoint: http://otel-agent-collector.observability.svc:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
  python:
    env:
      - name: OTEL_SERVICE_NAME
        value: "app-worker"
      - name: OTEL_LOGS_EXPORTER
        value: "otlp"
EOF
```

---

## Étape 9 — Déployer les 3 applications avec annotations d'injection

> L'annotation `instrumentation.opentelemetry.io/inject-<langage>: "true"` déclenche
> l'injection automatique du SDK par l'opérateur. Aucune modification du code applicatif
> n'est nécessaire.

> **IMPORTANT** : Chaque Deployment DOIT avoir `spec.selector.matchLabels` qui correspond
> exactement à `spec.template.metadata.labels`. C'est obligatoire en `apps/v1`.

```bash
kubectl apply -f - <<'EOF'
# --- App A : Frontend Node.js ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: app-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
      annotations:
        instrumentation.opentelemetry.io/inject-nodejs: "true"
    spec:
      containers:
        - name: frontend
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: app-frontend
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
---
# --- App B : Backend Java ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: app-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
        - name: backend
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: app-backend
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
---
# --- App C : Worker Python ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
  namespace: app-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
      annotations:
        instrumentation.opentelemetry.io/inject-python: "true"
    spec:
      containers:
        - name: worker
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: worker
  namespace: app-worker
spec:
  selector:
    app: worker
  ports:
    - port: 80
      targetPort: 80
EOF
```

> **Note** : Les images `nginx:alpine` sont des placeholders pour tester.
> En production, remplace par tes vraies images applicatives.

---

## Étape 10 — Configurer Grafana avec les datasources

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: observability
  labels:
    grafana_datasource: "1"
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-server.observability.svc:80
        isDefault: true
        access: proxy
      - name: Jaeger
        type: jaeger
        url: http://jaeger-query.observability.svc:16686
        access: proxy
      - name: Loki
        type: loki
        url: http://loki.observability.svc:3100
        access: proxy
EOF

kubectl rollout restart deployment/grafana -n observability
```

Accéder à Grafana :

```bash
kubectl port-forward svc/grafana 3000:80 -n observability
# Ouvrir http://localhost:3000
# Login : admin / admin
```

---

## Vérification finale

```bash
# Tous les pods doivent être Running
kubectl get pods -A

# Vérifier l'injection OTel sur les apps
kubectl describe pod -n app-frontend -l app=frontend | grep -A5 "Init Containers"
# Doit montrer un init-container "opentelemetry-auto-instrumentation"

# Vérifier les env vars injectées
kubectl exec -n app-frontend deploy/frontend -- env | grep OTEL

# Vérifier que le Gateway reçoit des données
kubectl logs -n observability deployment/otel-gateway --tail=20

# Vérifier les backends
kubectl logs -n observability deployment/prometheus-server --tail=10
kubectl logs -n observability deployment/grafana --tail=10
```

---

## Exemples de requêtes Grafana

**Métriques (Prometheus)** :

```promql
# Toutes les métriques HTTP de l'app frontend
http_server_duration_milliseconds_bucket{k8s_namespace_name="app-frontend"}

# Taux de requêtes par service
rate(http_server_request_count_total[5m])
```

**Traces (Jaeger)** :

Filtrer par `service.name = app-frontend` ou `service.name = app-backend`

**Logs (Loki)** :

```logql
# Logs du backend avec filtre erreur
{k8s_namespace_name="app-backend"} |= "error"

# Tous les logs du worker
{k8s_namespace_name="app-worker"}
```

---

## Pièges rencontrés et solutions

| Problème | Cause | Solution |
|----------|-------|----------|
| Gateway pods `FailedCreate: serviceaccount not found` | L'opérateur purge les SA qu'il considère "unmanaged" | Créer le SA manuellement AVANT les Collectors (étape 3) |
| Gateway CRD `READY 0/0`, pas de pods créés | Bug de l'opérateur qui ne reconcilie pas après crash | Utiliser un Deployment manuel au lieu de la CRD pour le Gateway |
| `unknown type: "loki" for id: "loki"` | L'exporter `loki` n'existe pas dans collector-contrib | Utiliser `otlphttp` avec endpoint `http://loki:3100/otlp` |
| `prometheus-node-exporter` CrashLoopBackOff | Docker Desktop ne fournit pas les sys/proc du host | `--set prometheus-node-exporter.enabled=false` |
| Deployment créé sans `spec.selector` | Obligatoire en `apps/v1` | Toujours inclure `spec.selector.matchLabels` |
| Opérateur bloqué sur leader election | Ancien lease non libéré après restart | `kubectl delete lease <name> -n observability` ou attendre ~30s |
| Apps en `ErrImagePull` | Images privées non accessibles en local | Utiliser `nginx:alpine` comme placeholder pour les tests |

---

## Ordre de déploiement (résumé)

1. Namespaces
2. Helm : opérateur OpenTelemetry → attendre Running
3. RBAC : ServiceAccount + ClusterRole + ClusterRoleBinding
4. Collector Agent (DaemonSet via CRD opérateur)
5. Collector Gateway (Deployment manuel + ConfigMap + Service) — d'abord en mode `debug`
6. Helm : Prometheus, Loki, Jaeger, Grafana → attendre Running
7. Mettre à jour le ConfigMap du Gateway avec les vrais exporters + restart
8. Instrumentations (une par namespace)
9. Applications avec annotations d'injection
10. Datasources Grafana
