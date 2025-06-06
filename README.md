# Пошаговая инструкция по установке сервиса регистрации зюзюбликов в Kubernetes
Все файлы необходимо создавать со всеми отступами, которые есть в инструкции.
## Предварительные требования

- Кластер kubernetes в виде minikube
- Терминал с оболочкой bash и текстовым редактором
- Установленный kubectl и Ingress Controller

## Порядок установки
**1. Открыть терминал с оболочкой `bash`**

**2. Создаем каталог `k8s_manifest`**
```
mkdir k8s_manifest
```
**3. Переходим в каталог `k8_manifest`, все работы будем производить из этого каталога.**
```
cd mkdir k8s_manifest
```
**4. Создадим файл `namespace.yaml`**
(манифест создания неймспейса)
```
apiVersion: v1
kind: Namespace
metadata:
  name: zouzoublique
```
**5. Создаем файл `configmap-postgresql.yaml`** 
(Конфигмапа, нужная для PostgreSQL с данными для созданной БД)
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
**6. Создаем файл `configmap-zouzoublique-config.yaml`** 
(Конфигмапа, нужная для сервиса zouzoublique с данными для подлключения к postgresql)
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
**7. Создаем файл `deployment.yaml`** (Deployment для нашего сервиса zouzoublique)
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
**8. Создаем файл `ingress.yaml`** (ресурс, позволяющий при наличии ingress-controller достучаться до сервиса по указанному DNS имени вне кластера)
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
**9. Создаем файл `pvc.yaml`**
(Ресурс PersistentVolumeClaim, позволяющий создать постоянное хранилище данных на хосте. Необходим для хранения данных postgresql)
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
**10. Создаем файл `statefulset.yaml`** (контролллер StatefulSet для работы postgresql и сохранения состояния данных)
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
**11. Создаем файл `svc-app.yaml`** (сервис для проксирования запросов до подов zouzoublique)
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
**12. Создаем файл `svc-postgres.yaml`** (сервис для проксирования запросов до подов postgresql)
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
**13. Проверим статус кластера minikub** при помощи команды:
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

**14. Проверим, что мы подключены к нужному кластеру** при помощи команды
```
kubectl config get-contexts
```
при необходимости необходимо переключить контекст

```
kubectl config use-context minikube
```

**15. Активируем и создадим ресурсы ingress-controller**
```
minikube addons enable ingress
```
**16. Установим все необходимые ресурсы и сервисы**
```
kubectl apply -f ./k8s_manifests
```
**17. Добавим доменное имя**

Для того, чтобы dns имя было связано с нужным ip адресом необходимо прописать в файле `/etc/hosts` следующее:
``` 
<IP адрес развернутого minikube> zouzoublique.example.com
```
IP адрес развернутого minikube можно узнать при помощи команды
```
minikube ip
```
**18. Проверим работоспособность сервиса**

- Убедимся, что сервис доступен
```
curl -X GET http://zouzoublique.example.com/healthz
```
- Убедимся, что мы можем получить пустой список зюзюбликов
```
curl -X GET http://zouzoublique.example.com/v1/zouzoubliques
```

- Убедимся, что мы можем добавить зюзюблика
```
curl -X POST http://zouzoublique.example.com/v1/zouzoubliques \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "name": "Зюзюблик-3000",
    "type": 42
  }'
```
- Проверим, что зузяблик добавился
```
curl -X GET http://zouzoublique.example.com/v1/zouzoubliques
```
