# Tag 2: Übungen - Pods & Container

## Übung 2.1: Pod mit Ressourcen-Limits

### Aufgabe

Erstelle einen Pod mit definierten Ressourcen.

### Schritte

1. **Erstelle `pod-resources.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: resource-demo
   spec:
     containers:
     - name: demo
       image: nginx
       resources:
         requests:
           memory: "64Mi"
           cpu: "100m"
         limits:
           memory: "128Mi"
           cpu: "200m"
   ```

2. **Pod erstellen und prüfen**

   ```bash
   kubectl apply -f pod-resources.yaml
   kubectl describe pod resource-demo | grep -A 10 "Limits"
   ```

### Fragen

- Was passiert, wenn der Container mehr als 128Mi RAM braucht?
- Was passiert bei CPU-Überschreitung?

---

## Übung 2.2: Multi-Container Pod (Sidecar)

### Aufgabe

Erstelle einen Pod mit zwei Containern, die sich ein Volume teilen.

### Schritte

1. **Erstelle `sidecar-pod.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: sidecar-demo
   spec:
     containers:
     - name: writer
       image: busybox
       command: ["/bin/sh", "-c"]
       args:
         - while true; do
             echo "$(date) - Hello from writer" >> /data/log.txt;
             sleep 5;
           done
       volumeMounts:
       - name: shared-data
         mountPath: /data

     - name: reader
       image: busybox
       command: ["/bin/sh", "-c"]
       args:
         - tail -f /data/log.txt
       volumeMounts:
       - name: shared-data
         mountPath: /data

     volumes:
     - name: shared-data
       emptyDir: {}
   ```

2. **Pod starten**

   ```bash
   kubectl apply -f sidecar-pod.yaml
   ```

3. **Logs des Reader-Containers anzeigen**

   ```bash
   kubectl logs sidecar-demo -c reader -f
   ```

4. **In den Writer-Container verbinden**

   ```bash
   kubectl exec -it sidecar-demo -c writer -- cat /data/log.txt
   ```

### Fragen

- Wie greifen beide Container auf die gleichen Daten zu?
- Was passiert mit den Daten, wenn der Pod gelöscht wird?

---

## Übung 2.3: Init-Container

### Aufgabe

Erstelle einen Pod, der erst startet, nachdem ein Init-Container fertig ist.

### Schritte

1. **Erstelle `init-container.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: init-demo
   spec:
     initContainers:
     - name: init-setup
       image: busybox
       command: ['sh', '-c']
       args:
         - echo "Initializing...";
           sleep 10;
           echo "Init complete!" > /work/status.txt
       volumeMounts:
       - name: workdir
         mountPath: /work

     containers:
     - name: main
       image: busybox
       command: ['sh', '-c']
       args:
         - cat /work/status.txt;
           echo "Main container running";
           sleep 3600
       volumeMounts:
       - name: workdir
         mountPath: /work

     volumes:
     - name: workdir
       emptyDir: {}
   ```

2. **Pod starten und beobachten**

   ```bash
   kubectl apply -f init-container.yaml
   kubectl get pods -w
   ```

3. **Init-Container-Logs anzeigen**

   ```bash
   kubectl logs init-demo -c init-setup
   ```

### Fragen

- Was zeigt `kubectl get pods` während der Init-Phase?
- In welcher Reihenfolge laufen die Container?

---

## Übung 2.4: Liveness und Readiness Probes

### Aufgabe

Erstelle einen Pod mit Health-Checks.

### Schritte

1. **Erstelle `probes-pod.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: probes-demo
   spec:
     containers:
     - name: app
       image: nginx
       ports:
       - containerPort: 80

       livenessProbe:
         httpGet:
           path: /
           port: 80
         initialDelaySeconds: 5
         periodSeconds: 5
         failureThreshold: 3

       readinessProbe:
         httpGet:
           path: /
           port: 80
         initialDelaySeconds: 3
         periodSeconds: 3
   ```

2. **Pod starten**

   ```bash
   kubectl apply -f probes-pod.yaml
   kubectl describe pod probes-demo | grep -A 15 "Liveness"
   ```

3. **Pod-Events beobachten**

   ```bash
   kubectl get events --field-selector involvedObject.name=probes-demo
   ```

---

## Übung 2.5: Fehlerhafte Liveness Probe

### Aufgabe

Beobachte, was passiert, wenn eine Liveness Probe fehlschlägt.

### Schritte

1. **Erstelle `failing-probe.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: failing-liveness
   spec:
     containers:
     - name: app
       image: busybox
       command: ["/bin/sh", "-c"]
       args:
         - touch /tmp/healthy;
           sleep 30;
           rm /tmp/healthy;
           sleep 600

       livenessProbe:
         exec:
           command:
           - cat
           - /tmp/healthy
         initialDelaySeconds: 5
         periodSeconds: 5
   ```

2. **Pod starten und beobachten**

   ```bash
   kubectl apply -f failing-probe.yaml
   kubectl get pods -w
   ```

3. **Restart-Zähler beobachten**

   ```bash
   # Nach ca. 40 Sekunden
   kubectl get pods failing-liveness
   ```

### Fragen

- Wie oft wurde der Container neu gestartet?
- Welche Events wurden generiert?

---

## Übung 2.6: Debugging

### Aufgabe

Debugge einen fehlerhaften Pod.

### Schritte

1. **Erstelle einen Pod mit Fehler**

   ```yaml
   # broken-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: broken-pod
   spec:
     containers:
     - name: broken
       image: nginx:nonexistent-tag
   ```

2. **Anwenden und diagnostizieren**

   ```bash
   kubectl apply -f broken-pod.yaml
   kubectl get pods
   kubectl describe pod broken-pod
   kubectl get events --sort-by='.lastTimestamp'
   ```

3. **Problem identifizieren und beheben**

   - Was ist das Problem?
   - Editiere den Pod und fixe ihn

   ```bash
   kubectl edit pod broken-pod
   # oder: lösche und erstelle neu mit korrektem Image
   ```

---

## Übung 2.7: Pod mit mehreren Probes

### Aufgabe

Erstelle einen Pod mit allen drei Probe-Typen.

### Schritte

1. **Erstelle `all-probes.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: all-probes-demo
   spec:
     containers:
     - name: app
       image: nginx
       ports:
       - containerPort: 80

       startupProbe:
         httpGet:
           path: /
           port: 80
         failureThreshold: 30
         periodSeconds: 2

       livenessProbe:
         httpGet:
           path: /
           port: 80
         periodSeconds: 10

       readinessProbe:
         httpGet:
           path: /
           port: 80
         periodSeconds: 5
   ```

2. **Beschreibe den Pod und analysiere die Probes**

   ```bash
   kubectl apply -f all-probes.yaml
   kubectl describe pod all-probes-demo
   ```

---

## Übung 2.8: Umgebungsvariablen

### Aufgabe

Erstelle einen Pod mit Umgebungsvariablen.

### Schritte

1. **Erstelle `env-pod.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: env-demo
   spec:
     containers:
     - name: app
       image: busybox
       command: ["/bin/sh", "-c", "env && sleep 3600"]
       env:
       - name: APP_NAME
         value: "MeineApp"
       - name: APP_VERSION
         value: "1.0.0"
       - name: POD_NAME
         valueFrom:
           fieldRef:
             fieldPath: metadata.name
       - name: POD_IP
         valueFrom:
           fieldRef:
             fieldPath: status.podIP
   ```

2. **Umgebungsvariablen prüfen**

   ```bash
   kubectl apply -f env-pod.yaml
   kubectl logs env-demo | grep -E "APP_|POD_"
   ```

---

## Aufräumen

```bash
kubectl delete pod resource-demo sidecar-demo init-demo probes-demo
kubectl delete pod failing-liveness broken-pod all-probes-demo env-demo
```

---

## Checkliste Tag 2

- [ ] Ich verstehe den Pod-Lifecycle
- [ ] Ich kann Ressourcen (CPU/Memory) definieren
- [ ] Ich habe einen Multi-Container Pod erstellt
- [ ] Ich verstehe Init-Container
- [ ] Ich kann Liveness/Readiness Probes konfigurieren
- [ ] Ich kann Pods debuggen (logs, describe, exec)
- [ ] Ich verstehe den Unterschied zwischen den Probe-Typen
