# Pro-Tipps für Kubernetes

Gesammelte Tipps von erfahrenen Kubernetes-Administratoren.

---

## Allgemein

### Kubernetes-Version beachten
```
Dieser Kurs basiert auf Kubernetes 1.29+ (Stand 2026).
Einige Features unterscheiden sich in älteren Versionen.
Prüfe immer: kubectl version
```

### kubectl Aliases einrichten
```bash
# In ~/.bashrc oder ~/.zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kga='kubectl get all'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias kx='kubectl exec -it'

# Namespace-Wechsel
alias kns='kubectl config set-context --current --namespace'
```

### Context-Management
```bash
# Aktuellen Context anzeigen
kubectl config current-context

# Alle Contexts auflisten
kubectl config get-contexts

# Context wechseln
kubectl config use-context production-cluster

# Namespace für aktuellen Context setzen
kubectl config set-context --current --namespace=my-namespace
```

---

## Tag 1: Architektur

### API Server ist das Nadelöhr
Der API Server ist die meistbelastete Komponente. Bei Performance-Problemen:
```bash
# API Server Metriken prüfen
kubectl get --raw /metrics | grep apiserver_request

# Langsame Requests identifizieren
kubectl get --raw /metrics | grep apiserver_request_duration
```

### etcd Backup - KRITISCH!
```bash
# etcd Snapshot erstellen (auf Control Plane Node)
ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Backup verifizieren
ETCDCTL_API=3 etcdctl snapshot status backup.db
```

**Regel:** Tägliche Backups, vor jedem Cluster-Upgrade!

### Label-Strategie von Anfang an planen
```yaml
# Empfohlene Labels (Kubernetes Standard)
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: webshop
    app.kubernetes.io/managed-by: helm
```

### Namespace-Isolation verstehen
```
WICHTIG: Namespaces bieten KEINE Netzwerk-Isolation!
Pods in verschiedenen Namespaces können sich standardmäßig erreichen.

Für echte Isolation:
→ Network Policies verwenden
→ Oder: Separate Cluster für kritische Workloads
```

---

## Tag 2: Pods & Container

### Ressourcen IMMER setzen
```yaml
# NIEMALS ohne Ressourcen deployen!
resources:
  requests:          # Garantiert, für Scheduling
    cpu: "100m"      # 0.1 CPU
    memory: "128Mi"
  limits:            # Maximum, wird enforced
    cpu: "500m"
    memory: "256Mi"
```

**Faustregel:**
- `requests` = typische Nutzung
- `limits` = 1.5-2x von requests
- CPU-Limit optional (Throttling statt Kill)
- Memory-Limit IMMER setzen (OOMKill!)

### QoS-Klassen verstehen
```
Kubernetes vergibt automatisch QoS-Klassen:

1. Guaranteed (höchste Priorität):
   - requests == limits für CPU UND Memory
   - Wird zuletzt evicted

2. Burstable:
   - requests < limits
   - Mittlere Priorität

3. BestEffort (niedrigste Priorität):
   - Keine requests/limits gesetzt
   - Wird zuerst evicted bei Node-Pressure!

→ Produktions-Workloads: Guaranteed oder Burstable
→ NIEMALS BestEffort in Produktion!
```

### Probe-Timeouts richtig setzen
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15    # Zeit zum Starten
  periodSeconds: 10          # Wie oft prüfen
  timeoutSeconds: 3          # Max Wartezeit
  failureThreshold: 3        # Wie oft fehlschlagen darf
  successThreshold: 1        # Wie oft OK sein muss

# TIPP: Startup Probe für langsam startende Apps
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30       # 30 x 10s = 5 Minuten zum Starten
  periodSeconds: 10
```

**Wichtig:** Startup Probe deaktiviert Liveness/Readiness während des Starts!

### Debugging ohne Pod-Änderung
```bash
# Ephemeral Container für Debugging (K8s 1.25+)
kubectl debug -it pod/myapp --image=busybox --target=myapp

# Debug-Copy erstellen
kubectl debug pod/myapp -it --copy-to=myapp-debug --container=myapp -- sh

# Vorherige Container-Logs
kubectl logs pod/myapp --previous
```

### CrashLoopBackOff verstehen
```
Backoff-Zeiten: 10s → 20s → 40s → 80s → ... → max 300s

Häufige Ursachen:
1. Fehlende ConfigMap/Secret
2. Falsche Command/Args
3. Fehlende Berechtigungen
4. Port bereits belegt
5. Dependency nicht erreichbar

Debug-Schritte:
kubectl describe pod <name>     # Events prüfen
kubectl logs <name> --previous  # Crash-Logs
kubectl get events --sort-by='.lastTimestamp'
```

---

## Tag 3: Workloads

### Rolling Update Strategie wählen
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    # Zero-Downtime (langsam, sicher):
    maxSurge: 1
    maxUnavailable: 0

    # Schnelles Update (kurze Unterbrechung OK):
    # maxSurge: 25%
    # maxUnavailable: 25%

    # Minimum-Ressourcen (alte Pods erst weg):
    # maxSurge: 0
    # maxUnavailable: 1
```

### progressDeadlineSeconds setzen
```yaml
spec:
  progressDeadlineSeconds: 600  # 10 Minuten für Rollout
  # Default: 600
  # Bei Timeout: Deployment wird als "failed" markiert
  # Aber: Rollout läuft weiter!
```

### StatefulSet richtig konfigurieren
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql  # MUSS Headless Service sein!
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  # WICHTIG: volumeClaimTemplates für persistenten Storage
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### DaemonSet auf Tainted Nodes
```yaml
# Für System-DaemonSets (z.B. auf Master-Nodes)
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
```

### CronJob Timezone (K8s 1.25+)
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"
  timeZone: "Europe/Berlin"  # NEU in K8s 1.25+
  concurrencyPolicy: Forbid  # Verhindert parallele Ausführung
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool
          restartPolicy: OnFailure
```

### Pod Disruption Budget für HA
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2          # ODER: maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp

# Schützt vor:
# - Node Drains
# - Cluster Upgrades
# - Unbeabsichtigtem Löschen
```

---

## Tag 4: Networking

### DNS-Auflösung verstehen
```bash
# Voller DNS-Name (FQDN):
<service>.<namespace>.svc.cluster.local

# Im GLEICHEN Namespace reicht:
curl http://my-service

# In ANDEREM Namespace - FQDN nötig:
curl http://my-service.other-namespace.svc.cluster.local

# NICHT: http://my-service.other-namespace (funktioniert nicht!)
```

### Headless Service für StatefulSets
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # ← Headless!
  selector:
    app: mysql
  ports:
  - port: 3306

# DNS gibt alle Pod-IPs zurück:
# mysql-0.mysql.default.svc.cluster.local
# mysql-1.mysql.default.svc.cluster.local
# mysql-2.mysql.default.svc.cluster.local
```

### Service Debug-Checkliste
```bash
# 1. Service existiert?
kubectl get svc my-service

# 2. Endpoints vorhanden?
kubectl get endpoints my-service
# LEER? → Selector stimmt nicht mit Pod-Labels!

# 3. Pods haben richtige Labels?
kubectl get pods --show-labels

# 4. DNS funktioniert?
kubectl run test --rm -it --image=busybox -- nslookup my-service

# 5. Verbindung möglich?
kubectl run test --rm -it --image=busybox -- wget -qO- http://my-service
```

### Gateway API statt Ingress (Modern)
```yaml
# Gateway API ist der moderne Ersatz für Ingress (K8s 1.26+)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: myapp-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "myapp.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 80
```

### LoadBalancer in On-Prem Umgebungen
```
Problem: LoadBalancer type zeigt <pending> für External IP

Lösungen:
1. MetalLB installieren (empfohlen für On-Prem)
2. NodePort verwenden
3. Ingress Controller mit NodePort

# MetalLB Layer 2 Mode (einfach):
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

---

## Tag 5: Storage & Konfiguration

### ConfigMap vs. Secret Entscheidung
```
ConfigMap verwenden für:
✓ Konfigurationsdateien
✓ Umgebungsvariablen (nicht-sensitiv)
✓ Feature Flags
✓ Verbindungsparameter (ohne Credentials)

Secret verwenden für:
✓ Passwörter
✓ API Keys
✓ TLS Zertifikate
✓ SSH Keys
✓ OAuth Tokens

MERKE: Secret = Base64 encoded, NICHT verschlüsselt!
→ Encryption at Rest aktivieren!
```

### Secret-Sicherheit
```yaml
# WARNUNG: Secrets als Env-Vars sind sichtbar!
# ps aux zeigt alle Umgebungsvariablen
# /proc/<pid>/environ enthält alle Env-Vars

# BESSER: Als Volume mounten
volumeMounts:
- name: secrets
  mountPath: /etc/secrets
  readOnly: true

volumes:
- name: secrets
  secret:
    secretName: my-secret
    defaultMode: 0400  # Nur lesbar für Owner
```

### ConfigMap/Secret Auto-Update
```
Volume-Mount: Auto-Update nach ~60 Sekunden ✓
Environment Variable: KEIN Auto-Update! Neustart nötig ✗

Für Auto-Update-fähige Configs:
→ Als Volume mounten
→ App muss File-Changes erkennen (inotify oder Polling)
```

### ReadWriteOnce (RWO) verstehen
```
WICHTIG: RWO bedeutet "ein NODE", nicht "ein Pod"!

RWO PVC kann von mehreren Pods genutzt werden,
ABER alle Pods müssen auf dem GLEICHEN Node laufen.

Pods auf verschiedenen Nodes → FEHLER!

Für Multi-Node-Zugriff:
→ ReadWriteMany (RWX) verwenden
→ Benötigt NFS, CephFS, etc.
```

### PVC Debugging
```bash
# PVC stuck in Pending?

# 1. Events prüfen
kubectl describe pvc my-pvc

# 2. StorageClass existiert?
kubectl get storageclass

# 3. Provisioner läuft?
kubectl get pods -n kube-system | grep provisioner

# 4. Ausreichend Kapazität?
kubectl describe pv  # Bei statischem Provisioning

# 5. Binding Mode prüfen
# WaitForFirstConsumer: PV wird erst bei Pod-Scheduling gebunden
```

### External Secrets für Produktion
```yaml
# Für echte Produktion: External Secrets Operator
# Synchronisiert Secrets von:
# - HashiCorp Vault
# - AWS Secrets Manager
# - Azure Key Vault
# - Google Secret Manager

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/db
      property: password
```

---

## Tag 6: Security

### RBAC Debugging
```bash
# Kann User X Aktion Y ausführen?
kubectl auth can-i delete pods --as=jane
kubectl auth can-i create deployments --as=system:serviceaccount:default:myapp

# Alle Berechtigungen eines Users anzeigen
kubectl auth can-i --list --as=jane

# Alle RoleBindings für einen User finden
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.subjects[]?.name=="jane")'
```

### Security Context Minimum
```yaml
# Minimale sichere Konfiguration für jeden Pod:
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
    # Für schreibbare Verzeichnisse: emptyDir
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

### Network Policy - CNI Prüfen!
```bash
# KRITISCH: NetworkPolicies funktionieren NUR mit unterstütztem CNI!

# Unterstützt:
# ✓ Calico
# ✓ Cilium
# ✓ Weave Net

# NICHT unterstützt:
# ✗ Flannel (Standard in vielen Setups!)
# ✗ kubenet

# Prüfen welches CNI aktiv ist:
kubectl get pods -n kube-system | grep -E "calico|cilium|weave|flannel"
```

### Pod Security Standards (PSS) Migration
```yaml
# Schritt 1: Warn Mode (zeigt nur Warnungen)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest

# Schritt 2: Audit Mode (loggt Verstöße)
    pod-security.kubernetes.io/audit: restricted

# Schritt 3: Enforce Mode (blockiert Verstöße)
    pod-security.kubernetes.io/enforce: restricted
```

### Image Security 2026
```yaml
# Signierte Images mit Cosign verifizieren
# In Kyverno Policy:
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-signature
    match:
      resources:
        kinds:
        - Pod
    verifyImages:
    - imageReferences:
      - "myregistry.io/*"
      attestors:
      - entries:
        - keyless:
            issuer: "https://accounts.google.com"
            subject: "ci@mycompany.com"
```

### Audit Logging aktivieren
```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Secrets: Alle Zugriffe loggen
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]

# RBAC Änderungen: Vollständig loggen
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]

# Alles andere: Nur Metadata
- level: Metadata
  omitStages:
  - RequestReceived
```

---

## Troubleshooting Quick Reference

### Pod startet nicht
```bash
kubectl describe pod <name>     # Events am Ende prüfen
kubectl get events --sort-by='.lastTimestamp'
kubectl logs <pod> --previous   # Logs vom letzten Crash
```

### Service nicht erreichbar
```bash
kubectl get endpoints <svc>     # Leer? Selector prüfen!
kubectl get pods --show-labels  # Labels korrekt?
kubectl run test --rm -it --image=busybox -- wget -qO- http://<svc>
```

### PVC bleibt Pending
```bash
kubectl describe pvc <name>     # Events prüfen
kubectl get sc                  # StorageClass existiert?
kubectl get pv                  # PV verfügbar?
```

### RBAC Fehler
```bash
kubectl auth can-i <verb> <resource> --as=<user>
kubectl describe role/clusterrole <name>
kubectl get rolebinding -A | grep <name>
```

---

## Weiterführende Ressourcen

- [Kubernetes Dokumentation](https://kubernetes.io/docs/)
- [CNCF Landscape](https://landscape.cncf.io/)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [CKA/CKAD Exam Tips](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)
