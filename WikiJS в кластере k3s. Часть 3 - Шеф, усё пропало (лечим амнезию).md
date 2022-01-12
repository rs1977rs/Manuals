## WikiJS в кластере k3s. Часть 3 - Шеф, усё пропало (лечим амнезию)

Wiki.js - это wiki-движок, работающий на Node.js и написанный на JavaScript.
В стеке технологий он относится к инструментам категории "Документация и обслуживание".

---
### Prerequisites

1. Wiki.js, установленная в кластер k3s. В качестве хранилища - PostgreSQL.

2. Возможные проблемы и способы восстановления работоспособности:
  * Восстановление работоспособности PostgreSQL.
  * Восстановление работоспособности Wiki.js.
  * Получение доступа к базе данных внутри Wiki.js.

3. Полезные ссылки
  * [Официальный сайт Wiki.js](https://github.com/Requarks/wiki)

---

### Запускаем PostgreSQL

Скейлим деплоймент в 1 и смотрим на поды в неймспейсе:
```
$ kubectl -n wiki get pod
NAME                        READY   STATUS             RESTARTS   AGE
postgres-5c9fd596b6-jk2xj   0/1     CrashLoopBackOff   1          20s
```
Надо бы узнать причину. Смотрим логи пода PostgreSQL:
```
$ kubectl -n wiki logs postgres-5c9fd596b6-jk2xj

PostgreSQL Database directory appears to contain a database; Skipping initialization

2022-01-12 10:01:50.505 GMT [1] LOG:  could not open configuration file "/var/lib/pgsql/openshift-custom-postgresql.conf": No such file or directory
2022-01-12 10:01:50.505 GMT [1] FATAL:  configuration file "/var/lib/postgresql/data/pgdata/postgresql.conf" contains errors
```
Как видим, ругается на то, что не может открыть файл `openshift-custom-postgresql.conf`, которого скорее всего нет, мы же в кубере, а не в шифте, а в файле конфига `postgresql.conf` он есть. Будем править наживую, через файловую систему хоста, не заходя в под. В OpenShift есть возможность запустить нерабочий под в режиме отладки, в кубере я такую возможность пока не искал.
Итак, идём в папку, куда мы только что скопировали файлы PostgreSQL на сервере k3s, находим там файл конфигурации и удаляем (комментируем) строку, в которой присутствует `openshift-custom-postgresql.conf`
```
# vim postgresql.conf
```
и в самом конце файла находим:
```
# Custom OpenShift configuration:
include '/var/lib/pgsql/openshift-custom-postgresql.conf'
```
Удаляем зависший под PostgreSQL:
```
$ kubectl -n wiki delete pod postgres-5c9fd596b6-jk2xj
pod "postgres-5c9fd596b6-jk2xj" deleted
```
И через какое-то время видим, что всё ОК, PostgreSQL запустился:
```
$ kubectl -n wiki get pod
NAME                        READY   STATUS    RESTARTS   AGE
postgres-5c9fd596b6-7hdhd   1/1     Running   0          70s
```
и в логах у него всё хорошо:
```
$ kubectl -n wiki logs postgres-5c9fd596b6-7hdhd

PostgreSQL Database directory appears to contain a database; Skipping initialization

2022-01-12 10:09:23.116 UTC [1] LOG:  starting PostgreSQL 12.9 (Debian 12.9-1.pgdg110+1) on aarch64-unknown-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
2022-01-12 10:09:23.117 UTC [1] LOG:  listening on IPv6 address "::1", port 5432
2022-01-12 10:09:23.117 UTC [1] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2022-01-12 10:09:23.122 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2022-01-12 10:09:23.132 UTC [1] LOG:  redirecting log output to logging collector process
2022-01-12 10:09:23.132 UTC [1] HINT:  Future log output will appear in directory "log".
```

### Запускаем приложение Wiki.js

Точно также редактируем деплоймент Wiki.js, скейлим в 1 и смотрим поды:
```
$ kubectl -n wiki get pod
NAME                        READY   STATUS    RESTARTS   AGE
postgres-5c9fd596b6-7hdhd   1/1     Running   0          3m33s
wiki-7d648bbd79-fq8hm       1/1     Running   0          3s
```
Смотрим логи Wiki:
```
$ kubectl -n wiki logs wiki-7d648bbd79-fq8hm
Loading configuration from /wiki/config.yml... OK
2022-01-12T10:12:52.787Z [MASTER] info: =======================================
2022-01-12T10:12:52.789Z [MASTER] info: = Wiki.js 2.5.260 =====================
2022-01-12T10:12:52.789Z [MASTER] info: =======================================
2022-01-12T10:12:52.789Z [MASTER] info: Initializing...
2022-01-12T10:12:53.482Z [MASTER] info: Using database driver pg for postgres [ OK ]
2022-01-12T10:12:53.486Z [MASTER] info: Connecting to database...
2022-01-12T10:12:53.498Z [MASTER] error: Database Connection Error: ECONNREFUSED 10.43.207.163:5432
2022-01-12T10:12:53.498Z [MASTER] warn: Will retry in 3 seconds... [Attempt 1 of 10]
2022-01-12T10:12:56.500Z [MASTER] info: Connecting to database...
2022-01-12T10:12:56.501Z [MASTER] error: Database Connection Error: ECONNREFUSED 10.43.207.163:5432
2022-01-12T10:12:56.502Z [MASTER] warn: Will retry in 3 seconds... [Attempt 2 of 10]
2022-01-12T10:12:59.504Z [MASTER] info: Connecting to database...
```
Приложение не может подцепиться к поду с базой, после чего падает окончательно.
```
$ kubectl -n wiki get pod
NAME                        READY   STATUS    RESTARTS   AGE
postgres-5c9fd596b6-7hdhd   1/1     Running   0          5m30s
wiki-7d648bbd79-fq8hm       0/1     Error     2          2m
```
Почему?

* Начинаем тюнить PostgreSQL

Мне пришлось сделать ещё одну установку PostgreSQL, чтобы найти разницу. Вот так выглядит лог PostgreSQL, когда всё работает:
```
2021-12-24 16:12:44.287 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2021-12-24 16:12:44.287 UTC [1] LOG:  listening on IPv6 address "::", port 5432
```
Разница очевидна. Нужно в настройках PostgreSQL разрешить доступ с любого адреса. Возвращаемся в файловую систему хоста k3s и редактируем всё тот же файл 'postgresql.conf'. Нужно раскомментировать строку `#listen_addresses = 'localhost'` и заменить значение на `listen_addresses = '*'`. После чего опять удалить работающий под PostgreSQL, чтобы применились новые настройки.
Можно увидеть, что теперь под с PostgreSQL готов принимать подключения с любого адреса:
```
2022-01-12 10:20:43.805 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2022-01-12 10:20:43.805 UTC [1] LOG:  listening on IPv6 address "::", port 5432
```
Таким же образом рестартуем под с приложением и смотрим логи Wiki:
```
$ kubectl -n wiki logs wiki-7d648bbd79-p6rhz
Loading configuration from /wiki/config.yml... OK
2022-01-12T10:21:33.323Z [MASTER] info: =======================================
2022-01-12T10:21:33.331Z [MASTER] info: = Wiki.js 2.5.260 =====================
2022-01-12T10:21:33.331Z [MASTER] info: =======================================
2022-01-12T10:21:33.331Z [MASTER] info: Initializing...
2022-01-12T10:21:34.097Z [MASTER] info: Using database driver pg for postgres [ OK ]
2022-01-12T10:21:34.100Z [MASTER] info: Connecting to database...
2022-01-12T10:21:34.116Z [MASTER] error: Database Connection Error: 28P01 undefined:undefined
2022-01-12T10:21:34.116Z [MASTER] warn: Will retry in 3 seconds... [Attempt 1 of 10]
2022-01-12T10:21:37.120Z [MASTER] info: Connecting to database...
2022-01-12T10:21:37.124Z [MASTER] error: Database Connection Error: 28P01 undefined:undefined
2022-01-12T10:21:37.124Z [MASTER] warn: Will retry in 3 seconds... [Attempt 2 of 10]
```
Видим, что ошибка поменялась, к базе так и не удается подключиться. Причина в том, что скорее всего владельцем БД в PostgreSQL является другой пользователь и с неизвестно каким паролем. Что ж, это можно поправить. Идём в консоль пода PostgreSQL:
```
$ kubectl -n wiki exec -ti postgres-5c9fd596b6-bp65s /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@postgres-5c9fd596b6-bp65s:/#
```
Второй строчкой нам сообщили, что такой формат команды уже не используется.
ОК, подключимся к базе и посмотрим, что там есть:
```
# psql -Upostgres
psql (12.9 (Debian 12.9-1.pgdg110+1))
Type "help" for help.

postgres=# \l
                                   List of databases
   Name      |     Owner     | Encoding |  Collate   |   Ctype    |   Access privileges   
---------------+---------------+----------+------------+------------+-----------------------
devops-wikijs | devops-wikijs | UTF8     | en_US.utf8 | en_US.utf8 |
postgres      | postgres      | UTF8     | en_US.utf8 | en_US.utf8 |
template0     | postgres      | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |               |          |            |            | postgres=CTc/postgres
template1     | postgres      | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
              |               |          |            |            | postgres=CTc/postgres
(4 rows)
```
Ну да, база называется `devops-wikijs` и владеет ей пользователь `devops-wikijs`, чьего пароля мы, возможно, не знаем.
Сбросим его, чтобы потом поменять в конфиге Wiki.js:
```
postgres=# ALTER USER devops-wikijs PASSWORD 'newpassword';
ERROR:  syntax error at or near "-"
LINE 1: ALTER USER devops-wikijs PASSWORD 'newpassword';
                       ^
postgres=# ALTER USER "devops-wikijs" PASSWORD 'newpassword';
ALTER ROLE
```
В первый раз пароль поменять не удалось, потому что в имени пользователя был дефис. Заключив имя в кавычки, мы решили проблему.

* Тюним конфиг Wiki.js

Правим файл `wiki-config.yaml`:
```
apiVersion: v1
kind: ConfigMap
metadata:
name: wiki-config
namespace: wiki
data:
DB_HOST: "postgres.wiki.svc"
DB_PORT: "5432"
DB_USER: "devops-wikijs"
DB_TYPE: "postgres"
DB_NAME: "devops-wikijs"
```
Правим файл `postgres.yaml` (ту его часть, где описаны переменные окружения):
```
      env:
      - name: POSTGRES_DB
        value: wiki
      - name: POSTGRES_USER
        value: wiki
      - name: POSTGRES_PASSWORD
```
Правим секрет с паролем `postgres-pass.yaml` (предварительно закодировав его в base64):
```
kind: Secret
apiVersion: v1
metadata:
name: postgres-pass
namespace: wiki
data:
postgres-pass: bmV3cGFzc3dvcmQ=
type: Opaque
```  
И применяем по очереди эти три ямлика, начиная с секрета, затем нужно передеплоить PostgreSQL, затем конфигмап Wiki.js и затем передеплоить приложение Wiki, чтобы применился новый конфиг. Простой приложения нам не критичен, всё равно ничего не работает, поэтому просто удалим деплойменты и создадим их заново.
```
$ kubectl apply -f postgres-pass.yaml
secret/postgres-pass configured

$ kubectl -n wiki get deployment
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
postgres   1/1     1            1           22h
wiki       0/1     1            0           22h

$ kubectl -n wiki delete deployment postgres
deployment.apps "postgres" deleted

$ kubectl -n wiki delete deployment wiki
deployment.apps "wiki" deleted

$ kubectl apply -f postgres.yaml
deployment.apps/postgres created

$ kubectl apply -f wiki-config.yaml
configmap/wiki-config configured

$ kubectl apply -f wiki-deployment.yaml
deployment.apps/wiki created

$ kubectl -n wiki get pod
NAME                        READY   STATUS    RESTARTS   AGE
postgres-5955bd58b8-dn46l   1/1     Running   0          3m12s
wiki-7d648bbd79-n2smc       1/1     Running   0          14s
```
Смотрим логи PostgreSQL:
```
$ kubectl -n wiki logs postgres-5955bd58b8-dn46l

PostgreSQL Database directory appears to contain a database; Skipping initialization

2022-01-12 11:05:37.996 UTC [1] LOG:  starting PostgreSQL 12.9 (Debian 12.9-1.pgdg110+1) on aarch64-unknown-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
2022-01-12 11:05:37.997 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2022-01-12 11:05:37.997 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2022-01-12 11:05:38.001 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2022-01-12 11:05:38.014 UTC [1] LOG:  redirecting log output to logging collector process
2022-01-12 11:05:38.014 UTC [1] HINT:  Future log output will appear in directory "log".
```
и Wiki:
```
$ kubectl -n wiki logs wiki-7d648bbd79-n2smc
Loading configuration from /wiki/config.yml... OK
2022-01-12T11:08:35.824Z [MASTER] info: =======================================
2022-01-12T11:08:35.825Z [MASTER] info: = Wiki.js 2.5.260 =====================
2022-01-12T11:08:35.825Z [MASTER] info: =======================================
2022-01-12T11:08:35.825Z [MASTER] info: Initializing...
2022-01-12T11:08:36.434Z [MASTER] info: Using database driver pg for postgres [ OK ]
2022-01-12T11:08:36.438Z [MASTER] info: Connecting to database...
2022-01-12T11:08:36.471Z [MASTER] info: Database Connection Successful [ OK ]
2022-01-12T11:08:37.230Z [MASTER] warn: Mail is not setup! Please set the configuration in the administration area!
2022-01-12T11:08:37.330Z [MASTER] info: Loading GraphQL Schema...
2022-01-12T11:08:38.081Z [MASTER] info: GraphQL Schema: [ OK ]
2022-01-12T11:08:38.387Z [MASTER] info: HTTP Server on port: [ 3000 ]
2022-01-12T11:08:38.462Z [MASTER] info: HTTP Server: [ RUNNING ]
...........
```
Всё запустилось, но радоваться рано ;).
Если мы зайдём в веб-морду, то увидим только стартовую страницу, а при попытке залогиниться, нас отредиректит на сервис `keycloack`, который в этой конкретной установке был сделан дефолтным. Плюс ко всему, пароля админа мы не знаем. Т.е. база для нас закрыта. Но, это тоже не проблема. Нужно всего лишь поменять способ авторизации и сбросить пароль для пользователя с админскими правами.

### Получаем доступ к базе данных

Для этого нам опять будет нужен подопытный кролик в виде чистой базы.
Но сначала подключимся к нашей базе и посмотрим структуру базы данных:
```
postgres=# \c devops-wikijs
You are now connected to database "devops-wikijs" as user "postgres".
devops-wikijs=# \dt
              List of relations
 Schema |       Name       | Type  |  Owner  
--------+------------------+-------+---------
 public | analytics        | table | pgbench
 public | apiKeys          | table | pgbench
 public | assetData        | table | pgbench
 public | assetFolders     | table | pgbench
 public | assets           | table | pgbench
 public | authentication   | table | pgbench
 public | brute            | table | pgbench
 public | commentProviders | table | pgbench
 public | comments         | table | pgbench
 public | editors          | table | pgbench
 public | groups           | table | pgbench
 public | locales          | table | pgbench
 public | loggers          | table | pgbench
 public | migrations       | table | pgbench
 public | migrations_lock  | table | pgbench
 public | navigation       | table | pgbench
 public | pageHistory      | table | pgbench
 public | pageHistoryTags  | table | pgbench
 public | pageLinks        | table | pgbench
 public | pageTags         | table | pgbench
 public | pageTree         | table | pgbench
 public | pages            | table | pgbench
 public | renderers        | table | pgbench
 public | searchEngines    | table | pgbench
 public | sessions         | table | pgbench
 public | settings         | table | pgbench
 public | storage          | table | pgbench
 public | tags             | table | pgbench
 public | userAvatars      | table | pgbench
 public | userGroups       | table | pgbench
 public | userKeys         | table | pgbench
 public | users            | table | pgbench
(32 rows)
```
Нас интересует таблица `users`, посмотрим что в ней:
```
select * from users;
```
Такой запрос выведет кучу плохочитаемого мусора, ограничим только нужные поля:
```
# select id, email, name, password, "providerKey" from users;
 id  |              email              |                       name                       |                           password                           |             providerKey              
-----+---------------------------------+--------------------------------------------------+--------------------------------------------------------------+--------------------------------------
..............
  44 | testuser@yandex.ru              | Test User                                        |                                                              | c2af9e0a-ff64-4eae-8279-c8c1b6def5d0
   1 | admin@example.com               | Administrator                                    | $2a$12$zHK5Bgp85VWIOMILv.pTgec/LzlJmEOTbJtNtdDquRRjfrKyW9dkC | local
..............
```
Если поле `providerKey` не взять в кавычки, вываливается ошибка. В реальном выводе будет гораздо больше строк, с целью недопущения утечки приватных данных я вывод малость порезал. Самое важное тут, что некоторые пользователи (все) имеют некий идентификатор провайдера авторизации, а админ может авторизоваться локально. Всё, что нам нужно, узнать пароль админа, который скрывается за хэшем. Вот тут то нам и пригодится "кролик".
Идём в веб-интерфейс нашей подопытной базы и создаем там пользователя. Не буду описывать процедуру - там интуитивно всё понятно.
После чего подключаемся в консоль пода PostgreSQL подопытной базы и проделываем там те же самые действия - нам нужно получить список пользователей и их хэши паролей.
```
 id |       email        |     name      |                           password                           | providerKey
----+--------------------+---------------+--------------------------------------------------------------+-------------
  4 | newuser@new.user   | New User      | $2a$12$fwb1iX48Bu.MaqnzEeJgzO2gLzSdIM.mwWndwsBpZaJ78rLT4Yrbm | local
```
Последний созданный мной пользователь с почтой `newuser@new.user` и паролем `newpassword` имеет такой вот хэш `$2a$12$fwb1iX48Bu.MaqnzEeJgzO2gLzSdIM.mwWndwsBpZaJ78rLT4Yrbm`, который нам нужно подставить админу нашей базы. Как видим, несмотря на то, что пароль я взял такой же, как для PostgreSQL, но здесь явно не `base64` авторизация и хэш заметно длиннее.
Идём в консоль пода рабочей базы и выполняем там запрос `UPDATE`:
```
UPDATE users SET password = '$2a$12$fwb1iX48Bu.MaqnzEeJgzO2gLzSdIM.mwWndwsBpZaJ78rLT4Yrbm' where id = '1';
```
Я бы еще обновил поле `email`, чтобы чисто теоретически, не улетело письмо реальному админу той базы, если вдруг он и вы - не один и тот же человек, но это больше паранойя.
```
UPDATE users SET email = 'newadmin@example.com' where id = '1';
```
```
devops-wikijs=# select id, email, name, password, "providerKey" from users where "providerKey"='local';
 id |        email         |     name      |                           password                           | providerKey
----+----------------------+---------------+--------------------------------------------------------------+-------------
  2 | guest@example.com    | Guest         |                                                              | local
  1 | newadmin@example.com | Administrator | $2a$12$fwb1iX48Bu.MaqnzEeJgzO2gLzSdIM.mwWndwsBpZaJ78rLT4Yrbm | local
(2 rows)
```

После чего можно вернуться в браузер и попробовать залогиниться с правами локального администратора. Для этого нужно в адресной строке в конце добавить `/login?all=1`, т.е. полный адрес будет `wiki.example.com/login?all=1`.
Выбираем локальную авторизацию, вписываем пользователя `newadmin@example.com`, его пароль `newpassword`.
Профит.
