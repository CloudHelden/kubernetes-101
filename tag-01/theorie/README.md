# Tag 1: Einführung in Kubernetes

## Lernziele

- Verstehen, was Kubernetes ist und welche Probleme es löst
- Die Kubernetes-Architektur kennenlernen
- Grundlegende Komponenten identifizieren können
- Einen lokalen Cluster aufsetzen

---

## 1. Was ist Kubernetes?

Kubernetes (K8s) ist eine Open-Source-Plattform zur **Automatisierung der Bereitstellung, Skalierung und Verwaltung von containerisierten Anwendungen**.

### Warum Kubernetes?

| Problem | Lösung durch Kubernetes |
|---------|------------------------|
| Container manuell starten/stoppen | Automatische Orchestrierung |
| Ausfälle von Containern | Self-Healing (automatischer Neustart) |
| Lastverteilung | Integriertes Load Balancing |
| Updates ohne Downtime | Rolling Updates |
| Skalierung bei Last | Horizontale Auto-Skalierung |

### Geschichte

- 2014: Von Google als Open Source veröffentlicht
- Basiert auf Googles internem System "Borg"
- Heute verwaltet von der CNCF (Cloud Native Computing Foundation)

---

## 2. Kubernetes-Architektur

```
┌─────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    CONTROL PLANE                          │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐    │   │
│  │  │ API Server │ │ Scheduler  │ │ Controller Manager │    │   │
│  │  └────────────┘ └────────────┘ └────────────────────┘    │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │                      etcd                           │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    WORKER NODES                           │   │
│  │  ┌─────────────────┐    ┌─────────────────┐              │   │
│  │  │     Node 1      │    │     Node 2      │              │   │
│  │  │  ┌───────────┐  │    │  ┌───────────┐  │              │   │
│  │  │  │  kubelet  │  │    │  │  kubelet  │  │              │   │
│  │  │  ├───────────┤  │    │  ├───────────┤  │              │   │
│  │  │  │kube-proxy │  │    │  │kube-proxy │  │              │   │
│  │  │  ├───────────┤  │    │  ├───────────┤  │              │   │
│  │  │  │ Container │  │    │  │ Container │  │              │   │
│  │  │  │  Runtime  │  │    │  │  Runtime  │  │              │   │
│  │  │  └───────────┘  │    │  └───────────┘  │              │   │
│  │  │  ┌────┐ ┌────┐  │    │  ┌────┐ ┌────┐  │              │   │
│  │  │  │Pod │ │Pod │  │    │  │Pod │ │Pod │  │              │   │
│  │  │  └────┘ └────┘  │    │  └────┘ └────┘  │              │   │
│  │  └─────────────────┘    └─────────────────┘              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Control Plane Komponenten

### API Server (kube-apiserver)

- **Zentrale Schnittstelle** für alle Interaktionen
- RESTful API
- Authentifizierung & Autorisierung
- Alle kubectl-Befehle gehen hierher

### etcd

- **Verteilte Key-Value-Datenbank**
- Speichert den gesamten Cluster-Zustand
- Hochverfügbar und konsistent
- "Single Source of Truth"

> ⚠️ **Produktions-Hinweis:** etcd ist standardmäßig **nicht verschlüsselt**!
> Für Produktion: Encryption at Rest aktivieren und regelmäßige Backups einrichten.
> etcd-Ausfall = Cluster-Ausfall → immer als Cluster (3+ Nodes) betreiben.

### Scheduler (kube-scheduler)

- **Entscheidet, auf welchem Node ein Pod läuft**
- Berücksichtigt: Ressourcen, Constraints, Affinität
- Platziert neue Pods auf passenden Nodes

### Controller Manager

- **Führt Controller-Loops aus**
- Überwacht den Ist-Zustand
- Stellt den Soll-Zustand her
- Beispiele: Node Controller, Deployment Controller, StatefulSet Controller, Job Controller

---

## 4. Node Komponenten

### kubelet

- **Agent auf jedem Node**
- Kommuniziert mit dem API Server
- Startet/Stoppt Container gemäß Pod-Spezifikation
- Meldet Node-Status zurück

### kube-proxy

- **Netzwerk-Proxy auf jedem Node**
- Implementiert Service-Konzept
- Leitet Traffic zu den richtigen Pods

### Container Runtime

- **Führt Container aus**
- Standard heute: **containerd** (in den meisten Distributionen)
- Alternativen: CRI-O, Docker (via cri-dockerd)
- Kommuniziert über CRI (Container Runtime Interface)

> 📝 **Info:** Docker-Shim wurde in K8s 1.24 (2022) entfernt. Docker funktioniert
> weiterhin als Build-Tool, containerd ist aber die empfohlene Runtime.

---

## 5. Grundlegende Kubernetes-Objekte

### Pod

- Kleinste deploybare Einheit
- Ein oder mehrere Container
- Teilen Netzwerk und Storage

### Node

- Eine Maschine (physisch oder virtuell)
- Führt Pods aus
- Wird vom Control Plane verwaltet

### Namespace

- Logische Trennung von Ressourcen
- Für Multi-Tenancy
- Standard: default, kube-system, kube-public

### Label & Selector

- Key-Value-Paare zur Organisation
- Ermöglichen Gruppierung und Selektion

---

## 6. Deklarativ vs. Imperativ

### Imperativ (WIE)

```bash
kubectl run nginx --image=nginx
kubectl expose pod nginx --port=80
kubectl scale deployment nginx --replicas=3
```

### Deklarativ (WAS) - Empfohlen!

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f pod.yaml
```

**Vorteile deklarativ:**
- Versionierbar (Git)
- Reproduzierbar
- Review-fähig
- Infrastructure as Code

---

## 7. YAML-Grundstruktur

Jede Kubernetes-Ressource hat diese Grundstruktur:

```yaml
apiVersion: v1              # API-Version
kind: Pod                   # Ressourcen-Typ
metadata:                   # Metadaten
  name: mein-pod           # Name (erforderlich)
  namespace: default       # Namespace (optional)
  labels:                  # Labels (optional)
    app: web
spec:                      # Spezifikation (variiert je nach Kind)
  containers:
  - name: container-1
    image: nginx:latest
```

---

## Zusammenfassung Tag 1

1. **Kubernetes** orchestriert Container automatisch
2. **Control Plane** = Gehirn des Clusters (API Server, etcd, Scheduler, Controller)
3. **Worker Nodes** = führen die eigentliche Arbeit aus (kubelet, kube-proxy, Runtime)
4. **Pod** = kleinste Einheit, enthält Container
5. **Deklarativ** mit YAML ist der empfohlene Ansatz
