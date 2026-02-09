# Tag 6: Security & Best Practices

## Lernziele

- Kubernetes-Sicherheitskonzepte verstehen
- RBAC (Role-Based Access Control) konfigurieren
- Security Contexts anwenden
- Network Policies einrichten
- Best Practices für Produktion

---

## 1. Kubernetes Security Overview

### Die 4 C's der Cloud-Native Security

```
┌─────────────────────────────────────────────┐
│                   Cloud                      │
│  ┌───────────────────────────────────────┐  │
│  │              Cluster                  │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │           Container             │  │  │
│  │  │  ┌───────────────────────────┐  │  │  │
│  │  │  │          Code             │  │  │  │
│  │  │  └───────────────────────────┘  │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

| Ebene | Maßnahmen |
|-------|-----------|
| **Cloud** | IAM, Netzwerk-Firewall, Encryption |
| **Cluster** | RBAC, Network Policies, Pod Security |
| **Container** | Image-Scanning, Security Context |
| **Code** | Sichere Programmierung, Dependency-Scanning |

---

## 2. RBAC - Role-Based Access Control

### Konzept

```
┌──────────────┐
│    User /    │
│ ServiceAccount│
└──────┬───────┘
       │ bindet
       ▼
┌──────────────┐     ┌──────────────┐
│ RoleBinding  │────▶│     Role     │
│              │     │   (Regeln)   │
└──────────────┘     └──────────────┘
       │
       ▼
┌──────────────┐
│  Ressourcen  │
│ (Pods, etc.) │
└──────────────┘
```

### RBAC-Komponenten

| Komponente | Beschreibung | Scope |
|------------|--------------|-------|
| **Role** | Berechtigungen in einem Namespace | Namespace |
| **ClusterRole** | Berechtigungen cluster-weit | Cluster |
| **RoleBinding** | Bindet Role an User/ServiceAccount | Namespace |
| **ClusterRoleBinding** | Bindet ClusterRole cluster-weit | Cluster |

### Role erstellen

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]              # "" = Core API
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### Verben (Verbs)

| Verb | HTTP | Beschreibung |
|------|------|--------------|
| `get` | GET | Einzelne Ressource lesen |
| `list` | GET | Ressourcen auflisten |
| `watch` | GET | Änderungen beobachten |
| `create` | POST | Ressource erstellen |
| `update` | PUT | Ressource aktualisieren |
| `patch` | PATCH | Ressource teilweise ändern |
| `delete` | DELETE | Ressource löschen |

### RoleBinding erstellen

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-service-account
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole für cluster-weite Rechte

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

---

## 3. ServiceAccounts

### Was sind ServiceAccounts?

- Identität für **Pods** (nicht Menschen)
- Jeder Pod hat einen ServiceAccount
- Standard: `default` ServiceAccount

### ServiceAccount erstellen

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
```

### In Pod verwenden

```yaml
spec:
  serviceAccountName: my-service-account
  containers:
  - name: app
    image: myapp
```

### Token automatisch mounten (oder nicht)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-token-sa
automountServiceAccountToken: false
```

---

## 4. Security Context

### Pod-Level Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000          # Nicht als root
    runAsGroup: 3000         # Gruppe
    fsGroup: 2000            # Dateisystem-Gruppe
    runAsNonRoot: true       # Erzwinge non-root

  containers:
  - name: app
    image: myapp
```

### Container-Level Security Context

```yaml
spec:
  containers:
  - name: app
    image: myapp
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
        add:
          - NET_BIND_SERVICE
```

### Wichtige Einstellungen

| Einstellung | Empfehlung | Beschreibung |
|-------------|------------|--------------|
| `runAsNonRoot` | true | Verhindert root-Ausführung |
| `readOnlyRootFilesystem` | true | Filesystem schreibgeschützt |
| `allowPrivilegeEscalation` | false | Keine Rechte-Erweiterung |
| `capabilities.drop` | ALL | Alle Linux-Capabilities entfernen |

---

## 5. Network Policies

### Konzept

- **Firewall-Regeln** für Pod-zu-Pod-Kommunikation
- Standardmäßig: alles erlaubt
- Network Policies sind **additiv** (Whitelist)

> ⚠️ **KRITISCH: CNI-Plugin erforderlich!**
>
> NetworkPolicies funktionieren **NUR** mit einem CNI-Plugin, das sie unterstützt:
>
> | CNI Plugin | NetworkPolicy Support |
> |------------|----------------------|
> | **Calico** | ✅ Ja |
> | **Cilium** | ✅ Ja |
> | **Weave Net** | ✅ Ja |
> | **Flannel** | ❌ Nein! |
> | **kubenet** | ❌ Nein! |
>
> Bei Flannel (häufig in Minikube/kubeadm): NetworkPolicies werden **stillschweigend ignoriert**!
> ```bash
> # Prüfen welches CNI aktiv ist:
> kubectl get pods -n kube-system | grep -E "calico|cilium|weave|flannel"
> ```

### Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}          # Alle Pods
  policyTypes:
  - Ingress
  - Egress
```

### Ingress erlauben

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Egress erlauben

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Namespace-übergreifend

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
    podSelector:
      matchLabels:
        app: frontend
```

---

## 6. Pod Security Standards

### Die drei Standards

| Standard | Beschreibung |
|----------|--------------|
| **Privileged** | Keine Einschränkungen |
| **Baseline** | Minimale Einschränkungen |
| **Restricted** | Best Practices, stark eingeschränkt |

### Namespace-Label für Enforcement

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

---

## 7. Image Security

### Best Practices für Images

1. **Spezifische Tags verwenden**
   ```yaml
   # Schlecht
   image: nginx:latest

   # Gut
   image: nginx:1.25.3
   ```

2. **Image Digest verwenden**
   ```yaml
   image: nginx@sha256:abc123...
   ```

3. **Nur vertrauenswürdige Registries**
   ```yaml
   # Privates Repository
   image: myregistry.com/myapp:v1.0.0
   ```

4. **Images scannen**
   - Trivy, Clair, Anchore
   - In CI/CD-Pipeline integrieren

### ImagePullPolicy

```yaml
containers:
- name: app
  image: myapp:v1
  imagePullPolicy: Always  # Immer neu pullen
  # imagePullPolicy: IfNotPresent  # Nur wenn nicht vorhanden
  # imagePullPolicy: Never  # Nie pullen (lokal)
```

---

## 8. Secrets Best Practices

### Verschlüsselung at Rest

```yaml
# EncryptionConfiguration für kube-apiserver
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-key>
  - identity: {}
```

### External Secrets Management

- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Secret Manager

### Secrets nie in Git!

- `.gitignore` für Secret-Dateien
- Sealed Secrets (Bitnami)
- SOPS (Mozilla)
- Git-Crypt

---

## 9. Resource Quotas & Limits

### ResourceQuota pro Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
```

### LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

---

## 10. Best Practices Checkliste

### Cluster-Sicherheit

- [ ] RBAC aktiviert und konfiguriert
- [ ] Network Policies implementiert
- [ ] Pod Security Standards aktiviert
- [ ] API-Server-Audit-Logging aktiviert
- [ ] etcd verschlüsselt

### Pod-Sicherheit

- [ ] Non-root User (`runAsNonRoot: true`)
- [ ] Read-only Filesystem wo möglich
- [ ] Keine Privilege Escalation
- [ ] Minimale Capabilities
- [ ] Resource Limits definiert

### Image-Sicherheit

- [ ] Spezifische Image-Tags
- [ ] Image-Scanning in CI/CD
- [ ] Vertrauenswürdige Registries
- [ ] Minimale Base-Images

### Secrets

- [ ] Encryption at Rest aktiviert
- [ ] Keine Secrets in Git
- [ ] Secrets regelmäßig rotieren
- [ ] Least Privilege für Secret-Zugriff

### Netzwerk

- [ ] Default Deny Network Policies
- [ ] TLS für alle internen Services
- [ ] Ingress mit TLS

---

## 11. Monitoring & Audit

### Wichtige Metriken

- Pod-Restarts
- Resource-Nutzung
- Failed API-Requests
- RBAC-Denials

### Audit-Logging

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
```

### Tools

- Prometheus + Grafana (Monitoring)
- Falco (Runtime Security)
- OPA/Gatekeeper (Policy Enforcement)
- Kube-bench (CIS Benchmark)

---

## Zusammenfassung Tag 6

1. **RBAC** = Wer darf was? (Role, RoleBinding, ServiceAccount)
2. **Security Context** = Wie läuft der Container? (runAsNonRoot, Capabilities)
3. **Network Policies** = Welche Pods kommunizieren?
4. **Pod Security Standards** = Baseline, Restricted, Privileged
5. **Best Practices**: Least Privilege, Defense in Depth
6. **Monitoring & Audit** = Kontinuierliche Überwachung
