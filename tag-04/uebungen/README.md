# Tag 4: Übungen - Networking

## Übung 4.1: ClusterIP Service

### Aufgabe

Erstelle ein Deployment mit einem ClusterIP Service.

### Schritte

1. **Erstelle `web-deployment.yaml`**

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web-app
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: web
     template:
       metadata:
         labels:
           app: web
       spec:
         containers:
         - name: nginx
           image: nginx
           ports:
           - containerPort: 80
   ```

2. **Erstelle `web-service.yaml`**

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: web-service
   spec:
     selector:
       app: web
     ports:
     - port: 80
       targetPort: 80
     type: ClusterIP
   ```

3. **Ressourcen erstellen**

   ```bash
   kubectl apply -f web-deployment.yaml
   kubectl apply -f web-service.yaml
   ```

4. **Service und Endpoints prüfen**

   ```bash
   kubectl get services
   kubectl get endpoints web-service
   kubectl describe service web-service
   ```

### Fragen

- Welche IP hat der Service?
- Wie viele Endpoints gibt es?

---

## Übung 4.2: Service-Konnektivität testen

### Aufgabe

Teste die Verbindung zum Service aus einem anderen Pod.

### Schritte

1. **Test-Pod starten**

   ```bash
   kubectl run test-pod --rm -it --image=busybox -- /bin/sh
   ```

2. **Im Test-Pod: Service aufrufen**

   ```sh
   # Per Service-Name
   wget -qO- http://web-service

   # Per vollem DNS-Namen
   wget -qO- http://web-service.default.svc.cluster.local

   # DNS auflösen
   nslookup web-service
   ```

3. **Mehrere Aufrufe machen**

   ```sh
   for i in 1 2 3 4 5; do wget -qO- http://web-service | grep -o "nginx"; done
   ```

### Beobachtung

Der Traffic wird auf alle Pods verteilt (Load Balancing).

---

## Übung 4.3: NodePort Service

### Aufgabe

Erstelle einen NodePort Service für externen Zugriff.

### Schritte

1. **Erstelle `nodeport-service.yaml`**

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: web-nodeport
   spec:
     selector:
       app: web
     ports:
     - port: 80
       targetPort: 80
       nodePort: 30080
     type: NodePort
   ```

2. **Service erstellen**

   ```bash
   kubectl apply -f nodeport-service.yaml
   kubectl get services
   ```

3. **Externen Zugriff testen**

   ```bash
   # Minikube: Service-URL bekommen
   minikube service web-nodeport --url

   # Oder direkt
   curl http://$(minikube ip):30080
   ```

### Fragen

- Auf welchem Port ist der Service erreichbar?
- Was passiert, wenn du einen anderen Node hättest?

---

## Übung 4.4: Verschiedene Ports

### Aufgabe

Erstelle einen Service mit unterschiedlichen Ports.

### Schritte

1. **Erstelle `multi-port-service.yaml`**

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: multi-port-service
   spec:
     selector:
       app: web
     ports:
     - name: http
       port: 8080        # Service-Port (extern)
       targetPort: 80    # Container-Port (intern)
     - name: https
       port: 8443
       targetPort: 443
   ```

2. **Service erstellen und testen**

   ```bash
   kubectl apply -f multi-port-service.yaml

   # Test-Pod
   kubectl run test --rm -it --image=busybox -- wget -qO- http://multi-port-service:8080
   ```

---

## Übung 4.5: Service ohne Selector

### Aufgabe

Erstelle einen Service, der auf eine externe Ressource zeigt.

### Schritte

1. **Service ohne Selector erstellen**

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: external-db
   spec:
     ports:
     - port: 5432
   ---
   apiVersion: v1
   kind: Endpoints
   metadata:
     name: external-db    # MUSS gleicher Name wie Service sein!
   subsets:
   - addresses:
     - ip: 192.168.1.100  # Externe IP
     ports:
     - port: 5432
   ```

2. **Endpoints prüfen**

   ```bash
   kubectl get endpoints external-db
   ```

### Anwendungsfall

- Legacy-Systeme einbinden
- Datenbanken außerhalb des Clusters

---

## Übung 4.6: Ingress einrichten

### Aufgabe

Richte Ingress für HTTP-Routing ein.

### Schritte

1. **Ingress Controller aktivieren (Minikube)**

   ```bash
   minikube addons enable ingress
   kubectl get pods -n ingress-nginx
   ```

2. **Zweites Backend erstellen**

   ```yaml
   # api-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: api-app
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: api
     template:
       metadata:
         labels:
           app: api
       spec:
         containers:
         - name: api
           image: hashicorp/http-echo
           args: ["-text=API Response"]
           ports:
           - containerPort: 5678
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: api-service
   spec:
     selector:
       app: api
     ports:
     - port: 80
       targetPort: 5678
   ```

3. **Ingress erstellen**

   ```yaml
   # ingress.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: demo-ingress
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /
   spec:
     rules:
     - http:
         paths:
         - path: /web
           pathType: Prefix
           backend:
             service:
               name: web-service
               port:
                 number: 80
         - path: /api
           pathType: Prefix
           backend:
             service:
               name: api-service
               port:
                 number: 80
   ```

4. **Alles anwenden und testen**

   ```bash
   kubectl apply -f api-deployment.yaml
   kubectl apply -f ingress.yaml

   # Warten bis Ingress eine Adresse hat
   kubectl get ingress

   # Testen
   curl http://$(minikube ip)/web
   curl http://$(minikube ip)/api
   ```

---

## Übung 4.7: Host-basiertes Routing

### Aufgabe

Konfiguriere Routing basierend auf Hostnamen.

### Schritte

1. **Ingress mit Hosts erstellen**

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: host-ingress
   spec:
     rules:
     - host: web.local
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: web-service
               port:
                 number: 80
     - host: api.local
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: api-service
               port:
                 number: 80
   ```

2. **Hosts-Datei anpassen (lokal)**

   ```bash
   # IP herausfinden
   minikube ip

   # /etc/hosts (macOS/Linux) oder C:\Windows\System32\drivers\etc\hosts
   # Füge hinzu:
   # <minikube-ip> web.local api.local
   ```

3. **Testen**

   ```bash
   curl http://web.local
   curl http://api.local
   ```

---

## Übung 4.8: DNS Debugging

### Aufgabe

Untersuche den Kubernetes DNS.

### Schritte

1. **DNS-Pod starten**

   ```bash
   kubectl run dns-test --rm -it --image=tutum/dnsutils -- /bin/bash
   ```

2. **DNS-Abfragen im Pod**

   ```bash
   # Service auflösen
   nslookup web-service
   nslookup web-service.default.svc.cluster.local

   # Kubernetes API
   nslookup kubernetes

   # Externe Domain
   nslookup google.com

   # DNS-Server prüfen
   cat /etc/resolv.conf
   ```

3. **CoreDNS prüfen**

   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   kubectl logs -n kube-system -l k8s-app=kube-dns
   ```

---

## Übung 4.9: Service Load Balancing beobachten

### Aufgabe

Beobachte, wie Traffic auf Pods verteilt wird.

### Schritte

1. **Deployment mit individuellem Response**

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: echo-deployment
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: echo
     template:
       metadata:
         labels:
           app: echo
       spec:
         containers:
         - name: echo
           image: hashicorp/http-echo
           args:
           - -text=$(POD_NAME)
           env:
           - name: POD_NAME
             valueFrom:
               fieldRef:
                 fieldPath: metadata.name
           ports:
           - containerPort: 5678
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: echo-service
   spec:
     selector:
       app: echo
     ports:
     - port: 80
       targetPort: 5678
   ```

2. **Mehrfach aufrufen**

   ```bash
   kubectl apply -f echo-deployment.yaml

   # Im Test-Pod
   kubectl run test --rm -it --image=busybox -- /bin/sh
   for i in $(seq 1 10); do wget -qO- http://echo-service; done
   ```

### Beobachtung

Unterschiedliche Pod-Namen zeigen Load Balancing.

---

## Aufräumen

```bash
kubectl delete deployment web-app api-app echo-deployment
kubectl delete service web-service web-nodeport multi-port-service api-service echo-service external-db
kubectl delete ingress demo-ingress host-ingress
```

---

## Checkliste Tag 4

- [ ] Ich kann einen ClusterIP Service erstellen
- [ ] Ich verstehe den Unterschied zwischen ClusterIP, NodePort und LoadBalancer
- [ ] Ich kann Services per DNS ansprechen
- [ ] Ich kann Endpoints inspizieren
- [ ] Ich kann Ingress für HTTP-Routing einrichten
- [ ] Ich verstehe host- und pfadbasiertes Routing
- [ ] Ich kann Netzwerk-Probleme debuggen
