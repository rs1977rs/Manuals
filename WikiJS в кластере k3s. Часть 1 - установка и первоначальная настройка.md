## Поднимаем Wiki.js в кластере k3s

Wiki.js - это wiki-движок, работающий на Node.js и написанный на JavaScript.
В стеке технологий он относится к инструментам категории "Документация и обслуживание".

---
### Prerequisites

1. Работающий кластер k3s с установленным ingress-контроллером Traefik

2. План установки:
  * Создание Namespace.
  * Развёртывание PostgreSQL.
  * Создание сервиса для PostgreSQL.
  * Создание файла конфигурации для Wiki.js.
  * Развёртывание приложения Wiki.js.
  * Создание Ingress.

3. Полезные ссылки
  * [Официальный сайт Wiki.js](https://js.wiki/)
  * [Wiki.js на GitHub](https://github.com/Requarks/wiki)
  * [Установка PostgreSQL в кластере k3s](https://portworx.com/blog/run-ha-postgresql-rancher-kubernetes-engine/)
  * [Деплой Wiki.js в Kubernetes](https://medium.com/swlh/deploy-wiki-js-on-kubernetes-686cec78b29)
  * [Использование Ingress в Kubernetes (Официальный сайт)](https://kubernetes.io/docs/concepts/services-networking/ingress/)
  * [Как использовать Ingress и Cert-manager](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes-ru)
  * [Cert-manager. Инструкция с официального сайта](https://cert-manager.io/docs/concepts/issuer/)
  * [SSL-сертификаты от Let's Encrypt с cert-manager в Kubernetes](https://habr.com/ru/company/flant/blog/496936/)
  * [Кодирование в base64 on-line](http://base64.ru/)

---
### Установка
#### Создаём Namespace
Подключаемся к кластеру k3s и создаем неймспейс:
```
$ kubectl create namespace wiki
```
или сохраняем в файл `wiki-ns.yaml`:
```
apiVersion: v1
kind: Namespace
metadata:
  name: wiki
  labels:
    name: wiki
```
и применяем его:
```
$ kubectl apply -f wiki-ns.yaml
namespace/wiki created
```
Можно запросить существующие неймспейсы:
```
$ kubectl get ns
NAME              STATUS   AGE
default           Active   19d
kube-system       Active   19d
kube-public       Active   19d
kube-node-lease   Active   19d
lens-metrics      Active   19d
cert-manager      Active   11d
wiki              Active   3s
```

#### Развёртываем PostgreSQL
* Создаем PVC для PostgreSQL

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgres-pvc
  namespace: wiki
  labels:
    app: postgres
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```
`namespace: wiki` - БД должна находиться в том же неймспейсе
`storageClassName: local-path` - у нас однонодовый тестовый кластер, подключаемые тома будут находиться в файловой системе ноды, если нод больше одной - такой способ хранения не подойдёт, в этом случае нужно будет с помощью labels указывать на какой ноде располагать под с БД

Сохраняем в файл `postgres-pvc.yaml` и применяем его
```
$ kubectl apply -f postgres-pvc.yaml
persistentvolumeclaim/postgres-pvc created

$ kubectl -n wiki get pvc
NAMESPACE      NAME                STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
lens-metrics   data-prometheus-0   Bound     pvc-92417c96-e7a0-4327-be00-470dd19eaed7   20G        RWO            local-path     19d
wiki           postgres-pvc        Pending                                                                        local-path     11s
```
Статус PVC пока ещё Pending, потому что не создан под, который будет использовать этот том.

* Создаём секрет для PostgreSQL

Сначала кодируем пароль в base64 и потом подставляем его значение в поле `data`.
Сохраняем в файл `postgress-pass.yaml`:
```
kind: Secret
apiVersion: v1
metadata:
  name: postgres-pass
  namespace: wiki
data:
  postgres-pass: bXlwb3N0Z3Jlc3Bhc3M=
type: Opaque
```
и применяем его:
```
$ kubectl apply -f postgres-pass.yaml

```
```
$ kubectl get secret postgres-pass -n wiki
NAME            TYPE     DATA   AGE
postgres-pass   Opaque   1      7s
```
```
$ kubectl get secret postgres-pass -n wiki -o yaml
apiVersion: v1
data:
  postgres-pass: bXlwb3N0Z3Jlc3Bhc3M=
kind: Secret
metadata:
  creationTimestamp: "2022-01-08T15:51:26Z"
  name: postgres-pass
  namespace: wiki
  resourceVersion: "1439111"
  uid: dc22bde7-8643-4b50-a163-fab9eb0b4d85
type: Opaque
```
где
```
data:
  postgres-pass: XXXXXXXXX
```
это наш ключ, который нужно прописать в ENV деплоймента PostgreSQL

* Создаём файл деплоймента для развёртывания PostgreSQL:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: wiki
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
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
        image: postgres:12
        imagePullPolicy: "Always"
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: wiki
        - name: POSTGRES_USER
          value: wiki
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-pass
              key: postgres-pass
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgredb
      volumes:
      - name: postgredb
        persistentVolumeClaim:
          claimName: postgres-pvc
```
и применяем его, чтобы создать под с PostgreSQL:
```
$ kubectl apply -f postgres.yaml
deployment.apps/postgres created
```
Можно было бы все переменные окружения сложить в один configmap, как это будет позже сделано при деплойменте приложения, но пусть в одном мануале будут разные варианты.

Смотрим, что получилось:
```
$ kubectl -n wiki get deployment
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
postgres   1/1     1            1           42s
```
```
$ kubectl -n wiki get pod
NAME                        READY   STATUS    RESTARTS   AGE
postgres-5d5c87bd86-w8wnj   1/1     Running   0          51s
```
```
$ kubectl -n wiki get pvc
NAMESPACE      NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
lens-metrics   data-prometheus-0   Bound    pvc-92417c96-e7a0-4327-be00-470dd19eaed7   20G        RWO            local-path     19d
wiki           postgres-pvc        Bound    pvc-142a20be-0efc-44ca-b748-9c2511bd2917   5Gi        RWO            local-path     52m
```
Видим, что статус тома сменился на Bound. Особенность кластера k3s в том, что не нужно было создавать отдельный ресурс PV. Как только под с PostgreSQL запросил себе место под PVC, кластер автоматически его нарезал.
```
$ kubectl get pv -A
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS   REASON   AGE
pvc-92417c96-e7a0-4327-be00-470dd19eaed7   20G        RWO            Delete           Bound    lens-metrics/data-prometheus-0   local-path              19d
pvc-142a20be-0efc-44ca-b748-9c2511bd2917   5Gi        RWO            Delete           Bound    wiki/postgres-pvc                local-path              56s
```
#### Создаём сервис для PostgreSQL
Для общения подов между собой внутри кластера есть удобная сущность - Service, которая позволяет абстрагироваться от IP-адреса пода, который при каждом пересоздании будет меняться.

Создаём файл с описанием сервиса `postgres-pvc.yaml`:
```
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: wiki
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
    protocol: TCP
    targetPort: 5432
```
и применяем его:
```
$ kubectl apply -f postgres-svc.yaml
service/postgres created
```
```
$ kubectl -n wiki get svc
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
postgres   ClusterIP   10.43.156.144   <none>        5432/TCP   7s
```
#### Создаём config-map для приложения Wiki.js
Создаём файл с описанием конфига приложения Wiki.js `wiki-config.yaml`:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: wiki-config
  namespace: wiki
data:
  DB_HOST: "postgres.wiki.svc"
  DB_PORT: "5432"
  DB_USER: "wiki"
  DB_TYPE: "postgres"
  DB_NAME: "wiki"
```
и применяем его:
```
$ kubectl apply -f wiki-config.yaml
configmap/wiki-config created
```
#### Развёртываем приложение Wiki.js
* Создаем PVC для Wiki.js

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: wiki-data-pvc
  namespace: wiki
  labels:
    app: wiki
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 7Gi
```
и применяем:
```
$ kubectl apply -f wiki-data-pvc.yaml
persistentvolumeclaim/wiki-data-pvc created
```
```
$ kubectl -n wiki get pvc
NAME            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgres-pvc    Bound     pvc-142a20be-0efc-44ca-b748-9c2511bd2917   5Gi        RWO            local-path     99m
wiki-data-pvc   Pending                                                                        local-path     10s
```
* Создаём файл деплоймента для развёртывания Wiki.js:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: wiki
  name: wiki
  namespace: wiki
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wiki  
  template:
    metadata:
      labels:
        app: wiki
    spec:
      securityContext:
        fsGroup: 1000            
      volumes:
        - name: wiki-data
          persistentVolumeClaim:
            claimName: wiki-data-pvc            
      containers:
        - name: wiki
          image: requarks/wiki:2
          env:
          - name: DB_HOST
            valueFrom:
              configMapKeyRef:
                key: DB_HOST
                name: wiki-config
          - name: DB_PORT
            valueFrom:
              configMapKeyRef:
                key: DB_PORT
                name: wiki-config
          - name: DB_USER
            valueFrom:
              configMapKeyRef:
                key: DB_USER
                name: wiki-config
          - name: DB_TYPE
            valueFrom:
              configMapKeyRef:
                key: DB_TYPE
                name: wiki-config  
          - name: DB_NAME
            valueFrom:
              configMapKeyRef:
                key: DB_NAME
                name: wiki-config  
          - name: DB_PASS
            valueFrom:
              secretKeyRef:
                name: postgres-pass
                key: postgres-pass
          volumeMounts:
            - name: wiki-data
              readOnly: false
              mountPath: /wiki/data
          ports:
          - containerPort: 3000
            protocol: TCP
```
и применяем его:
```
$ kubectl apply -f wiki-deployment.yaml
deployment.apps/wiki created
```
Под с приложением создался:
```
$ kubectl -n wiki get pod
NAME                        READY   STATUS    RESTARTS   AGE
postgres-5d5c87bd86-w8wnj   1/1     Running   0          50m
wiki-7b786d9465-twgh7       1/1     Running   0          6s
```
том к поду присоединился:
```
$ kubectl -n wiki get pvc
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgres-pvc    Bound    pvc-142a20be-0efc-44ca-b748-9c2511bd2917   5Gi        RWO            local-path     103m
wiki-data-pvc   Bound    pvc-85a5143a-b7d6-44b6-9e0b-d569027d03e2   7Gi        RWO            local-path     3m48s
```
Можно зайти в под с Wiki.js и посмотреть логи:
```
$ kubectl -n wiki logs wiki-7b786d9465-twgh7
Loading configuration from /wiki/config.yml... OK
2022-01-08T17:27:55.734Z [MASTER] info: =======================================
2022-01-08T17:27:55.736Z [MASTER] info: = Wiki.js 2.5.260 =====================
2022-01-08T17:27:55.736Z [MASTER] info: =======================================
2022-01-08T17:27:55.736Z [MASTER] info: Initializing...
2022-01-08T17:27:56.357Z [MASTER] info: Using database driver pg for postgres [ OK ]
2022-01-08T17:27:56.361Z [MASTER] info: Connecting to database...
2022-01-08T17:27:56.390Z [MASTER] info: Database Connection Successful [ OK ]
2022-01-08T17:27:56.723Z [MASTER] warn: DB Configuration is empty or incomplete. Switching to Setup mode...
2022-01-08T17:27:56.723Z [MASTER] info: Starting setup wizard...
2022-01-08T17:27:56.865Z [MASTER] info: Starting HTTP server on port 3000...
2022-01-08T17:27:56.866Z [MASTER] info: HTTP Server on port: [ 3000 ]
2022-01-08T17:27:56.869Z [MASTER] info: HTTP Server: [ RUNNING ]
2022-01-08T17:27:56.869Z [MASTER] info: 🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻🔻
2022-01-08T17:27:56.869Z [MASTER] info:
2022-01-08T17:27:56.869Z [MASTER] info: Browse to http://YOUR-SERVER-IP:3000/ to complete setup!
2022-01-08T17:27:56.869Z [MASTER] info:
2022-01-08T17:27:56.869Z [MASTER] info: 🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺🔺
```
#### Создаём Ingress, для доступа к приложению извне
Прежде, чем заходить по адресу и донастраивать Wiki.js, создадим Service и Ingress.
* Создаём файл для деплоя сервиса `wiki-service.yaml`, который будет прокидывать трафик с 80-го порта на 3000, на котором работает приложение Wiki.js в поде:

```
apiVersion: v1
kind: Service
metadata:
  name: wiki
  namespace: wiki
spec:
  type: ClusterIP
  selector:
    app: wiki
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
```
применяем его:
```
$ kubectl apply -f wiki-service.yaml
service/wiki created
```
и видим, что у нас теперь два сервиса:
```
$ kubectl -n wiki get svc
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
postgres   ClusterIP   10.43.156.144   <none>        5432/TCP   66m
wiki       ClusterIP   10.43.169.160   <none>        80/TCP     2s
```
* Создаём файл для деплоя сущности Ingress - аналога Route в OpenShift.

В качестве маршрутизатора входящего трафика у нас используется Ingress-контроллер Traefik, поэтому Ingress будет настраиваться учитывая это. Сразу сделаем возможным доступ к сайту через https с выдачей и автоматическим продлением сертификата через Letsencrypt. Я опущу процесс поднятия Ingress-контроллера, cert-manager'а и создания фиктивного сертификата в стейджевом окружении, сразу сделаем для прода.

Создаем секрет в файле `wiki-tls-secret.yaml`:
```
apiVersion: v1
kind: Secret
metadata:
  name: wiki-tls
  namespace: wiki
type: Opaque
stringData:
  api-token: B0vT7YXHpRNDf-M3-y1f0DwXVx5dUfNhEblE-iGz
```
токен можно получить у своего DNS-провайдера.

Создаем Issuer в файле `wiki-issuer.yaml`:
```
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: wiki-issuer
  namespace: wiki
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - dns01:
        cloudflare:
          email: <your-email@on-cloudflare>
          apiTokenSecretRef:
            name: wiki-tls
            key: api-token
```
И, наконец, создаём Ingrress в файле `wiki-ingress.yaml` :
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wiki-tls
  namespace: wiki
  annotations:
    kubernetes.io/ingress.class: "traefik"
    cert-manager.io/issuer: "wiki-issuer"
spec:
  rules:
    - host: <your.host.name>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wiki
                port:
                  number: 80
  tls:
  - hosts:
    - <your.host.name>
    secretName: wiki-tls
```      
Последовательно применяем их:
```
$ kubectl apply -f wiki-tls-secret.yaml
secret/wiki-tls created
```
```
$ kubectl -n wiki get secret
NAME                  TYPE                                  DATA   AGE
default-token-slgq7   kubernetes.io/service-account-token   3      178m
postgres-pass         Opaque                                1      125m
wiki-tls              Opaque                                1      12s
```
```
$ kubectl apply -f wiki-issuer.yaml
issuer.cert-manager.io/wiki-issuer created
```
```
$ kubectl -n wiki get issuer
NAME          READY   AGE
wiki-issuer   True    6s
```
```
$ kubectl apply -f wiki-ingress.yaml
ingress.networking.k8s.io/wiki-tls created
```
```
$ kubectl -n wiki get ingress
NAME       CLASS    HOSTS                 ADDRESS     PORTS     AGE
wiki-tls   <none>   wikijs.redstar.name   10.0.0.38   80, 443   9s
```
Можно посмотреть описание созданного ингресса:
```
$ kubectl -n wiki describe ingress wiki-tls
Name:             wiki-tls
Namespace:        wiki
Address:          10.0.0.38
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
TLS:
  wiki-tls terminates wikijs.redstar.name
Rules:
  Host                 Path  Backends
  ----                 ----  --------
  wikijs.redstar.name  
                       /   wiki:80 (<none>)
Annotations:           cert-manager.io/issuer: wiki-issuer
                       kubernetes.io/ingress.class: traefik
Events:
  Type    Reason             Age    From          Message
  ----    ------             ----   ----          -------
  Normal  CreateCertificate  2m21s  cert-manager  Successfully created Certificate "wiki-tls"
```
и сертификата:
```
$ kubectl -n wiki describe Certificate wiki-tls
Name:         wiki-tls
Namespace:    wiki
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
........
<тут много метаданных>
........
Spec:
  Dns Names:
    wikijs.redstar.name
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       Issuer
    Name:       wiki-issuer
  Secret Name:  wiki-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2022-01-08T17:58:58Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2022-04-08T16:58:56Z
  Not Before:              2022-01-08T16:58:57Z
  Renewal Time:            2022-03-09T16:58:56Z
  Revision:                1
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    18m   cert-manager  Issuing certificate as Secret does not contain a private key
  Normal  Generated  18m   cert-manager  Stored new private key in temporary Secret resource "wiki-tls-zzsmk"
  Normal  Requested  18m   cert-manager  Created new CertificateRequest resource "wiki-tls-wcjjg"
  Normal  Issuing    17m   cert-manager  The certificate has been successfully issued
```
После чего зайти в браузер и продолжить настройку Wiki.js.

После несложной настройки в браузере, в логах можно увидеть:
```
2022-01-11T12:19:12.561Z [MASTER] info: Creating data directories...
2022-01-11T12:19:12.563Z [MASTER] info: Generating certificates...
2022-01-11T12:19:12.811Z [MASTER] info: Persisting config to DB...
2022-01-11T12:19:13.109Z [MASTER] info: Installing default locale...
2022-01-11T12:19:13.119Z [MASTER] info: Creating default groups...
2022-01-11T12:19:13.156Z [MASTER] info: Loaded 6 new editors: [ OK ]
2022-01-11T12:19:13.180Z [MASTER] info: Loaded 12 new loggers: [ OK ]
........
<много разного вывода>
........
Loading configuration from /wiki/config.yml... OK
2022-01-11T12:19:17.421Z [JOB] info: Rebuilding page tree...
2022-01-11T12:19:17.476Z [MASTER] info: Fetching latest updates from Graph endpoint: [ COMPLETED ]
2022-01-11T12:19:17.888Z [MASTER] info: Pulled latest locale updates for English from Graph endpoint: [ COMPLETED ]
2022-01-11T12:19:17.892Z [MASTER] info: Syncing locales with Graph endpoint: [ COMPLETED ]
2022-01-11T12:19:18.088Z [JOB] info: Using database driver pg for postgres [ OK ]
2022-01-11T12:19:18.146Z [JOB] info: Rebuilding page tree: [ COMPLETED ]
```
[В следующий раз рассмотрим процесс бэкапа и восстановления из него.](http://gitea.redstar.name/RedStar/Manuals/src/branch/master/Kubernetes/WikiJS%20%d0%b2%20%d0%ba%d0%bb%d0%b0%d1%81%d1%82%d0%b5%d1%80%d0%b5%20k3s.%20%d0%a7%d0%b0%d1%81%d1%82%d1%8c%202%20-%20%d0%b1%d1%8d%d0%ba%d0%b0%d0%bf%20%d0%b8%20%d0%b2%d0%be%d1%81%d1%81%d1%82%d0%b0%d0%bd%d0%be%d0%b2%d0%bb%d0%b5%d0%bd%d0%b8%d0%b5.md)
