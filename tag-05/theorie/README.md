# Tag 5: Storage & Konfiguration

## Lernziele

- Kubernetes Storage-Konzepte verstehen
- Verschiedene Volume-Typen kennen
- ConfigMaps für Konfiguration nutzen
- Secrets für sensible Daten verwenden
- Persistent Volumes und Claims

---

## 1. Das Storage-Problem

### Container sind ephemer

```
Pod startet → Daten werden geschrieben → Pod stirbt → Daten weg!
```

### Lösungen

| Anforderung | Lösung |
|-------------|--------|
| Daten zwischen Container teilen | emptyDir |
| Daten über Pod-Lebenszyklus | Persistent Volume |
| Konfiguration | ConfigMap |
| Sensible Daten | Secret |

---

## 2. Volume-Typen Übersicht

### Temporäre Volumes

| Typ | Beschreibung | Lebensdauer |
|-----|--------------|-------------|
| `emptyDir` | Leeres Verzeichnis | Pod-Lebensdauer |
| `configMap` | Konfigurationsdaten | Unabhängig |
| `secret` | Sensible Daten | Unabhängig |

### Persistente Volumes

| Typ | Beschreibung |
|-----|--------------|
| `hostPath` | Verzeichnis auf dem Node |
| `nfs` | Network File System |
| `awsElasticBlockStore` | AWS EBS |
| `gcePersistentDisk` | Google Cloud Disk |
| `azureDisk` | Azure Managed Disk |

---

## 3. emptyDir

### Anwendungsfälle

- Zwischenspeicher für Berechnungen
- Daten zwischen Containern im Pod teilen
- Checkpoints für lange Prozesse

### Beispiel

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ["/bin/sh", "-c", "echo 'Hello' > /data/file.txt; sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data

  - name: reader
    image: busybox
    command: ["/bin/sh", "-c", "cat /data/file.txt; sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data

  volumes:
  - name: shared-data
    emptyDir: {}
```

### emptyDir mit RAM (tmpfs)

```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory
    sizeLimit: 100Mi
```

---

## 4. ConfigMaps

### Was sind ConfigMaps?

- **Konfigurationsdaten** von Anwendung trennen
- Key-Value-Paare oder ganze Dateien
- Nicht für sensible Daten (dafür: Secrets)

### ConfigMap erstellen

**Imperativ:**

```bash
# Aus Literal
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2

# Aus Datei
kubectl create configmap my-config --from-file=config.properties

# Aus Verzeichnis
kubectl create configmap my-config --from-file=./config-dir/
```

**Deklarativ:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Einfache Key-Values
  database_host: "db.example.com"
  database_port: "5432"
  log_level: "info"

  # Ganze Datei
  app.properties: |
    server.port=8080
    server.host=0.0.0.0
    feature.enabled=true
```

### ConfigMap verwenden

**Als Umgebungsvariablen:**

```yaml
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host

    # Alle Keys als Env-Vars
    envFrom:
    - configMapRef:
        name: app-config
```

**Als Volume:**

```yaml
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config

  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

---

## 5. Secrets

### Was sind Secrets?

- Für **sensible Daten** (Passwörter, API-Keys, Zertifikate)
- Base64-kodiert (nicht verschlüsselt!)
- Kubernetes schützt Secrets besser als ConfigMaps

> ⚠️ **KRITISCHE SICHERHEITSWARNUNG:**
>
> Base64 ist **Encoding, KEINE Verschlüsselung!** Jeder mit `kubectl get secret`
> Berechtigung kann Secrets sofort im Klartext lesen:
> ```bash
> kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
> ```
>
> **Für Produktion IMMER:**
> - Encryption at Rest aktivieren (`--encryption-provider-config`)
> - RBAC für Secret-Zugriff einschränken
> - External Secrets Manager verwenden (Vault, AWS Secrets Manager, etc.)

### Secret-Typen

| Typ | Verwendung |
|-----|------------|
| `Opaque` | Allgemeine Secrets |
| `kubernetes.io/tls` | TLS-Zertifikate |
| `kubernetes.io/dockerconfigjson` | Docker Registry Auth |
| `kubernetes.io/basic-auth` | Basic Authentication |

### Secret erstellen

**Imperativ:**

```bash
# Generic Secret
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=geheim123

# TLS Secret
kubectl create secret tls my-tls \
  --cert=tls.crt \
  --key=tls.key

# Docker Registry
kubectl create secret docker-registry my-registry \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```

**Deklarativ:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Base64-kodiert!
  username: YWRtaW4=          # echo -n "admin" | base64
  password: Z2VoZWltMTIz      # echo -n "geheim123" | base64
```

```yaml
# Mit stringData (wird automatisch kodiert)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: geheim123
```

### Secrets verwenden

**Als Umgebungsvariablen:**

```yaml
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

**Als Volume:**

```yaml
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: secrets
    secret:
      secretName: db-secret
```

---

## 6. Persistent Volumes (PV) & Claims (PVC)

### Konzept

```
┌───────────────────────────────────────────────────────┐
│                                                       │
│   Administrator erstellt          User fordert an    │
│                                                       │
│   ┌──────────────────┐          ┌──────────────────┐  │
│   │ Persistent Volume │◀────────│ PersistentVolume │  │
│   │       (PV)       │  bindet  │    Claim (PVC)   │  │
│   └──────────────────┘          └──────────────────┘  │
│          ▲                              │             │
│          │                              │             │
│    Storage (NFS,                  Pod verwendet      │
│    Cloud Disk,                         │             │
│    etc.)                               ▼             │
│                                   ┌─────────┐        │
│                                   │   Pod   │        │
│                                   └─────────┘        │
│                                                       │
└───────────────────────────────────────────────────────┘
```

### Persistent Volume (PV)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /data/pv-demo
```

### Access Modes

| Mode | Beschreibung |
|------|--------------|
| `ReadWriteOnce (RWO)` | Ein **Node** kann lesen/schreiben |
| `ReadOnlyMany (ROX)` | Mehrere Nodes können lesen |
| `ReadWriteMany (RWX)` | Mehrere Nodes können lesen/schreiben |

> ⚠️ **Wichtige Klarstellung zu RWO:**
>
> `ReadWriteOnce` bedeutet **ein Node**, nicht ein Pod!
> - Mehrere Pods auf dem **gleichen Node** können RWO-Volumes teilen
> - Pods auf **verschiedenen Nodes** können RWO-Volumes **NICHT** teilen
>
> Für Multi-Node-Zugriff: `ReadWriteMany` (benötigt NFS, CephFS, etc.)

### Reclaim Policies

| Policy | Beschreibung |
|--------|--------------|
| `Retain` | PV bleibt nach PVC-Löschung |
| `Delete` | PV wird mit PVC gelöscht |
| `Recycle` | Daten werden gelöscht, PV wiederverwendet (deprecated) |

### Persistent Volume Claim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: standard
```

### PVC in Pod verwenden

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html

  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-demo
```

---

## 7. Storage Classes

### Dynamische Provisionierung

Storage Classes ermöglichen **automatische PV-Erstellung**.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### PVC mit StorageClass

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-storage  # PV wird automatisch erstellt!
```

---

## 8. Best Practices

### ConfigMaps & Secrets

- **Niemals** Secrets in Git committen
- ConfigMaps für nicht-sensible Konfiguration
- Secrets für Passwörter, Keys, Zertifikate
- Secrets im Cluster verschlüsseln (Encryption at Rest)

### Storage

- Passenden Access Mode wählen
- Storage Classes für dynamische Provisionierung
- Backups für wichtige Daten
- Reclaim Policy beachten

### Konfigurationsänderungen

ConfigMap/Secret-Änderungen erfordern Pod-Neustart:

```bash
# Rollout Restart
kubectl rollout restart deployment my-deployment
```

Oder mit Hash im Deployment:

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print .Template.BasePath "/configmap.yaml") . | sha256sum }}
```

---

## Zusammenfassung Tag 5

1. **emptyDir** = temporärer Speicher, Pod-Lebensdauer
2. **ConfigMaps** = Konfiguration von Anwendung trennen
3. **Secrets** = sensible Daten (Base64, nicht verschlüsselt!)
4. **PV/PVC** = persistenter Speicher über Pod-Lebenszyklus
5. **Storage Classes** = dynamische Provisionierung
6. **Access Modes**: RWO, ROX, RWX
