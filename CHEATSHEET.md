# Kubernetes Cheat Sheet

## kubectl Grundlagen

```bash
# Cluster-Info
kubectl cluster-info
kubectl get nodes
kubectl version

# Kontext wechseln
kubectl config get-contexts
kubectl config use-context <name>
kubectl config set-context --current --namespace=<ns>
```

## Ressourcen anzeigen

```bash
# Basis
kubectl get pods
kubectl get pods -o wide              # Mehr Details
kubectl get pods -o yaml              # YAML-Output
kubectl get pods --all-namespaces     # Alle Namespaces
kubectl get pods -n <namespace>       # Bestimmter Namespace
kubectl get pods -w                   # Watch (live)

# Alle Ressourcen
kubectl get all
kubectl get all -l app=myapp          # Mit Label-Filter

# Details
kubectl describe pod <name>
kubectl describe node <name>
```

## Ressourcen erstellen/ändern

```bash
# Erstellen
kubectl apply -f datei.yaml
kubectl apply -f verzeichnis/
kubectl create -f datei.yaml

# Generieren (dry-run)
kubectl run test --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment test --image=nginx --dry-run=client -o yaml > deploy.yaml

# Bearbeiten
kubectl edit deployment <name>
kubectl patch deployment <name> -p '{"spec":{"replicas":3}}'

# Löschen
kubectl delete -f datei.yaml
kubectl delete pod <name>
kubectl delete pods --all
```

## Debugging

```bash
# Logs
kubectl logs <pod>
kubectl logs <pod> -c <container>     # Multi-Container
kubectl logs <pod> -f                  # Follow
kubectl logs <pod> --previous          # Vorheriger Container

# Shell im Container
kubectl exec -it <pod> -- /bin/sh
kubectl exec -it <pod> -c <container> -- /bin/bash

# Port-Forward
kubectl port-forward pod/<name> 8080:80
kubectl port-forward svc/<name> 8080:80

# Events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.name=<pod>
```

## Deployments

```bash
# Skalieren
kubectl scale deployment <name> --replicas=5

# Update
kubectl set image deployment/<name> container=image:tag

# Rollout
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2
kubectl rollout restart deployment/<name>
```

## Labels & Selektoren

```bash
# Labels anzeigen
kubectl get pods --show-labels

# Nach Label filtern
kubectl get pods -l app=nginx
kubectl get pods -l 'app in (nginx, apache)'
kubectl get pods -l app=nginx,environment=prod

# Label setzen/entfernen
kubectl label pod <name> env=prod
kubectl label pod <name> env-                # Entfernen
```

## ConfigMaps & Secrets

```bash
# ConfigMap
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=config.txt
kubectl get configmap <name> -o yaml

# Secret
kubectl create secret generic <name> --from-literal=password=geheim
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 -d
```

## RBAC

```bash
# Berechtigungen prüfen
kubectl auth can-i create pods
kubectl auth can-i delete pods --as=<user>
kubectl auth can-i --list
```

## Namespaces

```bash
kubectl get namespaces
kubectl create namespace <name>
kubectl delete namespace <name>

# In Namespace arbeiten
kubectl -n <namespace> get pods
```

## Nützliche Flags

| Flag | Beschreibung |
|------|--------------|
| `-o wide` | Mehr Spalten |
| `-o yaml` | YAML-Output |
| `-o json` | JSON-Output |
| `-o jsonpath='{...}'` | JSONPath-Query |
| `-w` | Watch-Modus |
| `-l key=value` | Label-Selector |
| `--all-namespaces` / `-A` | Alle Namespaces |
| `--dry-run=client` | Nur simulieren |
| `-f` | Datei/Verzeichnis |

## JSONPath Beispiele

```bash
# Pod-IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Container-Images
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Node-Namen
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'
```

## Häufige Probleme

| Symptom | Befehl | Mögliche Ursache |
|---------|--------|------------------|
| ImagePullBackOff | `kubectl describe pod` | Image nicht gefunden |
| CrashLoopBackOff | `kubectl logs --previous` | App stürzt ab |
| Pending | `kubectl describe pod` | Keine Ressourcen |
| OOMKilled | `kubectl describe pod` | Memory-Limit |

## Shortcuts

```bash
# Kurzformen
po = pods
svc = services
deploy = deployments
rs = replicasets
ds = daemonsets
sts = statefulsets
cm = configmaps
ns = namespaces
pv = persistentvolumes
pvc = persistentvolumeclaims
```
