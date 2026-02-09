# Tag 3: Übungen - Deployments & Workloads

## Übung 3.1: Erstes Deployment

### Aufgabe

Erstelle ein Deployment mit 3 Replicas.

### Schritte

1. **Erstelle `nginx-deployment.yaml`**

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
     labels:
       app: nginx
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

2. **Deployment erstellen**

   ```bash
   kubectl apply -f nginx-deployment.yaml
   ```

3. **Status prüfen**

   ```bash
   kubectl get deployments
   kubectl get replicasets
   kubectl get pods --show-labels
   ```

### Fragen

- Wie heißt das automatisch erstellte ReplicaSet?
- Welche Labels haben die Pods?

---

## Übung 3.2: Skalierung

### Aufgabe

Skaliere das Deployment hoch und runter.

### Schritte

1. **Auf 5 Replicas skalieren**

   ```bash
   kubectl scale deployment nginx-deployment --replicas=5
   kubectl get pods -w
   ```

2. **Auf 2 Replicas reduzieren**

   ```bash
   kubectl scale deployment nginx-deployment --replicas=2
   kubectl get pods -w
   ```

3. **Per YAML skalieren**

   Ändere `replicas: 4` in der YAML-Datei und:

   ```bash
   kubectl apply -f nginx-deployment.yaml
   ```

### Beobachtung

- Wie schnell werden neue Pods erstellt/gelöscht?
- Welche Pods werden beim Runterskalieren gelöscht?

---

## Übung 3.3: Rolling Update

### Aufgabe

Führe ein Rolling Update durch.

### Schritte

1. **Stelle sicher, dass 3 Replicas laufen**

   ```bash
   kubectl scale deployment nginx-deployment --replicas=3
   ```

2. **Update starten (in separatem Terminal Watch starten)**

   ```bash
   # Terminal 1: Watch
   kubectl get pods -w

   # Terminal 2: Update
   kubectl set image deployment/nginx-deployment nginx=nginx:1.25
   ```

3. **Rollout-Status verfolgen**

   ```bash
   kubectl rollout status deployment/nginx-deployment
   ```

4. **Prüfen, dass Update erfolgreich war**

   ```bash
   kubectl describe deployment nginx-deployment | grep Image
   ```

### Fragen

- Was passiert mit den alten Pods?
- Wie viele Pods laufen maximal gleichzeitig?

---

## Übung 3.4: Rollback

### Aufgabe

Mache das Update rückgängig.

### Schritte

1. **History anzeigen**

   ```bash
   kubectl rollout history deployment/nginx-deployment
   ```

2. **Details einer Revision**

   ```bash
   kubectl rollout history deployment/nginx-deployment --revision=1
   ```

3. **Rollback durchführen**

   ```bash
   kubectl rollout undo deployment/nginx-deployment
   ```

4. **Prüfen**

   ```bash
   kubectl describe deployment nginx-deployment | grep Image
   kubectl rollout history deployment/nginx-deployment
   ```

### Fragen

- Welches Image läuft jetzt?
- Wie hat sich die History verändert?

---

## Übung 3.5: Update-Strategie

### Aufgabe

Konfiguriere die Rolling-Update-Strategie.

### Schritte

1. **Erstelle `controlled-deployment.yaml`**

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: controlled-deployment
   spec:
     replicas: 4
     strategy:
       type: RollingUpdate
       rollingUpdate:
         maxSurge: 1
         maxUnavailable: 0
     selector:
       matchLabels:
         app: controlled
     template:
       metadata:
         labels:
           app: controlled
       spec:
         containers:
         - name: nginx
           image: nginx:1.24
   ```

2. **Deployment erstellen**

   ```bash
   kubectl apply -f controlled-deployment.yaml
   ```

3. **Langsames Update beobachten**

   ```bash
   # Terminal 1
   kubectl get pods -l app=controlled -w

   # Terminal 2
   kubectl set image deployment/controlled-deployment nginx=nginx:1.25
   ```

### Frage

- Wie verändert sich das Verhalten, wenn du `maxUnavailable: 2` setzt?

---

## Übung 3.6: Fehlerhaftes Update

### Aufgabe

Beobachte, was bei einem fehlerhaften Update passiert.

### Schritte

1. **Update mit falschem Image**

   ```bash
   kubectl set image deployment/nginx-deployment nginx=nginx:nonexistent
   ```

2. **Status beobachten**

   ```bash
   kubectl rollout status deployment/nginx-deployment
   kubectl get pods
   ```

3. **Problem diagnostizieren**

   ```bash
   kubectl describe pod -l app=nginx | grep -A 5 "Events"
   ```

4. **Rollback**

   ```bash
   kubectl rollout undo deployment/nginx-deployment
   ```

### Beobachtung

- Das alte ReplicaSet bleibt aktiv
- Nur ein Teil der Pods wird aktualisiert
- Rollback ist jederzeit möglich

---

## Übung 3.7: DaemonSet

### Aufgabe

Erstelle ein DaemonSet.

### Schritte

1. **Erstelle `daemonset.yaml`**

   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: node-logger
   spec:
     selector:
       matchLabels:
         app: node-logger
     template:
       metadata:
         labels:
           app: node-logger
       spec:
         containers:
         - name: logger
           image: busybox
           command: ["/bin/sh", "-c"]
           args:
             - while true; do
                 echo "$(date) - Running on $(hostname)";
                 sleep 30;
               done
   ```

2. **DaemonSet erstellen**

   ```bash
   kubectl apply -f daemonset.yaml
   kubectl get daemonsets
   kubectl get pods -l app=node-logger -o wide
   ```

### Frage

- Wie viele Pods wurden erstellt? Warum?
- Was passiert, wenn ein neuer Node hinzukommt?

---

## Übung 3.8: Job

### Aufgabe

Erstelle einen Job, der eine einmalige Aufgabe ausführt.

### Schritte

1. **Erstelle `job.yaml`**

   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: pi-calculator
   spec:
     template:
       spec:
         containers:
         - name: pi
           image: perl
           command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
         restartPolicy: Never
     backoffLimit: 4
   ```

2. **Job ausführen**

   ```bash
   kubectl apply -f job.yaml
   kubectl get jobs
   kubectl get pods
   ```

3. **Ergebnis anzeigen**

   ```bash
   kubectl logs job/pi-calculator
   ```

---

## Übung 3.9: CronJob

### Aufgabe

Erstelle einen CronJob, der alle 2 Minuten läuft.

### Schritte

1. **Erstelle `cronjob.yaml`**

   ```yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: hello-cron
   spec:
     schedule: "*/2 * * * *"
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: hello
               image: busybox
               command: ["/bin/sh", "-c", "date; echo Hello from CronJob"]
             restartPolicy: OnFailure
   ```

2. **CronJob erstellen und beobachten**

   ```bash
   kubectl apply -f cronjob.yaml
   kubectl get cronjobs
   kubectl get jobs -w
   ```

3. **Nach 2-4 Minuten: Jobs und Logs prüfen**

   ```bash
   kubectl get jobs
   kubectl logs job/hello-cron-<timestamp>
   ```

---

## Übung 3.10: Labels und Selektoren

### Aufgabe

Arbeite mit Labels zur Organisation.

### Schritte

1. **Labels zu bestehendem Pod hinzufügen**

   ```bash
   kubectl label deployment nginx-deployment environment=development
   kubectl label deployment nginx-deployment team=frontend
   ```

2. **Labels anzeigen**

   ```bash
   kubectl get deployments --show-labels
   ```

3. **Nach Labels filtern**

   ```bash
   kubectl get pods -l app=nginx
   kubectl get pods -l environment=development
   kubectl get all -l app=nginx
   ```

4. **Label entfernen**

   ```bash
   kubectl label deployment nginx-deployment environment-
   ```

---

## Aufräumen

```bash
kubectl delete deployment nginx-deployment controlled-deployment
kubectl delete daemonset node-logger
kubectl delete job pi-calculator
kubectl delete cronjob hello-cron
```

---

## Checkliste Tag 3

- [ ] Ich kann ein Deployment erstellen
- [ ] Ich verstehe die Beziehung Deployment → ReplicaSet → Pods
- [ ] Ich kann Deployments skalieren
- [ ] Ich kann Rolling Updates durchführen
- [ ] Ich kann Rollbacks machen
- [ ] Ich verstehe DaemonSets
- [ ] Ich kann Jobs und CronJobs erstellen
- [ ] Ich kann mit Labels filtern
