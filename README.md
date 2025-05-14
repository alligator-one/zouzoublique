# Пошаговая инструкция по установке сервиса регистрации зюзюбликов в Kubernetes
Все файлы необходимо создавать со всеми отступами, которые есть в инструкции.
## Предварительные требования

- Кластер kubernetes в виде minikube
- Терминал с оболочкой bash и текстовым редактором
- Установленный kubectl

## Порядок установки
1. Открыть терминал с оболочкой `bash`

2. Создаем каталог k8s_manifest
```
mkdir k8s_manifest
```
3. Переходим в каталог k8_manifest, все работы будем производить из этого каталога.
```
cd mkdir k8s_manifest
```
3. Создаем файл configmap-postgresql.yaml (Конфигмапа, нужная для PostgreSQL с данными для созданной БД)
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
4. Создаем файл configmap-zouzoublique-config.yaml (Конфигмапа, нужная для сервиса zouzoublique с данными для подлключения к postgresql)
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
5. Создаем файл deployment.yaml (Deployment для нашего сервиса zouzoublique)
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
6. Создаем файл ingress.yaml (ресурс, позволяющий при наличии ingress-controller достучаться до сервиса по указанному DNS имения вне кластера)
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
7. Создаем файл pvc.yaml (Ресурс PersistentVolumeClaim, позволяющий создать постоянное хранилище данных на хосте. Необходим для хранения данных postgresql

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
8. Создаем файл statefulset.yaml (ресурса StatefulSet для работы postgresql и сохранения состояния данных)
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
9. Создаем файл svc-app.yaml (сервис для проксирования запросов до подов zouzoublique)
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
10. Создаем файл svc-postgres.yaml (сервис для проксирования запросов до подов postgresql)
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
11. Проверим статус кластера minikube командой
```
minikube status
```
Команда должна вывести статус, что minikube запущен. Ожидаемый вид:
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```
Если кластер остановлен, то запустим его командой `minikube start`
12. Проверим, что мы подключены к нужному кластеру командой
```
kubectl config get-contexts
```
13. Активируем и создадим ресурсы ingress-controller
```
minikube addons enable ingress
```
14. 
