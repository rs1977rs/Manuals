## Поднимаем Rocket.Chat в кластере k3s на архитектуре arm64

Rocket.Chat — это мессенджер с открытым исходным кодом, который поддерживает групповые чаты, обмен файлами, видеоконференции, ботов и многое другое.

---
### Prerequisites

1. Работающий кластер k3s с установленным ingress-контроллером Traefik

2. План установки:
  * Создание Namespace.
  * Развёртывание MongoDB.
  * Развёртывание приложения Rocket.Chat.
  * Создание Ingress.
  * Обновление MongoDB.
  * Обновление Rocket.Chat.

3. Полезные ссылки
  * [Официальный сайт документации Rocket.Chat](https://docs.rocket.chat/)
  * [Релизы Rocket.Chat на GitHub](https://github.com/RocketChat/Rocket.Chat/releases)
  * [Dockerfile для сборки своего образа](https://github.com/RocketChat/Rocket.Chat.Embedded.arm64/blob/develop/docker/rocketchat/dockerfile)
  * [Deploy Rocket chat server using Kubernetes](https://ajorloo.medium.com/deploy-rocket-chat-server-using-kubernetes-2d6c4228853)
  * [Официальные образы на hub.docker.com](https://hub.docker.com/_/rocket-chat?tab=tags)
  * [Сборка под arm64, которую мне удалось запустить](https://hub.docker.com/r/singli/rocketchat/tags)
  * [Моя сборка под arm64 на hub.docker.com](https://hub.docker.com/repository/docker/rs1977rs/rocketchat)
  * [Использование Ingress в Kubernetes (Официальный сайт)](https://kubernetes.io/docs/concepts/services-networking/ingress/)
  * [Как использовать Ingress и Cert-manager](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes-ru)
  * [Cert-manager. Инструкция с официального сайта](https://cert-manager.io/docs/concepts/issuer/)
  * [SSL-сертификаты от Let's Encrypt с cert-manager в Kubernetes](https://habr.com/ru/company/flant/blog/496936/)
  * [Кодирование в base64 on-line](http://base64.ru/)

---
### Установка
#### Создаём Namespace
Подключаемся к кластеру k3s и создаем файл `rocket-ns.yaml`:
```
apiVersion: v1
kind: Namespace
metadata:
  name: rocket-chat
  labels:
    name: rocket-chat
```
и применяем его:
```
$ kubectl apply -f rocket-ns.yaml
namespace/rocket-chat created
```

#### Развёртываем MongoDB
Сначала надо создать сервис для `MongoDB`. Как ни странно, он потом нигде не используется, но без него не получилось инициализировать базу.
Создаем файл `mongo-svc.yaml`:
```
apiVersion: v1
kind: Service
metadata:
  name: rocketmongo
  namespace: rocket-chat
  labels:
    app: rocketmongo
spec:
  selector:
    app: rocketmongo
  ports:
    - port: 27017
      targetPort: 27017
  clusterIP: None
```
и применяем его:
```
$ kubectl apply -f rocket-svc.yaml
service/rocketchat-server created
```

Создаем файл `mongo-deploy.yaml`:
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rocketmongo
  namespace: rocket-chat
spec:
  serviceName: rocketmongo
  replicas: 3
  selector:
    matchLabels:
      app: rocketmongo
  template:
    metadata:
      labels:
        app: rocketmongo
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: rocketmongo
          image: mongo:4.0
          command:
          - mongod
          - "--bind_ip_all"
          - "--replSet"
          - rs0
          ports:
            - containerPort: 27017
          volumeMounts:
          - name: mongo-volume
            mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongo-volume
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi
```
Важно отметить, что для развёртывания `MongoDB` используется сущность `StatefulSet`. Она позволит нам развернуть отказоустойчивый и масштабируемый кластер из трёх инстансов БД. В качестве хранилища для каждого инстанса будет автоматически создан том, заранее сущности `pvc` и `pv` создавать не надо.

Применяем созданный файл:
```
$ kubectl apply -f mongo-deploy.yaml
statefulset.apps/rocketmongo created
```
#### Настройка MognoDB
Теперь нужно настроить БД, указав, какой из инстансов будет мастером, а какие - репликами.
Получаем список подов:
```
$ kubectl -n rocket-chat get pod
NAME                                 READY   STATUS    RESTARTS   AGE
rocketmongo-2                        1/1     Running   0          153m
rocketmongo-1                        1/1     Running   0          153m
rocketmongo-0                        1/1     Running   0          153m
```
Подключаемся в консоль к `rocketmongo-0` и выполняем команды для инициации кластера:
```
$ kubectl -n rocket-chat exec -ti rocketmongo-0 -- mongo
> rs.initiate()
{
    "info2" : "no configuration specified. Using a default configuration for the set",
    "me" : "rocketmongo-0:27017",
    "ok" : 1,
    "operationTime" : Timestamp(1609450853, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1609450853, 1),
        "signature" : {
           "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
           "keyId" : NumberLong(0)
        }
    }
}
```
и его настройки:
```
rs0:SECONDARY>  var config = rs.conf()
rs0:PRIMARY>  config.members[0].host="rocketmongo-0.rocketmongo:27017"
rocketmongo-0.rocketmongo:27017
rs0:PRIMARY> rs.reconfig(config)
{
    "ok" : 1,
    "operationTime" : Timestamp(1609451525, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1609451525, 1),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```
И добавляем по очереди реплики:
```
> rs.add("rocketmongo-1.rocketmongo:27017")
> rs.add("rocketmongo-2.rocketmongo:27017")
```
После чего можно проверить статус кластера:
```
rs0:PRIMARY> rs.status()
```
и выйти
```
rs0:PRIMARY>  exit
bye
```

#### Деплоим приложение Rocket.Chat
Создаём файл `rocket-deploy.yaml`:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rocketchat-server
  namespace: rocket-chat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rocketchat-server
  template:
    metadata:
      labels:
        app: rocketchat-server
    spec:
      containers:
        - name: rocketchat-server
          image: singli/rocketchat:arm64-3.2.2
          ports:
            - containerPort: 3000
          env:
            - name: MONGO_URL
              value: mongodb://rocketmongo-0.rocketmongo:27017/rocketchat
            - name: MONGO_OPLOG_URL
              value: mongodb://rocketmongo-0.rocketmongo:27017/local?replSet=rs0
          imagePullPolicy: Always
```
Основная проблема (и интерес) заключалась в том, что на официальном сайте Rocket.Chat я не нашел ничего про то, как его развёртывать на архитектуре arm64. Ну, а так как мой кластер в облаке Oracle именно на этих ядрах, то пошел искать готовые образы на `hub.docker.com`. Как выяснилось позже, нашел не самое свежее, но чтоб запустить - сойдёт, потом обновим.

Разворачиваем:
```
$ kubectl apply -f rocket-deploy.yaml
deployment.apps/rocketchat-server created
```
Через какое-то время можно зайти в логи пода и убедиться что приложение работает.

#### Настройка Ingress
Настройка `ingress` с доступом по https и установкой сертификатов `Letsencrypt` аналогична его настройке для `WikiJS`.

Сначала создаем сервис в файле `rocket-svc.yaml`, чтоб в дальнейшем подключаться к нему из мира через ингресс:
```
apiVersion: v1
kind: Service
metadata:
  name: rocketchat-server
  namespace: rocket-chat
spec:
  selector:
    app: rocketchat-server
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  clusterIP: None
```
и применяем его:
```
$ kubectl apply -f rocket-svc.yaml
service/rocketchat-server created
```
Создаем секрет в файле `rocket-tls-secret.yaml`:
```
apiVersion: v1
kind: Secret
metadata:
  name: rocket-tls
  namespace: rocket-chat
type: Opaque
stringData:
  api-token: XXXXXXXXX
```
токен можно получить у своего DNS-провайдера.

Создаем Issuer в файле `rocket-issuer.yaml`:
```
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: rocket-issuer
  namespace: rocket-chat
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
            name: rocket-tls
            key: api-token
```
И, наконец, создаём Ingress в файле `rocket-ingress.yaml` :
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rocket-tls
  namespace: rocket-chat
  annotations:
    kubernetes.io/ingress.class: "traefik"
    cert-manager.io/issuer: "docs-issuer"
spec:
  rules:
    - host: <your.host.name>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rocketchat-server
                port:
                  number: 3000
  tls:
  - hosts:
    - <your.host.name>
    secretName: rocket-tls
```      
Последовательно применяем их:
```
$ kubectl apply -f rocket-tls-secret.yaml
secret/rocket-tls created

$ kubectl apply -f rocket-issuer.yaml
issuer.cert-manager.io/rocket-issuer created

$ kubectl apply -f rocket-ingress.yaml
ingress.networking.k8s.io/rocket-tls created
```
После чего зайти в браузер и продолжить настройку Rocket.Chat.

#### Обновление MongoDB
После запуска приложения `RocketChat` в его логах можно увидеть, что наша версия `MongoDB` скоро перестанет поддерживаться, поэтому иммет смысл обновить версию. С сущностью `StatefulSet` это очень просто - нужно исправить версию `MongoDB` в файле `mongo-deploy.yaml` и заново применить его. API кубернетиса увидит, что манифест изменился и по очереди задеплоит все три инстанса БД.

Открываем файл `mongo-deploy.yaml` и правим версию на `4.2`:
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rocketmongo
  namespace: rocket-chat
spec:
  serviceName: rocketmongo
  replicas: 3
  selector:
    matchLabels:
      app: rocketmongo
  template:
    metadata:
      labels:
        app: rocketmongo
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: rocketmongo
          image: mongo:4.2
```
сохраняем и применяем:
```
$ kubectl apply -f mongo-deploy.yaml
statefulset.apps/rocketmongo configured
```
Если возникнут проблемы с авторизацией в браузере, имеет смысл сначала перезапустить под с приложением `RocketChat`, если не поможет - то все три пода `MongoDB`.

#### Обновление Rocket.Chat
В общедоступном реестре образов я не нашел работающей версии новее этой. Другие образы с `hub.docker.com` у меня запускаться отказывались - постоянно вываливались какие-то ошибки. Было принято решение собрать образ самому.
Мне удалось запустить версию `4.3.3` от 28 января 2022 года. Этот релиз вышел чуть более двух недель назад от времени написания этого манула, считаю его достаточно свежим. Были и более поздние релизы, но собрать образ на их основе под архитектуру `arm64` у меня не получилось. В версии `4.4.0` используется `14.18.2` версия `Node`, а в `4.3.3` пока что `12.22.1`, возможно дело в этом.

Находим на `GitHub` нужный нам `Dockerfile` и слегка правим его:
```
FROM arm64v8/node:12.22.1-stretch as builder

# crafted and tuned by pierre@ozoux.net and sing.li@rocket.chat
MAINTAINER buildmaster@rocket.chat

RUN groupadd -r rocketchat \
&&  useradd -r -g rocketchat rocketchat \
&&  mkdir -p /app/uploads \
&&  chown rocketchat.rocketchat /app/uploads \
# `RUN mkdir ~/.gnupg
&& echo "disable-ipv6" >> ~/.gnupg/dirmngr.conf

VOLUME /app/uploads

# gpg: key 4FD08014: public key "Rocket.Chat Buildmaster <buildmaster@rocket.chat>" imported
RUN gpg --batch --keyserver keyserver.ubuntu.com --recv-keys  0E163286C20D07B9787EBE9FD7F9D0414FD08104



ENV RC_VERSION 4.1.2

WORKDIR /app

RUN apt-get update && apt-get -y install g++ build-essential
RUN apt-get update && apt-get -y upgrade
RUN apt-get install -y curl ca-certificates imagemagick --no-install-recommends
RUN apt-get install libstdc++6
RUN apt-get -y install python
RUN curl -fSL "https://releases.rocket.chat/${RC_VERSION}/download" -o rocket.chat.tgz \
&&  curl -fSL "https://releases.rocket.chat/${RC_VERSION}/asc" -o rocket.chat.tgz.asc \
&&  gpg --batch --verify rocket.chat.tgz.asc rocket.chat.tgz \
&&  tar zxvf rocket.chat.tgz \
&&  rm rocket.chat.tgz rocket.chat.tgz.asc
ADD . /app


RUN set -x \
 && cd /app/bundle/programs/server \
 && npm install \
 # Start hack for sharp...
 && rm -rf npm/node_modules/sharp \
# && rm -rf npm/node_modules/grpc \
 && npm install sharp@0.22.1 \
# && npm install grpc@1.12.2 \
 && mv node_modules/sharp npm/node_modules/sharp \
# && mv node_modules/grpc npm/node_modules/grpc \
 # End hack for sharp
 && cd npm \
 && npm rebuild bcrypt --build-from-source \
 && npm cache clear --force

FROM arm64v8/node:12.22.1-stretch

RUN groupadd -r rocketchat \
&&  useradd -r -g rocketchat rocketchat \
&&  mkdir -p /app

COPY --from=builder /app /app

RUN  mkdir -p /app/uploads \
&&   chown rocketchat.rocketchat /app/uploads

VOLUME /app/uploads

USER rocketchat

WORKDIR /app/bundle


# needs a mongo instance - defaults to container linking with alias 'mongo'
ENV DEPLOY_METHOD=docker-arm64 \
    NODE_ENV=production \
    MONGO_URL=mongodb://mongo:27017/rocketchat \
    MONGO_OPLOG_URL=mongodb://mongo:27017/local \
    HOME=/tmp \
    PORT=3000 \
    ROOT_URL=http://localhost:3000 \
    Accounts_AvatarStorePath=/app/uploads


EXPOSE 3000

CMD ["node", "main.js"]
```

Подправить нужно переменную окружения `ENV RC_VERSION 4.1.2` изменив версию на `4.3.3`.
Сохраняем файл и собираем образ. Собирать нужно там же, на процессоре `arm64` или можно нагуглить варианты, как можно его эмулировать.
Идём туда, где есть нужный процессор, ставим `docker`, собираем образ и пушим его к себе в `hub.docker.com` предварительно залогинившись туда, ну, или в свой собственный реестр образов, откуда потом его будет стягивать `Kubernetes`:
```
$ docker build -t rs1977rs/rocketchat:arm64-4.3.3 .
$ docker push rs1977rs/rocketchat:arm64-4.3.3
```

После этого осталось поправить файл 'rocket-deploy.yaml' и еще раз применить его. Меняем строку, указывающую на расположение образа:
```
    spec:
      containers:
        - name: rocketchat-server
          image: rs1977rs/rocketchat:arm64-4.3.3
```
сохраняем и применяем.

Через какое-то время можно зайти в логи приложения и убедиться, что сервер запущен. Вполне возможно там увидеть ошибки вида:
```
{"level":40,"time":"2022-02-17T17:10:55.529Z","pid":1,"hostname":"rocketchat-server-5cbbd94884-pbqw2","name":"Migrations","msg":"Not migrating, control is locked. Attempt 0/30. Trying again in 10 seconds.
```
Проблемы с миграцией, возможно из-за того, что мы перескочили сразу через много версий. Надо дождаться пока пройдут все миграции, сервер стартует и только после чего можно идти в браузер и смотреть, какие появились новые фичи. Это может занять довольно много времени, паниковать раньше времени не стоит. Если процесс не движется - можно попробовать перезапустить под с приложением, мне помогло.
Финальный лог выглядит примерно так:
```
{"level":51,"time":"2022-02-17T17:15:40.518Z","pid":1,"hostname":"rocketchat-server-5cbbd94884-krzqc","name":"Migrations","msg":"Running up() on version 249"}
{"level":51,"time":"2022-02-17T17:15:40.524Z","pid":1,"hostname":"rocketchat-server-5cbbd94884-krzqc","name":"Migrations","msg":"Running up() on version 250"}
{"level":51,"time":"2022-02-17T17:15:40.531Z","pid":1,"hostname":"rocketchat-server-5cbbd94884-krzqc","name":"Migrations","msg":"Running up() on version 251"}
{"level":51,"time":"2022-02-17T17:15:40.546Z","pid":1,"hostname":"rocketchat-server-5cbbd94884-krzqc","name":"Migrations","msg":"Running up() on version 252"}
{"level":51,"time":"2022-02-17T17:15:40.554Z","pid":1,"hostname":"rocketchat-server-5cbbd94884-krzqc","name":"Migrations","msg":"Finished migrating."}
ufs: temp directory created at "/tmp/ufs"
+----------------------------------------------+
| SERVER RUNNING |
+----------------------------------------------+
| |
| Rocket.Chat Version: 4.3.3 |
| NodeJS Version: 12.22.1 - arm64 |
| MongoDB Version: 4.2.18 |
| MongoDB Engine: wiredTiger |
| Platform: linux |
| Process Port: 3000 |
| Site URL: http://localhost:3000 |
| ReplicaSet OpLog: Enabled |
| Commit Hash: 9b685693fb |
| Commit Branch: HEAD |
| |
+----------------------------------------------+
```
