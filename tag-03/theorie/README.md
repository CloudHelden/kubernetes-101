# Tag 3: Workloads - Deployments, ReplicaSets & mehr

## Lernziele

- Deployments verstehen und einsetzen
- ReplicaSets und deren Beziehung zu Deployments
- Rolling Updates und Rollbacks durchführen
- Weitere Workload-Controller kennenlernen

---

## 1. Warum nicht nur Pods?

### Probleme mit einzelnen Pods

- **Kein Neustart** bei Node-Ausfall
- **Keine Skalierung** möglich
- **Kein Update-Mechanismus**
- **Manuelle Verwaltung** nötig

### Lösung: Controller

Controller überwachen den Ist-Zustand und stellen den Soll-Zustand her.

```
┌─────────────┐     überwacht     ┌─────────────┐
│  Controller │ ─────────────────▶│    Pods     │
└─────────────┘                   └─────────────┘
       │                                 ▲
       │ erstellt/löscht/aktualisiert   │
       └─────────────────────────────────┘
```

---

## 2. ReplicaSet

### Was macht ein ReplicaSet?

- Stellt sicher, dass **X Pods** laufen
- Ersetzt ausgefallene Pods
- Nutzt Label-Selector zur Zuordnung

### ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:              # Pod-Template
    metadata:
      labels:
        app: nginx       # MUSS zu selector passen!
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
```

### Wichtig

- **Selector** muss zu **Pod-Labels** passen
- ReplicaSets werden selten direkt erstellt
- → Besser: **Deployment** nutzen

---

## 3. Deployment

### Das wichtigste Workload-Objekt

```
┌─────────────────────────────────────────────┐
│                 Deployment                   │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │            ReplicaSet               │    │
│  │                                     │    │
│  │   ┌─────┐   ┌─────┐   ┌─────┐      │    │
│  │   │ Pod │   │ Pod │   │ Pod │      │    │
│  │   └─────┘   └─────┘   └─────┘      │    │
│  │                                     │    │
│  └─────────────────────────────────────┘    │
│                                             │
└─────────────────────────────────────────────┘
```

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
```

### Deployment Vorteile

| Feature | Beschreibung |
|---------|--------------|
| **Skalierung** | Replicas erhöhen/verringern |
| **Rolling Updates** | Schrittweise Updates ohne Downtime |
| **Rollback** | Zurück zur vorherigen Version |
| **History** | Revisions-Geschichte |
| **Pause/Resume** | Updates pausieren |

---

## 4. Rolling Updates

### Strategie-Konfiguration

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max zusätzliche Pods
      maxUnavailable: 0  # Max nicht verfügbare Pods
```

### Update-Ablauf

```
Soll: 3 Pods mit nginx:1.25 (aktuell: nginx:1.24)

Schritt 1: [1.24] [1.24] [1.24] [1.25]  ← neuer Pod
Schritt 2: [1.24] [1.24] [1.25]         ← alter Pod weg
Schritt 3: [1.24] [1.24] [1.25] [1.25]  ← neuer Pod
Schritt 4: [1.24] [1.25] [1.25]         ← alter Pod weg
Schritt 5: [1.24] [1.25] [1.25] [1.25]  ← neuer Pod
Schritt 6: [1.25] [1.25] [1.25]         ← fertig!
```

### Update-Befehle

```bash
# Image aktualisieren
kubectl set image deployment/nginx-deployment nginx=nginx:1.25

# Status verfolgen
kubectl rollout status deployment/nginx-deployment

# History anzeigen
kubectl rollout history deployment/nginx-deployment

# Rollback
kubectl rollout undo deployment/nginx-deployment

# Zu bestimmter Revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

---

## 5. Skalierung

### Manuell

```bash
# Imperativ
kubectl scale deployment nginx-deployment --replicas=5

# Deklarativ (YAML ändern)
spec:
  replicas: 5
```

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
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

```bash
# Kurzform
kubectl autoscale deployment nginx-deployment --min=2 --max=10 --cpu-percent=50
```

---

## 6. Weitere Workload-Typen

### DaemonSet

**Ein Pod pro Node**

Anwendungsfälle:
- Log-Collector auf jedem Node
- Monitoring-Agent
- Network-Plugin

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: fluentd
        image: fluentd
```

### StatefulSet

**Für zustandsbehaftete Anwendungen**

Eigenschaften:
- Stabile Netzwerk-Identität (pod-0, pod-1, ...)
- Geordnetes Deployment/Skalierung
- Persistenter Storage pro Pod

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql       # Muss auf Headless Service zeigen!
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

> ⚠️ **Wichtig:** StatefulSets benötigen einen **Headless Service** (`clusterIP: None`)
> für die DNS-Auflösung der einzelnen Pods (mysql-0.mysql, mysql-1.mysql, etc.)

### Job

**Einmalige Aufgabe**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool
        command: ["./backup.sh"]
      restartPolicy: Never
  backoffLimit: 4
```

### CronJob

**Geplante, wiederkehrende Aufgaben**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"  # Cron-Syntax
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool
          restartPolicy: OnFailure
```

---

## 7. Übersicht Workload-Typen

| Typ | Anwendungsfall | Beispiel |
|-----|----------------|----------|
| **Deployment** | Stateless Apps | Web-Server, APIs |
| **StatefulSet** | Stateful Apps | Datenbanken |
| **DaemonSet** | Auf jedem Node | Log-Agents |
| **Job** | Einmalige Tasks | Migrationen |
| **CronJob** | Geplante Tasks | Backups |

---

## 8. Labels und Selektoren

### Labels setzen

```yaml
metadata:
  labels:
    app: webshop
    environment: production
    team: backend
    version: v1.2.3
```

### Selektoren

```yaml
# Equality-based
selector:
  matchLabels:
    app: webshop
    environment: production

# Set-based
selector:
  matchExpressions:
  - key: environment
    operator: In
    values: [production, staging]
  - key: team
    operator: NotIn
    values: [legacy]
```

### kubectl mit Labels

```bash
# Pods mit bestimmtem Label
kubectl get pods -l app=webshop

# Mehrere Labels (AND)
kubectl get pods -l app=webshop,environment=production

# Alle Labels anzeigen
kubectl get pods --show-labels
```

---

## Zusammenfassung Tag 3

1. **Deployment** = Standard für stateless Anwendungen
2. **ReplicaSet** = stellt Replica-Anzahl sicher (von Deployment verwaltet)
3. **Rolling Updates** = Updates ohne Downtime
4. **Rollback** = jederzeit zur vorherigen Version
5. **DaemonSet** = ein Pod pro Node
6. **StatefulSet** = für Datenbanken und zustandsbehaftete Apps
7. **Job/CronJob** = einmalige oder geplante Aufgaben
