# zouzoublique
Все файлы необходимо создавать со всеми отступами, которые есть в инструкции.
Необходимо убедиться, что в операционной системе есть текстовый редактор. В инструкции используется текстовый редактор nano, но можно использовать любой другой.

1. Создаем каталог k8s_manifest

```
mkdir k8s_manifest
```
2. Переходим в каталог k8_manifest
```
cd mkdir k8s_manifest
```
3. Создаем файл configmap-postgresql.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: zouzoublique
data:
  POSTGRES_DB: postgres
  POSTGRES_PASSWORD: postgres
  POSTGRES_USER: postgres
```  
4. Создаем файл configmap-zouzoublique-config.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: zouzoublique-config
  namespace: zouzoublique
data:
  config.ini: |
    [main]
    database_url = postgres://postgres:postgres@postgres:5432/postgres?sslmode=disable
    port = 3000
```	
5. Создаем файл deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zouzoublique-api
  namespace: zouzoublique
spec:
  replicas: 3
  selector:
    matchLabels:
      app: zouzoublique-api
  template:
    metadata:
      labels:
        app: zouzoublique-api
    spec:
      containers:
        - name: zouzoublique-api
          image: docker.io/operations42/zouzoublique-api:v1
          ports:
          - containerPort: 3000
          volumeMounts:
          - name: config-volume
            mountPath: /etc/zouzoublique-api/config.ini
            subPath: config.ini
      volumes:
      - name: config-volume
        configMap:
          name: zouzoublique-config	
```
6. Создаем файл ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: zouzoublique-ingress
  namespace: zouzoublique
spec:
  rules:
  - host: zouzoublique.example.com
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: zouzoublique-service
              port:
                number: 80
```
6. Создаем файл pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: zouzoublique
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi				
```	  
7. Создаем файл statefulset.yaml
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: zouzoublique
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        envFrom:
        - configMapRef:
            name: postgres-config
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```
8. Создаем файл svc-app.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: zouzoublique-service
  namespace: zouzoublique
spec:
  selector:
    app: zouzoublique-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```	  
9. Создаем файл svc-postgres.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: zouzoublique
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432 
```	  
10. Запускаем сервис:
```
kubectl apply -f ./k8s_manifests
```
