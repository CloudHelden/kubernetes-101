# Tag 4: Networking - Services, Ingress & DNS

## Lernziele

- Das Kubernetes Netzwerk-Modell verstehen
- Services erstellen und nutzen
- Verschiedene Service-Typen kennen
- Ingress für HTTP-Routing einrichten
- Kubernetes-DNS verstehen

---

## 1. Kubernetes Netzwerk-Modell

### Grundregeln

1. **Jeder Pod hat eine eigene IP**
2. **Alle Pods können sich gegenseitig erreichen** (ohne NAT)
3. **Nodes können alle Pods erreichen** (und umgekehrt)

### Das Problem

```
Pod-IPs sind:
- Temporär (Pod-Neustart = neue IP)
- Dynamisch (Skalierung)
- Unvorhersehbar

→ Wie finden sich Pods?
→ Wie erreicht externer Traffic die Pods?
```

### Die Lösung: Services

```
┌─────────────────────────────────────────────────────┐
│                     Service                         │
│                  (Stabile IP)                       │
│                                                     │
│     ┌─────────────────────────────────────────┐     │
│     │            Load Balancing               │     │
│     └───────────┬───────────┬───────────┬─────┘     │
│                 ▼           ▼           ▼           │
│             ┌─────┐     ┌─────┐     ┌─────┐         │
│             │Pod A│     │Pod B│     │Pod C│         │
│             └─────┘     └─────┘     └─────┘         │
└─────────────────────────────────────────────────────┘
```

---

## 2. Service-Grundlagen

### Was ist ein Service?

- **Stabile Netzwerk-Identität** für eine Gruppe von Pods
- **Load Balancing** über alle Pods
- **Service Discovery** via DNS

### Service YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: nginx          # Welche Pods?
  ports:
  - port: 80            # Service-Port
    targetPort: 80      # Container-Port
  type: ClusterIP       # Service-Typ
```

### Selector

Der Service leitet Traffic an alle Pods mit passenden Labels:

```yaml
# Service-Selector
selector:
  app: nginx
  environment: production

# Passende Pods haben:
labels:
  app: nginx
  environment: production
```

---

## 3. Service-Typen

### ClusterIP (Standard)

- **Nur cluster-intern erreichbar**
- Bekommt eine interne IP
- Standard-Typ

```yaml
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
```

```
┌───────────────────────────────────────┐
│              Cluster                  │
│                                       │
│  Client ──▶ ClusterIP ──▶ Pods       │
```

### Headless Service

- **Kein Load Balancing**, DNS gibt alle Pod-IPs zurück
- Für StatefulSets und direkte Pod-Kommunikation
- `clusterIP: None`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None      # ← Headless!
  selector:
    app: mysql
  ports:
  - port: 3306
```

DNS-Verhalten:
```bash
# Normale Service: gibt Service-IP zurück
nslookup web-service → 10.96.0.1

# Headless Service: gibt alle Pod-IPs zurück
nslookup mysql → 10.244.1.5, 10.244.2.6, 10.244.3.7

# StatefulSet Pod direkt ansprechen:
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
```

```
┌───────────────────────────────────────┐
│              Cluster                  │
│                                       │
│  Client ──▶ DNS ──▶ Pod-IPs direkt   │
│  (intern)   (10.x.x.x)               │
│                                       │
└───────────────────────────────────────┘
```

### NodePort

- **Von außen erreichbar** über Node-IP:Port
- Port-Bereich: 30000-32767
- Öffnet Port auf **jedem** Node

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080    # Optional, sonst automatisch
```

```
┌───────────────────────────────────────┐
│              Cluster                  │
│                                       │
│  External ──▶ Node:30080 ──▶ Pods    │
│                                       │
└───────────────────────────────────────┘
```

### LoadBalancer

- **Externer Load Balancer** (Cloud-Provider)
- Automatische Provisionierung
- Bekommt öffentliche IP

```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
```

```
┌───────────────────────────────────────┐
│              Cloud                    │
│                                       │
│  Internet ──▶ Cloud-LB ──▶ Pods      │
│               (Public IP)             │
│                                       │
└───────────────────────────────────────┘
```

### ExternalName

- **DNS-Alias** für externe Services
- Kein Proxying, nur DNS

```yaml
spec:
  type: ExternalName
  externalName: api.external-service.com
```

---

## 4. Service Discovery & DNS

### Kubernetes DNS

Jeder Service bekommt automatisch einen DNS-Eintrag:

```
<service-name>.<namespace>.svc.cluster.local
```

### Beispiele

```bash
# Im GLEICHEN Namespace - Kurzname reicht
curl http://my-service

# ANDERER Namespace - vollständiger Name nötig!
curl http://my-service.other-namespace.svc.cluster.local

# Vollständiger Name (FQDN) - funktioniert immer
curl http://my-service.default.svc.cluster.local
```

> ⚠️ **Häufiger Fehler:**
> ```bash
> # Das funktioniert NICHT:
> curl http://my-service.other-namespace
>
> # Das funktioniert:
> curl http://my-service.other-namespace.svc.cluster.local
> ```
> Für Cross-Namespace-Zugriffe immer den vollständigen Namen verwenden!

### DNS für Pods

```
<pod-ip-mit-bindestrichen>.<namespace>.pod.cluster.local
```

Beispiel: `10-244-1-5.default.pod.cluster.local`

---

## 5. Endpoints

### Was sind Endpoints?

- Liste der Pod-IPs, die zu einem Service gehören
- Automatisch verwaltet durch Kubernetes
- Werden bei Pod-Änderungen aktualisiert

```bash
kubectl get endpoints my-service
kubectl describe endpoints my-service
```

### Beispiel-Output

```
NAME         ENDPOINTS                         AGE
my-service   10.244.1.5:80,10.244.2.3:80      5m
```

---

## 6. Ingress

### Was ist Ingress?

- **HTTP/HTTPS-Routing** in den Cluster
- **Ein Einstiegspunkt** für mehrere Services
- **Host- und pfadbasiertes Routing**
- **TLS-Terminierung**

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Internet                                           │
│      │                                              │
│      ▼                                              │
│  ┌────────────────────────────────────────────┐     │
│  │              Ingress Controller            │     │
│  │                                            │     │
│  │  /api/*  ──────▶  api-service              │     │
│  │  /web/*  ──────▶  web-service              │     │
│  │  /admin/* ─────▶  admin-service            │     │
│  │                                            │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Ingress Controller

**Wichtig:** Ingress-Ressourcen funktionieren nur mit einem Ingress Controller!

Beliebte Controller:
- NGINX Ingress Controller
- Traefik
- HAProxy
- Istio Gateway

```bash
# Minikube
minikube addons enable ingress
```

### Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Pfad-basiertes Routing

```yaml
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Host-basiertes Routing

```yaml
spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: www.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### TLS/HTTPS

```yaml
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls-secret
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## 7. Netzwerk-Debugging

### Wichtige Befehle

```bash
# Services auflisten
kubectl get services
kubectl describe service my-service

# Endpoints prüfen
kubectl get endpoints my-service

# DNS testen (im Cluster)
kubectl run test --rm -it --image=busybox -- nslookup my-service

# Port-Forwarding zum Testen
kubectl port-forward service/my-service 8080:80

# Service-Logs (Ingress Controller)
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

### Häufige Probleme

| Problem | Mögliche Ursache |
|---------|------------------|
| Service nicht erreichbar | Selector stimmt nicht mit Pod-Labels überein |
| Keine Endpoints | Keine laufenden Pods mit passenden Labels |
| DNS funktioniert nicht | CoreDNS-Pods prüfen |
| Ingress 404 | Ingress Controller läuft nicht |

---

## 8. Zusammenfassung Service-Typen

| Typ | Erreichbar von | Anwendungsfall |
|-----|----------------|----------------|
| **ClusterIP** | Nur Cluster-intern | Interne Kommunikation |
| **NodePort** | Node-IP:Port | Entwicklung, einfacher externer Zugriff |
| **LoadBalancer** | Öffentliche IP | Produktion in der Cloud |
| **ExternalName** | DNS-Weiterleitung | Externe Services einbinden |
| **Ingress** | HTTP/HTTPS | Webapps mit Routing |

---

## Zusammenfassung Tag 4

1. **Service** = stabile Netzwerk-Identität für Pods
2. **ClusterIP** = intern, **NodePort** = extern über Node, **LoadBalancer** = Cloud-LB
3. **DNS**: `<service>.<namespace>.svc.cluster.local`
4. **Endpoints** = dynamische Liste der Pod-IPs
5. **Ingress** = HTTP-Routing, benötigt Ingress Controller
6. **Selector** verbindet Service mit Pods über Labels
