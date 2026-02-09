# Tag 5: Übungen - Storage & Konfiguration

## Übung 5.1: emptyDir Volume

### Aufgabe

Erstelle einen Pod mit zwei Containern, die sich ein emptyDir Volume teilen.

### Schritte

1. **Erstelle `emptydir-pod.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: shared-volume-pod
   spec:
     containers:
     - name: producer
       image: busybox
       command: ["/bin/sh", "-c"]
       args:
         - while true; do
             echo "$(date) - Neue Nachricht" >> /shared/messages.txt;
             sleep 5;
           done
       volumeMounts:
       - name: shared-data
         mountPath: /shared

     - name: consumer
       image: busybox
       command: ["/bin/sh", "-c", "tail -f /shared/messages.txt"]
       volumeMounts:
       - name: shared-data
         mountPath: /shared

     volumes:
     - name: shared-data
       emptyDir: {}
   ```

2. **Pod erstellen und Logs beobachten**

   ```bash
   kubectl apply -f emptydir-pod.yaml
   kubectl logs shared-volume-pod -c consumer -f
   ```

3. **In Producer-Container prüfen**

   ```bash
   kubectl exec shared-volume-pod -c producer -- cat /shared/messages.txt
   ```

### Fragen

- Was passiert mit den Daten, wenn du den Pod löschst?
- Wo werden die Daten physisch gespeichert?

---

## Übung 5.2: ConfigMap erstellen

### Aufgabe

Erstelle eine ConfigMap und verwende sie in einem Pod.

### Schritte

1. **ConfigMap imperativ erstellen**

   ```bash
   kubectl create configmap app-settings \
     --from-literal=APP_NAME="MeineApp" \
     --from-literal=LOG_LEVEL="debug" \
     --from-literal=MAX_CONNECTIONS="100"

   kubectl get configmap app-settings -o yaml
   ```

2. **ConfigMap deklarativ erstellen**

   ```yaml
   # app-config.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: app-config
   data:
     database.host: "mysql.default.svc.cluster.local"
     database.port: "3306"
     feature.new_ui: "true"

     # Mehrzeilige Konfiguration
     nginx.conf: |
       server {
         listen 80;
         server_name localhost;
         location / {
           root /usr/share/nginx/html;
         }
       }
   ```

   ```bash
   kubectl apply -f app-config.yaml
   ```

3. **ConfigMap anzeigen**

   ```bash
   kubectl describe configmap app-config
   ```

---

## Übung 5.3: ConfigMap als Umgebungsvariablen

### Aufgabe

Verwende ConfigMap-Werte als Umgebungsvariablen.

### Schritte

1. **Erstelle `env-pod.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: configmap-env-pod
   spec:
     containers:
     - name: app
       image: busybox
       command: ["/bin/sh", "-c", "env | grep -E 'APP_|LOG_|DB_' && sleep 3600"]
       env:
       # Einzelne Keys
       - name: APP_NAME
         valueFrom:
           configMapKeyRef:
             name: app-settings
             key: APP_NAME
       - name: LOG_LEVEL
         valueFrom:
           configMapKeyRef:
             name: app-settings
             key: LOG_LEVEL

       # Alle Keys aus anderer ConfigMap
       envFrom:
       - configMapRef:
           name: app-config
         prefix: DB_   # Optional: Prefix hinzufügen
   ```

2. **Pod erstellen und Logs prüfen**

   ```bash
   kubectl apply -f env-pod.yaml
   kubectl logs configmap-env-pod
   ```

---

## Übung 5.4: ConfigMap als Volume

### Aufgabe

Mounte eine ConfigMap als Dateien im Container.

### Schritte

1. **Erstelle `volume-pod.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: configmap-volume-pod
   spec:
     containers:
     - name: nginx
       image: nginx
       volumeMounts:
       - name: config-volume
         mountPath: /etc/nginx/conf.d
         readOnly: true

     volumes:
     - name: config-volume
       configMap:
         name: app-config
         items:
         - key: nginx.conf
           path: default.conf
   ```

2. **Pod erstellen und prüfen**

   ```bash
   kubectl apply -f volume-pod.yaml
   kubectl exec configmap-volume-pod -- cat /etc/nginx/conf.d/default.conf
   ```

---

## Übung 5.5: Secrets erstellen

### Aufgabe

Erstelle Secrets für sensible Daten.

### Schritte

1. **Secret imperativ erstellen**

   ```bash
   kubectl create secret generic db-credentials \
     --from-literal=username=dbadmin \
     --from-literal=password=SuperGeheim123!

   kubectl get secret db-credentials -o yaml
   ```

2. **Secret deklarativ mit stringData**

   ```yaml
   # secret.yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: api-keys
   type: Opaque
   stringData:
     api_key: "sk-1234567890abcdef"
     api_secret: "geheimer-api-schluessel"
   ```

   ```bash
   kubectl apply -f secret.yaml
   ```

3. **Secret-Werte anzeigen (Base64 dekodieren)**

   ```bash
   kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
   ```

---

## Übung 5.6: Secrets verwenden

### Aufgabe

Verwende Secrets in einem Pod.

### Schritte

1. **Erstelle `secret-pod.yaml`**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: secret-pod
   spec:
     containers:
     - name: app
       image: busybox
       command: ["/bin/sh", "-c"]
       args:
         - echo "Username: $DB_USER";
           echo "Password: $DB_PASS";
           echo "Files in /secrets:";
           ls -la /secrets;
           sleep 3600

       env:
       - name: DB_USER
         valueFrom:
           secretKeyRef:
             name: db-credentials
             key: username
       - name: DB_PASS
         valueFrom:
           secretKeyRef:
             name: db-credentials
             key: password

       volumeMounts:
       - name: secret-volume
         mountPath: /secrets
         readOnly: true

     volumes:
     - name: secret-volume
       secret:
         secretName: api-keys
   ```

2. **Pod erstellen und prüfen**

   ```bash
   kubectl apply -f secret-pod.yaml
   kubectl logs secret-pod
   kubectl exec secret-pod -- cat /secrets/api_key
   ```

---

## Übung 5.7: Persistent Volume & Claim

### Aufgabe

Erstelle ein Persistent Volume und verwende es.

### Schritte

1. **Erstelle `pv-pvc.yaml`**

   ```yaml
   # Persistent Volume
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: local-pv
   spec:
     capacity:
       storage: 1Gi
     accessModes:
       - ReadWriteOnce
     persistentVolumeReclaimPolicy: Retain
     storageClassName: manual
     hostPath:
       path: /data/pv-test

   ---
   # Persistent Volume Claim
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: local-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 500Mi
     storageClassName: manual
   ```

2. **PV und PVC erstellen**

   ```bash
   kubectl apply -f pv-pvc.yaml
   kubectl get pv
   kubectl get pvc
   ```

3. **Pod mit PVC erstellen**

   ```yaml
   # pvc-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pvc-pod
   spec:
     containers:
     - name: app
       image: nginx
       volumeMounts:
       - name: data
         mountPath: /usr/share/nginx/html

     volumes:
     - name: data
       persistentVolumeClaim:
         claimName: local-pvc
   ```

   ```bash
   kubectl apply -f pvc-pod.yaml
   ```

4. **Daten schreiben**

   ```bash
   kubectl exec pvc-pod -- sh -c 'echo "Persistente Daten!" > /usr/share/nginx/html/index.html'
   kubectl exec pvc-pod -- cat /usr/share/nginx/html/index.html
   ```

5. **Pod löschen und neu erstellen**

   ```bash
   kubectl delete pod pvc-pod
   kubectl apply -f pvc-pod.yaml
   kubectl exec pvc-pod -- cat /usr/share/nginx/html/index.html
   ```

### Fragen

- Sind die Daten noch da?
- Was passiert mit dem PV, wenn du den PVC löschst?

---

## Übung 5.8: Dynamische Provisionierung

### Aufgabe

Nutze dynamische Storage-Provisionierung (wenn verfügbar).

### Schritte

1. **Verfügbare Storage Classes prüfen**

   ```bash
   kubectl get storageclasses
   ```

2. **PVC mit StorageClass erstellen**

   ```yaml
   # dynamic-pvc.yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: dynamic-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
     storageClassName: standard  # Name der StorageClass
   ```

3. **PVC und automatisch erstelltes PV prüfen**

   ```bash
   kubectl apply -f dynamic-pvc.yaml
   kubectl get pvc dynamic-pvc
   kubectl get pv
   ```

---

## Übung 5.9: ConfigMap-Update

### Aufgabe

Ändere eine ConfigMap und beobachte das Verhalten.

### Schritte

1. **ConfigMap aktualisieren**

   ```bash
   kubectl edit configmap app-config
   # Ändere einen Wert, z.B. feature.new_ui: "false"
   ```

2. **Volume-gemountete Datei prüfen**

   ```bash
   # Nach kurzer Zeit (~1 Minute) wird der Wert aktualisiert
   kubectl exec configmap-volume-pod -- cat /etc/nginx/conf.d/default.conf
   ```

3. **Umgebungsvariablen prüfen**

   ```bash
   kubectl exec configmap-env-pod -- env | grep DB_
   # Umgebungsvariablen werden NICHT aktualisiert!
   ```

### Wichtig

- Volume-Mounts werden aktualisiert (mit Verzögerung)
- Umgebungsvariablen benötigen Pod-Neustart

---

## Übung 5.10: TLS Secret für Ingress

### Aufgabe

Erstelle ein TLS-Secret für HTTPS.

### Schritte

1. **Self-Signed Zertifikat erstellen**

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout tls.key -out tls.crt \
     -subj "/CN=myapp.local"
   ```

2. **TLS Secret erstellen**

   ```bash
   kubectl create secret tls myapp-tls \
     --cert=tls.crt \
     --key=tls.key

   kubectl describe secret myapp-tls
   ```

3. **Secret in Ingress verwenden (optional)**

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: tls-ingress
   spec:
     tls:
     - hosts:
       - myapp.local
       secretName: myapp-tls
     rules:
     - host: myapp.local
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

## Aufräumen

```bash
kubectl delete pod shared-volume-pod configmap-env-pod configmap-volume-pod secret-pod pvc-pod
kubectl delete configmap app-settings app-config
kubectl delete secret db-credentials api-keys myapp-tls
kubectl delete pvc local-pvc dynamic-pvc
kubectl delete pv local-pv
rm tls.key tls.crt
```

---

## Checkliste Tag 5

- [ ] Ich verstehe emptyDir und wann man es verwendet
- [ ] Ich kann ConfigMaps erstellen (imperativ und deklarativ)
- [ ] Ich kann ConfigMaps als Env-Vars und Volumes nutzen
- [ ] Ich kann Secrets erstellen und verwenden
- [ ] Ich verstehe den Unterschied zwischen ConfigMap und Secret
- [ ] Ich verstehe PV und PVC
- [ ] Ich kann Daten persistent speichern
- [ ] Ich weiß, dass Env-Vars einen Pod-Neustart benötigen
