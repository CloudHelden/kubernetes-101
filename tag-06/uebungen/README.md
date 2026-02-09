# Tag 6: Übungen - Security & Best Practices

## Übung 6.1: ServiceAccount erstellen

### Aufgabe

Erstelle einen ServiceAccount und verwende ihn in einem Pod.

### Schritte

1. **ServiceAccount erstellen**

   ```yaml
   # serviceaccount.yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: app-service-account
     namespace: default
   ```

   ```bash
   kubectl apply -f serviceaccount.yaml
   kubectl get serviceaccounts
   ```

2. **Pod mit ServiceAccount**

   ```yaml
   # sa-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: sa-demo-pod
   spec:
     serviceAccountName: app-service-account
     containers:
     - name: app
       image: busybox
       command: ["/bin/sh", "-c", "sleep 3600"]
   ```

   ```bash
   kubectl apply -f sa-pod.yaml
   kubectl get pod sa-demo-pod -o yaml | grep serviceAccount
   ```

---

## Übung 6.2: Role und RoleBinding

### Aufgabe

Erstelle eine Role und binde sie an einen ServiceAccount.

### Schritte

1. **Role erstellen**

   ```yaml
   # role.yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: pod-reader
     namespace: default
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "list", "watch"]
   - apiGroups: [""]
     resources: ["pods/log"]
     verbs: ["get"]
   ```

2. **RoleBinding erstellen**

   ```yaml
   # rolebinding.yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: read-pods-binding
     namespace: default
   subjects:
   - kind: ServiceAccount
     name: app-service-account
     namespace: default
   roleRef:
     kind: Role
     name: pod-reader
     apiGroup: rbac.authorization.k8s.io
   ```

   ```bash
   kubectl apply -f role.yaml
   kubectl apply -f rolebinding.yaml
   ```

3. **Berechtigungen testen**

   ```bash
   # Als ServiceAccount agieren
   kubectl auth can-i list pods --as=system:serviceaccount:default:app-service-account
   kubectl auth can-i delete pods --as=system:serviceaccount:default:app-service-account
   kubectl auth can-i list secrets --as=system:serviceaccount:default:app-service-account
   ```

### Fragen

- Welche Aktionen sind erlaubt?
- Welche sind verboten?

---

## Übung 6.3: Security Context

### Aufgabe

Erstelle einen Pod mit sicheren Security Context Einstellungen.

### Schritte

1. **Erstelle `secure-pod.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: secure-pod
   spec:
     securityContext:
       runAsUser: 1000
       runAsGroup: 3000
       fsGroup: 2000

     containers:
     - name: secure-container
       image: busybox
       command: ["/bin/sh", "-c"]
       args:
         - id;
           echo "User ID:" $(id -u);
           echo "Group ID:" $(id -g);
           touch /data/testfile;
           ls -la /data;
           sleep 3600
       securityContext:
         allowPrivilegeEscalation: false
         readOnlyRootFilesystem: true
         capabilities:
           drop:
             - ALL
       volumeMounts:
       - name: data
         mountPath: /data

     volumes:
     - name: data
       emptyDir: {}
   ```

2. **Pod erstellen und prüfen**

   ```bash
   kubectl apply -f secure-pod.yaml
   kubectl logs secure-pod
   ```

3. **Testen, was nicht funktioniert**

   ```bash
   # Versuche im Container etwas zu schreiben (außerhalb /data)
   kubectl exec secure-pod -- touch /tmp/test
   # Sollte fehlschlagen wegen readOnlyRootFilesystem
   ```

---

## Übung 6.4: Pod mit runAsNonRoot

### Aufgabe

Erzwinge, dass ein Pod nicht als root läuft.

### Schritte

1. **Fehlerhafter Pod (läuft als root)**

   ```yaml
   # root-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: root-pod
   spec:
     securityContext:
       runAsNonRoot: true
     containers:
     - name: nginx
       image: nginx  # nginx läuft standardmäßig als root!
   ```

   ```bash
   kubectl apply -f root-pod.yaml
   kubectl get pods root-pod
   kubectl describe pod root-pod | grep -A 5 "Events"
   ```

2. **Korrigierter Pod**

   ```yaml
   # nonroot-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: nonroot-pod
   spec:
     securityContext:
       runAsNonRoot: true
       runAsUser: 101  # nginx user
     containers:
     - name: nginx
       image: nginxinc/nginx-unprivileged  # Non-root nginx image
       ports:
       - containerPort: 8080
   ```

   ```bash
   kubectl apply -f nonroot-pod.yaml
   kubectl get pods nonroot-pod
   ```

---

## Übung 6.5: Network Policy - Default Deny

### Aufgabe

Implementiere eine Default-Deny Network Policy.

### Schritte

1. **Test-Pods erstellen**

   ```yaml
   # network-test.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: web
     labels:
       app: web
   spec:
     containers:
     - name: nginx
       image: nginx
       ports:
       - containerPort: 80
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: client
     labels:
       app: client
   spec:
     containers:
     - name: busybox
       image: busybox
       command: ["/bin/sh", "-c", "sleep 3600"]
   ```

   ```bash
   kubectl apply -f network-test.yaml
   ```

2. **Konnektivität testen (ohne Policy)**

   ```bash
   kubectl exec client -- wget -qO- --timeout=2 http://$(kubectl get pod web -o jsonpath='{.status.podIP}')
   ```

3. **Default Deny Policy**

   ```yaml
   # default-deny.yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: default-deny-ingress
   spec:
     podSelector: {}
     policyTypes:
     - Ingress
   ```

   ```bash
   kubectl apply -f default-deny.yaml
   ```

4. **Konnektivität erneut testen**

   ```bash
   kubectl exec client -- wget -qO- --timeout=2 http://$(kubectl get pod web -o jsonpath='{.status.podIP}')
   # Sollte fehlschlagen / timeout
   ```

**Hinweis:** Network Policies benötigen einen Network Plugin, der sie unterstützt (z.B. Calico, Cilium). Minikube mit Standard-CNI unterstützt möglicherweise keine Network Policies.

---

## Übung 6.6: Network Policy - Selektiver Zugriff

### Aufgabe

Erlaube nur bestimmten Pods den Zugriff.

### Schritte

1. **Policy für selektiven Zugriff**

   ```yaml
   # allow-client.yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-client-to-web
   spec:
     podSelector:
       matchLabels:
         app: web
     policyTypes:
     - Ingress
     ingress:
     - from:
       - podSelector:
           matchLabels:
             app: client
       ports:
       - protocol: TCP
         port: 80
   ```

   ```bash
   kubectl apply -f allow-client.yaml
   ```

2. **Test**

   ```bash
   # Client sollte nun funktionieren
   kubectl exec client -- wget -qO- --timeout=2 http://$(kubectl get pod web -o jsonpath='{.status.podIP}')
   ```

---

## Übung 6.7: Resource Quota

### Aufgabe

Begrenze Ressourcen in einem Namespace.

### Schritte

1. **Namespace erstellen**

   ```bash
   kubectl create namespace limited
   ```

2. **ResourceQuota erstellen**

   ```yaml
   # quota.yaml
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: compute-quota
     namespace: limited
   spec:
     hard:
       requests.cpu: "1"
       requests.memory: 1Gi
       limits.cpu: "2"
       limits.memory: 2Gi
       pods: "5"
   ```

   ```bash
   kubectl apply -f quota.yaml
   kubectl describe quota -n limited
   ```

3. **Pods erstellen und Quota testen**

   ```yaml
   # quota-test-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: quota-pod
     namespace: limited
   spec:
     containers:
     - name: app
       image: nginx
       resources:
         requests:
           cpu: "200m"
           memory: 256Mi
         limits:
           cpu: "500m"
           memory: 512Mi
   ```

   ```bash
   kubectl apply -f quota-test-pod.yaml
   kubectl describe quota -n limited
   ```

4. **Überschreitung testen**

   Erstelle mehrere Pods bis das Quota erreicht ist.

---

## Übung 6.8: LimitRange

### Aufgabe

Setze Standard-Limits für Container.

### Schritte

1. **LimitRange erstellen**

   ```yaml
   # limitrange.yaml
   apiVersion: v1
   kind: LimitRange
   metadata:
     name: default-limits
     namespace: limited
   spec:
     limits:
     - default:
         cpu: "300m"
         memory: "200Mi"
       defaultRequest:
         cpu: "100m"
         memory: "100Mi"
       type: Container
   ```

   ```bash
   kubectl apply -f limitrange.yaml
   ```

2. **Pod ohne Ressourcen erstellen**

   ```yaml
   # no-limits-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: auto-limits-pod
     namespace: limited
   spec:
     containers:
     - name: app
       image: nginx
   ```

   ```bash
   kubectl apply -f no-limits-pod.yaml
   kubectl describe pod auto-limits-pod -n limited | grep -A 5 "Limits"
   ```

### Beobachtung

LimitRange hat automatisch Defaults gesetzt.

---

## Übung 6.9: RBAC Debugging

### Aufgabe

Finde heraus, warum eine Aktion fehlschlägt.

### Schritte

1. **Berechtigungen prüfen**

   ```bash
   # Kann der ServiceAccount Secrets lesen?
   kubectl auth can-i get secrets --as=system:serviceaccount:default:app-service-account

   # Was kann der ServiceAccount alles?
   kubectl auth can-i --list --as=system:serviceaccount:default:app-service-account
   ```

2. **Fehlende Berechtigung hinzufügen**

   ```yaml
   # secrets-role.yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: secret-reader
   rules:
   - apiGroups: [""]
     resources: ["secrets"]
     verbs: ["get", "list"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: read-secrets-binding
   subjects:
   - kind: ServiceAccount
     name: app-service-account
   roleRef:
     kind: Role
     name: secret-reader
     apiGroup: rbac.authorization.k8s.io
   ```

   ```bash
   kubectl apply -f secrets-role.yaml
   kubectl auth can-i get secrets --as=system:serviceaccount:default:app-service-account
   ```

---

## Übung 6.10: Abschlussprojekt - Sichere Anwendung

### Aufgabe

Deploye eine Anwendung mit allen Sicherheits-Best-Practices.

### Anforderungen

- [ ] Eigener ServiceAccount
- [ ] Non-root Container
- [ ] Read-only Filesystem
- [ ] Resource Limits
- [ ] Network Policy
- [ ] Keine unnecessary Capabilities

### Lösung

```yaml
# secure-app.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-app-sa
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: secure-app-config
data:
  APP_NAME: "SecureApp"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      serviceAccountName: secure-app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000

      containers:
      - name: app
        image: nginxinc/nginx-unprivileged:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
              - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
        envFrom:
        - configMapRef:
            name: secure-app-config

      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: secure-app-service
spec:
  selector:
    app: secure-app
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: secure-app-policy
spec:
  podSelector:
    matchLabels:
      app: secure-app
  policyTypes:
  - Ingress
  ingress:
  - from: []  # Von überall (für Service)
    ports:
    - protocol: TCP
      port: 8080
```

```bash
kubectl apply -f secure-app.yaml
kubectl get all -l app=secure-app
kubectl exec -it deploy/secure-app -- id
```

---

## Aufräumen

```bash
kubectl delete pod sa-demo-pod secure-pod root-pod nonroot-pod web client quota-pod auto-limits-pod
kubectl delete serviceaccount app-service-account secure-app-sa
kubectl delete role pod-reader secret-reader
kubectl delete rolebinding read-pods-binding read-secrets-binding
kubectl delete networkpolicy default-deny-ingress allow-client-to-web secure-app-policy
kubectl delete namespace limited
kubectl delete deployment secure-app
kubectl delete service secure-app-service
kubectl delete configmap secure-app-config
```

---

## Checkliste Tag 6

- [ ] Ich verstehe RBAC (Role, ClusterRole, RoleBinding)
- [ ] Ich kann ServiceAccounts erstellen und verwenden
- [ ] Ich kann Berechtigungen mit `kubectl auth can-i` prüfen
- [ ] Ich kann Security Contexts konfigurieren
- [ ] Ich verstehe runAsNonRoot und readOnlyRootFilesystem
- [ ] Ich kann Network Policies erstellen
- [ ] Ich verstehe ResourceQuotas und LimitRanges
- [ ] Ich kann eine sichere Anwendung deployen
