# Abschlussprojekt: Gästebuch-Anwendung

## Ziel

Deploye eine vollständige Gästebuch-Anwendung mit Frontend, Backend und Datenbank unter Anwendung aller gelernten Konzepte.

## Architektur

```
┌─────────────────────────────────────────────────────────────┐
│                        Ingress                              │
│                    (guestbook.local)                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Frontend Service                         │
│                      (ClusterIP)                            │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│               Frontend Deployment (3 Replicas)              │
│                     nginx + HTML                            │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Redis Service                           │
│                      (ClusterIP)                            │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                Redis Deployment (1 Replica)                 │
│               + Persistent Volume Claim                     │
└─────────────────────────────────────────────────────────────┘
```

## Anforderungen

### Must-Have
- [ ] Frontend Deployment mit 3 Replicas
- [ ] Redis Deployment mit Persistenz
- [ ] Services für Frontend und Redis
- [ ] ConfigMap für Konfiguration
- [ ] Resource Limits für alle Container
- [ ] Health Checks (Liveness & Readiness)

### Should-Have
- [ ] Ingress mit Host-Routing
- [ ] Eigene ServiceAccounts
- [ ] Security Context (non-root)
- [ ] Network Policies

### Nice-to-Have
- [ ] HorizontalPodAutoscaler
- [ ] TLS-Zertifikat für Ingress
- [ ] Namespace-Isolation

---

## Schritt 1: Namespace erstellen

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: guestbook
  labels:
    app: guestbook
```

```bash
kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=guestbook
```

---

## Schritt 2: Redis Backend

```yaml
# redis.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: guestbook
data:
  redis.conf: |
    maxmemory 64mb
    maxmemory-policy allkeys-lru
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: guestbook
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: guestbook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: data
          mountPath: /data
        - name: config
          mountPath: /usr/local/etc/redis
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: redis-pvc
      - name: config
        configMap:
          name: redis-config
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: guestbook
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

---

## Schritt 3: Frontend

```yaml
# frontend.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: guestbook
data:
  REDIS_HOST: "redis"
  REDIS_PORT: "6379"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: guestbook
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: gcr.io/google-samples/gb-frontend:v5
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: GET_HOSTS_FROM
          value: "env"
        - name: REDIS_SLAVE_SERVICE_HOST
          value: "redis"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: guestbook
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

---

## Schritt 4: Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: guestbook-ingress
  namespace: guestbook
spec:
  rules:
  - host: guestbook.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

---

## Schritt 5: Network Policy (Optional)

```yaml
# networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-policy
  namespace: guestbook
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 6379
```

---

## Deployment

```bash
# Alle Ressourcen anwenden
kubectl apply -f namespace.yaml
kubectl apply -f redis.yaml
kubectl apply -f frontend.yaml
kubectl apply -f ingress.yaml
kubectl apply -f networkpolicy.yaml

# Status prüfen
kubectl get all -n guestbook
kubectl get ingress -n guestbook

# Für lokalen Zugriff (Minikube)
minikube addons enable ingress
echo "$(minikube ip) guestbook.local" | sudo tee -a /etc/hosts

# Oder Port-Forward
kubectl port-forward -n guestbook svc/frontend 8080:80
```

---

## Verifikation

### Checkliste

1. **Pods laufen**
   ```bash
   kubectl get pods -n guestbook
   # Alle Pods sollten Running und Ready sein
   ```

2. **Services erreichbar**
   ```bash
   kubectl get svc -n guestbook
   kubectl run test --rm -it --image=busybox -n guestbook -- wget -qO- http://frontend
   ```

3. **Persistenz funktioniert**
   ```bash
   # Eintrag im Gästebuch machen
   # Redis Pod löschen
   kubectl delete pod -l app=redis -n guestbook
   # Warten bis neuer Pod läuft
   # Prüfen ob Einträge noch da sind
   ```

4. **Skalierung funktioniert**
   ```bash
   kubectl scale deployment frontend --replicas=5 -n guestbook
   kubectl get pods -n guestbook -w
   ```

---

## Erweiterungen

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: guestbook
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Security Context

Füge zu den Deployments hinzu:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: app
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
```

---

## Aufräumen

```bash
kubectl delete namespace guestbook
```

---

## Bewertungskriterien

| Kriterium | Punkte |
|-----------|--------|
| Alle Pods laufen | 10 |
| Services korrekt konfiguriert | 10 |
| Persistenz funktioniert | 10 |
| Resource Limits gesetzt | 10 |
| Health Checks funktionieren | 10 |
| Ingress funktioniert | 10 |
| Security Context | 10 |
| Network Policy | 10 |
| Saubere YAML-Struktur | 10 |
| Dokumentation | 10 |
| **Gesamt** | **100** |
