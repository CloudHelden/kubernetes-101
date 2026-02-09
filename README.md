# Kubernetes 101 - Einführungskurs

Ein 6-tägiger praxisorientierter Kurs zu den Grundlagen von Kubernetes.

## Kursübersicht

| Tag | Thema | Vormittag (3h) | Nachmittag (5h) |
|-----|-------|----------------|-----------------|
| 1 | Einführung & Architektur | Container-Grundlagen, K8s-Architektur | Minikube Installation, erste Schritte |
| 2 | Pods & Container | Pod-Lifecycle, Multi-Container Pods | Pod-Übungen, Debugging |
| 3 | Workloads | Deployments, ReplicaSets, DaemonSets | Rolling Updates, Skalierung |
| 4 | Networking | Services, Ingress, DNS | Service-Typen, Load Balancing |
| 5 | Storage & Konfiguration | Volumes, ConfigMaps, Secrets | Persistente Anwendungen |
| 6 | Security & Best Practices | RBAC, Network Policies, Security Context | Abschlussprojekt |

## Voraussetzungen

- Grundlegende Linux-Kenntnisse (Bash, Navigation)
- Docker-Grundlagen (Images, Container)
- Ein Laptop mit mindestens 8 GB RAM
- Texteditor (VS Code empfohlen)

## Setup

### Option 1: Minikube (Empfohlen für diesen Kurs)

```bash
# macOS
brew install minikube kubectl

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Cluster starten
minikube start --cpus=2 --memory=4096
```

### Option 2: kind (Kubernetes in Docker)

```bash
# Installation
brew install kind kubectl  # macOS
# oder
go install sigs.k8s.io/kind@latest  # mit Go

# Cluster erstellen
kind create cluster --name k8s-kurs
```

### Option 3: Docker Desktop

Docker Desktop enthält eine eingebaute Kubernetes-Option, die in den Einstellungen aktiviert werden kann.

## Struktur des Repositories

```
kubernetes-101/
├── README.md
├── tag-01/                 # Tag 1: Einführung & Architektur
│   ├── theorie/
│   ├── uebungen/
│   └── loesungen/
├── tag-02/                 # Tag 2: Pods & Container
├── tag-03/                 # Tag 3: Workloads
├── tag-04/                 # Tag 4: Networking
├── tag-05/                 # Tag 5: Storage & Konfiguration
├── tag-06/                 # Tag 6: Security & Best Practices
└── beispiele/              # Wiederverwendbare YAML-Beispiele
```

## Wichtige kubectl-Befehle

```bash
# Cluster-Info
kubectl cluster-info
kubectl get nodes

# Ressourcen anzeigen
kubectl get pods
kubectl get services
kubectl get deployments

# Ressourcen erstellen/löschen
kubectl apply -f <datei.yaml>
kubectl delete -f <datei.yaml>

# Debugging
kubectl describe pod <name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/sh
```

## Hilfreiche Links

- [Offizielle Kubernetes Dokumentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [CNCF Landscape](https://landscape.cncf.io/)

## Lizenz

Dieses Material ist für Bildungszwecke erstellt.
