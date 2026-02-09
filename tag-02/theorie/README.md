# Tag 2: Pods & Container

## Lernziele

- Den Pod-Lifecycle verstehen
- Multi-Container Pods erstellen können
- Container-Ressourcen definieren
- Pods debuggen können
- Init-Container verstehen

---

## 1. Der Pod im Detail

### Was ist ein Pod?

- **Kleinste deploybare Einheit** in Kubernetes
- **Wrapper um einen oder mehrere Container**
- Container in einem Pod teilen:
  - Netzwerk (gleiche IP-Adresse)
  - Storage (gemeinsame Volumes)
  - Lifecycle (starten/stoppen zusammen)

### Wann mehrere Container in einem Pod?

**Ja, wenn:**
- Container eng gekoppelt sind
- Sie Daten über lokale Dateien teilen
- Sidecar-Pattern (Logging, Proxy, etc.)

**Nein, wenn:**
- Container unabhängig skalieren sollen
- Container unterschiedliche Lifecycles haben
- Keine enge Kopplung besteht

---

## 2. Pod-Lifecycle

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌───────────┐
│ Pending │───▶│ Running │───▶│Succeeded│ or │  Failed   │
└─────────┘    └─────────┘    └───────────────┴───────────┘
     │              │
     │              ▼
     │         ┌─────────┐
     └────────▶│ Unknown │
               └─────────┘
```

### Pod-Phasen

| Phase | Beschreibung |
|-------|--------------|
| **Pending** | Pod akzeptiert, Container-Images werden geladen |
| **Running** | Mindestens ein Container läuft |
| **Succeeded** | Alle Container erfolgreich beendet (Exit 0) |
| **Failed** | Mindestens ein Container mit Fehler beendet |
| **Unknown** | Status kann nicht ermittelt werden |

### Container-Zustände

| Zustand | Beschreibung |
|---------|--------------|
| **Waiting** | Container startet noch nicht (z.B. Image-Pull) |
| **Running** | Container wird ausgeführt |
| **Terminated** | Container wurde beendet |

---

## 3. Restart-Policies

```yaml
spec:
  restartPolicy: Always  # Standardwert
```

| Policy | Verhalten |
|--------|-----------|
| **Always** | Container wird immer neu gestartet |
| **OnFailure** | Nur bei Exit-Code != 0 |
| **Never** | Nie neu starten |

---

## 4. Ressourcen-Management

### Requests vs. Limits

```yaml
spec:
  containers:
  - name: app
    image: myapp
    resources:
      requests:        # Garantierte Ressourcen
        memory: "128Mi"
        cpu: "250m"
      limits:          # Maximale Ressourcen
        memory: "256Mi"
        cpu: "500m"
```

### CPU-Einheiten

- `1` = 1 vCPU/Core
- `500m` = 0.5 vCPU (m = Milli)
- `250m` = 0.25 vCPU

### Memory-Einheiten

- `Ki` = Kibibyte (1024 Bytes)
- `Mi` = Mebibyte (1024 Ki)
- `Gi` = Gibibyte (1024 Mi)

### Was passiert bei Überschreitung?

| Ressource | Überschreitung von Limit |
|-----------|-------------------------|
| **CPU** | Throttling (gedrosselt) |
| **Memory** | OOMKilled (Prozess beendet) |

---

## 5. Multi-Container Patterns

### Sidecar Pattern

Ein zusätzlicher Container, der den Hauptcontainer unterstützt.

```
┌─────────────────────────────────┐
│              Pod                │
│  ┌───────────┐  ┌────────────┐  │
│  │   App     │  │  Log       │  │
│  │ Container │──│  Shipper   │  │
│  └───────────┘  └────────────┘  │
│        │              ▲         │
│        └──────────────┘         │
│         (shared volume)         │
└─────────────────────────────────┘
```

Beispiel: Log-Collector, Monitoring-Agent

### Ambassador Pattern

Ein Proxy-Container für externe Kommunikation.

```
┌─────────────────────────────────┐
│              Pod                │
│  ┌───────────┐  ┌────────────┐  │
│  │   App     │──│  Proxy     │──────▶ External
│  │ Container │  │ Container  │  │      Service
│  └───────────┘  └────────────┘  │
└─────────────────────────────────┘
```

Beispiel: Database-Proxy, API-Gateway

### Adapter Pattern

Ein Container, der Daten transformiert.

```
┌─────────────────────────────────┐
│              Pod                │
│  ┌───────────┐  ┌────────────┐  │
│  │   App     │──│  Adapter   │──────▶ Monitoring
│  │ Container │  │ Container  │  │      System
│  └───────────┘  └────────────┘  │
└─────────────────────────────────┘
```

Beispiel: Prometheus-Exporter, Format-Converter

---

## 6. Init-Container

Container, die **vor** den App-Containern laufen.

### Anwendungsfälle

- Warten auf Abhängigkeiten (Datenbank)
- Konfigurationsdateien generieren
- Datenbank-Migrationen
- Secrets von externen Quellen laden

### Eigenschaften

- Laufen sequentiell (einer nach dem anderen)
- Müssen erfolgreich beenden (Exit 0)
- Erst dann starten App-Container

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']

  containers:
  - name: app
    image: myapp
```

---

## 7. Probes (Gesundheitschecks)

### Liveness Probe

"Ist der Container noch am Leben?"

- Bei Fehlschlag: Container wird neu gestartet
- Erkennt Deadlocks, eingefrorene Prozesse

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
```

### Readiness Probe

"Ist der Container bereit für Traffic?"

- Bei Fehlschlag: Pod wird aus Service-Endpoints entfernt
- Kein Traffic mehr zum Pod

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Startup Probe

"Ist die Anwendung gestartet?"

- Für langsam startende Anwendungen
- Deaktiviert Liveness/Readiness während Startup

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

### Probe-Methoden

| Methode | Beschreibung |
|---------|--------------|
| `httpGet` | HTTP-Request, Erfolg bei 200-399 |
| `tcpSocket` | TCP-Verbindung zum Port |
| `exec` | Befehl im Container, Erfolg bei Exit 0 |
| `grpc` | gRPC Health-Check |

---

## 8. Pod-Debugging

### Wichtige Befehle

```bash
# Pod-Status
kubectl get pods
kubectl describe pod <name>

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>  # Multi-Container
kubectl logs <pod-name> --previous           # Vorheriger Container
kubectl logs <pod-name> -f                   # Follow (live)

# In Container verbinden
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -c <container> -- /bin/bash

# Events anzeigen
kubectl get events --sort-by='.lastTimestamp'

# Pod-YAML mit aktuellem Status
kubectl get pod <name> -o yaml
```

### Häufige Probleme

| Symptom | Mögliche Ursache |
|---------|------------------|
| ImagePullBackOff | Image nicht gefunden, falscher Name |
| CrashLoopBackOff | Container stürzt wiederholt ab |
| Pending | Keine Ressourcen, Node-Selector passt nicht |
| OOMKilled | Memory-Limit überschritten |

---

## Zusammenfassung Tag 2

1. **Pod** = Gruppe von Containern mit geteiltem Netzwerk/Storage
2. **Lifecycle**: Pending → Running → Succeeded/Failed
3. **Ressourcen**: Requests (garantiert) vs. Limits (maximum)
4. **Multi-Container Patterns**: Sidecar, Ambassador, Adapter
5. **Init-Container**: Laufen vor App-Containern
6. **Probes**: Liveness, Readiness, Startup für Gesundheitschecks
