# Tag 1: Übungen - Cluster Setup & Erste Schritte

## Übung 1.1: Minikube Installation und Start

### Aufgabe

Installiere Minikube und starte deinen ersten Kubernetes-Cluster.

### Schritte

1. **Installiere Minikube**

   ```bash
   # macOS
   brew install minikube

   # Linux (Ubuntu/Debian)
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   sudo install minikube-linux-amd64 /usr/local/bin/minikube

   # Windows (PowerShell als Admin)
   winget install minikube
   ```

2. **Installiere kubectl**

   ```bash
   # macOS
   brew install kubectl

   # Linux
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   sudo install kubectl /usr/local/bin/kubectl
   ```

3. **Starte den Cluster**

   ```bash
   minikube start --cpus=2 --memory=4096
   ```

4. **Überprüfe die Installation**

   ```bash
   kubectl cluster-info
   kubectl get nodes
   minikube status
   ```

### Erwartetes Ergebnis

- Cluster läuft
- Ein Node mit Status "Ready"

---

## Übung 1.2: Cluster erkunden

### Aufgabe

Erkunde deinen neuen Cluster mit kubectl-Befehlen.

### Schritte

1. **Zeige alle Namespaces**

   ```bash
   kubectl get namespaces
   ```

   Welche Namespaces existieren standardmäßig?

2. **Zeige System-Pods**

   ```bash
   kubectl get pods -n kube-system
   ```

   Erkennst du Komponenten aus der Theorie wieder?

3. **Node-Details anzeigen**

   ```bash
   kubectl describe node minikube
   ```

   Finde heraus:
   - Wie viel CPU hat der Node?
   - Wie viel Memory?
   - Welche Kubernetes-Version läuft?

4. **API-Ressourcen auflisten**

   ```bash
   kubectl api-resources
   ```

   Wie viele verschiedene Ressourcen-Typen gibt es?

---

## Übung 1.3: Dein erster Pod (imperativ)

### Aufgabe

Erstelle einen Pod mit einem nginx-Container.

### Schritte

1. **Pod erstellen**

   ```bash
   kubectl run mein-erster-pod --image=nginx:latest
   ```

2. **Pod-Status prüfen**

   ```bash
   kubectl get pods
   kubectl get pods -o wide
   ```

3. **Pod-Details anzeigen**

   ```bash
   kubectl describe pod mein-erster-pod
   ```

4. **Logs anzeigen**

   ```bash
   kubectl logs mein-erster-pod
   ```

5. **In den Pod verbinden**

   ```bash
   kubectl exec -it mein-erster-pod -- /bin/bash
   # Im Container:
   curl localhost
   exit
   ```

6. **Pod löschen**

   ```bash
   kubectl delete pod mein-erster-pod
   ```

---

## Übung 1.4: Dein erster Pod (deklarativ)

### Aufgabe

Erstelle den gleichen Pod, aber diesmal mit einer YAML-Datei.

### Schritte

1. **Erstelle die Datei `mein-pod.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-deklarativ
     labels:
       app: webserver
       environment: learning
   spec:
     containers:
     - name: nginx
       image: nginx:latest
       ports:
       - containerPort: 80
   ```

2. **Pod anwenden**

   ```bash
   kubectl apply -f mein-pod.yaml
   ```

3. **Überprüfen**

   ```bash
   kubectl get pods --show-labels
   ```

4. **Mit Label-Selector filtern**

   ```bash
   kubectl get pods -l app=webserver
   kubectl get pods -l environment=learning
   ```

---

## Übung 1.5: Port-Forwarding

### Aufgabe

Greife auf den nginx-Webserver im Pod zu.

### Schritte

1. **Port-Forwarding starten**

   ```bash
   kubectl port-forward pod/nginx-deklarativ 8080:80
   ```

2. **In einem neuen Terminal testen**

   ```bash
   curl http://localhost:8080
   ```

   Oder öffne http://localhost:8080 im Browser.

3. **Port-Forwarding beenden**

   Mit `Ctrl+C` im ersten Terminal.

---

## Übung 1.6: YAML generieren lassen

### Aufgabe

Lerne, wie kubectl dir YAML generieren kann.

### Schritte

1. **YAML für einen neuen Pod generieren (ohne zu erstellen)**

   ```bash
   kubectl run test-pod --image=nginx --dry-run=client -o yaml
   ```

2. **In Datei speichern**

   ```bash
   kubectl run test-pod --image=nginx --dry-run=client -o yaml > test-pod.yaml
   ```

3. **Bestehenden Pod als YAML exportieren**

   ```bash
   kubectl get pod nginx-deklarativ -o yaml > export.yaml
   ```

   Schau dir die Datei an - was wurde alles hinzugefügt?

---

## Übung 1.7: Aufräumen

### Aufgabe

Lösche alle erstellten Ressourcen.

### Schritte

```bash
# Einzelnen Pod löschen
kubectl delete pod nginx-deklarativ

# Alle Pods löschen (Vorsicht in Produktion!)
kubectl delete pods --all

# Per YAML-Datei löschen
kubectl delete -f mein-pod.yaml
```

---

## Bonus-Übung: Minikube Dashboard

### Aufgabe

Starte das Kubernetes Dashboard.

```bash
minikube dashboard
```

Erkunde die grafische Oberfläche:
- Welche Workloads laufen?
- Wie sieht die Cluster-Übersicht aus?
- Kannst du einen Pod über das Dashboard erstellen?

---

## Checkliste Tag 1

- [ ] Minikube ist installiert und läuft
- [ ] kubectl funktioniert
- [ ] Ich kann Pods auflisten
- [ ] Ich habe einen Pod imperativ erstellt
- [ ] Ich habe einen Pod deklarativ erstellt
- [ ] Ich verstehe den Unterschied zwischen imperativ und deklarativ
- [ ] Ich kann Port-Forwarding nutzen
- [ ] Ich kann YAML generieren lassen
