## Wiki.js в кластере k3s. Бэкап и восстановление из него.

Wiki.js - это wiki-движок, работающий на Node.js и написанный на JavaScript.
В стеке технологий он относится к инструментам категории "Документация и обслуживание".

---
### Prerequisites

1. Wiki.js, установленная в кластер k3s. В качестве хранилища - PostgreSQL.

2. План восстановления:
  * Восстановление из бэкапа, заменой содержимого папки pgdata.
  * Постоянная синхронизация в стороннее хранилище с помощью git.
  * Восстановление из бэкапа, созданного синхронизацией в git.

3. Полезные ссылки
  * [Официальный сайт Wiki.js](https://github.com/Requarks/wiki)

---
### Варианты бэкапа

Wiki.js поддерживает самые разные формы бэкапа. Рассмотрим два варианта:
  * Постоянная синхронизация с внешним хранилищем с помощью git.
  * Бэкап самой базы данных PostgreSQL.

#### Бэкап средствами PostgreSQL

Начнём со второго варианта.

Предположим, что у нас есть работающий экземпляр Wiki.js, в котором при установке в качестве хранилища была выбрана БД PostgreSQL. Также предположим, что БД у нас бэкапится средствами самой PostgreSQL, например Barman, который по расписанию просто копирует каталог pgdata к себе, откуда его можно при необходимости достать. Второй вариант - нам нужно просто перенести Wiki.js в другое место. Это можно сделать просто подсунув существущую базу в новую инсталляцию.
В моём случае, каталог pgdata находился в постоянном томе OpenShift, откуда я его просто скопировал.

Итак, у нас есть чистая установка Wiki.js в кластере k3s. Перед установкой было бы неплохо узнать версию PostgreSQL той базы, которую будем прицеплять к нашей Wiki. Посмотреть можно в файле `PG_VERSION`:
```
$ cat PG_VERSION
12
```
Мне повезло, я угадал - сразу поставил 12-ю версию PostgreSQL.

  * Узнаём, где лежит БД.

Так как к кластеру не подключено никаких внешних хранилищ, кластер однонодовый, для тестовых целей, в качестве хранилища используется дисковой пространство хоста. Т.е. все тома данных, присоединенные к подам лежат где-то в файловой системе сервера. Для того, чтобы найти, где именно, нужно посмотреть описание тома. Получаем список 'pvc':
```
$ kubectl -n wiki get pvc
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgres-pvc    Bound    pvc-89f82eb9-7845-41cd-9b24-07b2490ec362   5Gi        RWO            local-path     20h
wiki-data-pvc   Bound    pvc-c9f6cab3-a160-416f-9ca3-20f93feee2cc   7Gi        RWO            local-path     20h
```
Можно посмотреть описание нужного нам `postgres-pvc`:
```
$ kubectl -n wiki get pvc postgres-pvc -o yaml
```
но ничего нового в выводе мы не увидим, так как нужная нам информация уже есть - нам нужно имя волума, его описание и будем смотреть (вывод сокращён):
```
$ kubectl -n wiki get pv pvc-89f82eb9-7845-41cd-9b24-07b2490ec362 -o yaml
apiVersion: v1
kind: PersistentVolume
metadata:
...........
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 5Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: postgres-pvc
    namespace: wiki
    resourceVersion: "1679358"
    uid: 89f82eb9-7845-41cd-9b24-07b2490ec362
  hostPath:
    path: /var/lib/rancher/k3s/storage/pvc-89f82eb9-7845-41cd-9b24-07b2490ec362_wiki_postgres-pvc
    type: DirectoryOrCreate
```
Нас интересует поле `hostPath: path:`

Заходим на сервер по ssh, получаем рута и идём в папку `/var/lib/rancher/k3s/storage/`:
```
$ sudo su
# cd /var/lib/rancher/k3s/storage/
# ll
total 36
drwx-----x  9 root   root    4096 Jan 11 12:14 ./
drwxr-xr-x  6 root   root    4096 Dec 20 10:40 ../
drwxrwxrwx  3 root   root    4096 Jan 11 12:12 pvc-89f82eb9-7845-41cd-9b24-07b2490ec362_wiki_postgres-pvc/
drwxrwxrwx 29 nobody nogroup 4096 Jan 12 07:00 pvc-92417c96-e7a0-4327-be00-470dd19eaed7_lens-metrics_data-prometheus-0/
drwxrwxrwx  5 root   root    4096 Jan 11 12:19 pvc-c9f6cab3-a160-416f-9ca3-20f93feee2cc_wiki_wiki-data-pvc/
```
Находим каталог с нашим волумом и проваливаемся в него:
```
# cd pvc-89f82eb9-7845-41cd-9b24-07b2490ec362_wiki_postgres-pvc/pgdata/
# ll
total 132
drwx------ 19 systemd-coredump root              4096 Jan 11 12:12 ./
drwxrwxrwx  3 root             root              4096 Jan 11 12:12 ../
-rw-------  1 systemd-coredump systemd-coredump     3 Jan 11 12:12 PG_VERSION
drwx------  6 systemd-coredump systemd-coredump  4096 Jan 11 12:12 base/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:21 global/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_commit_ts/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_dynshmem/
-rw-------  1 systemd-coredump systemd-coredump  4782 Jan 11 12:12 pg_hba.conf
-rw-------  1 systemd-coredump systemd-coredump  1636 Jan 11 12:12 pg_ident.conf
drwx------  4 systemd-coredump systemd-coredump  4096 Jan 12 08:13 pg_logical/
drwx------  4 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_multixact/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_notify/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_replslot/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_serial/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_snapshots/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_stat/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 12 08:41 pg_stat_tmp/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_subtrans/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_tblspc/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_twophase/
drwx------  3 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_wal/
drwx------  2 systemd-coredump systemd-coredump  4096 Jan 11 12:12 pg_xact/
-rw-------  1 systemd-coredump systemd-coredump    88 Jan 11 12:12 postgresql.auto.conf
-rw-------  1 systemd-coredump systemd-coredump 26810 Jan 11 12:12 postgresql.conf
-rw-------  1 systemd-coredump systemd-coredump    36 Jan 11 12:12 postmaster.opts
-rw-------  1 systemd-coredump systemd-coredump   101 Jan 11 12:12 postmaster.pid
```
Видим стандартную структуру файлов PostgreSQL. Нужно потушить под, к которому прицеплен этот волум, удалить тут всё, положить данные бэкапа и запустить всё назад. Небольшой спойлер - можно не удалять файлы `postgres.conf` и `pg_hba.conf` (и, возможно, некоторые другие), но мы наступим на все грабли, что поможет траблшутингу в будущем.
  **Здесь неплохо запомнить, кто был владельцем файлов!**

  * Выключаем нагрузку

Получаем список деплойментов в нашем неймспейсе:
```
$ kubectl -n wiki get deployments

NAME       READY   UP-TO-DATE   AVAILABLE   AGE
postgres   1/1     1            1           20h
wiki       1/1     1            1           20h
```
Потушим сначала приложение, потом базу, путём простого редактирования деплоймента. Скейлим в ноль, т.е. устанавливаем значение поле `replicas: 1` равным нулю:
```
$ kubectl -n wiki edit deployment wiki

# редактируем в любом текстовом редакторе

apiVersion: apps/v1
kind: Deployment
metadata:
................
  labels:
    app: wiki
  name: wiki
  namespace: wiki
spec:
  progressDeadlineSeconds: 600
  replicas: 0

# и после редактирования видим что всё ОК:

deployment.apps/wiki edited
```
Видим, что под с приложением начал удаляться и через какое-то время останется только один (ещё спойлер - это не Маклауд ;)) ):
```
$ kubectl -n wiki get pod
NAME                        READY   STATUS        RESTARTS   AGE
postgres-5c9fd596b6-lhtqv   1/1     Running       0          20h
wiki-7d648bbd79-v7g9h       1/1     Terminating   0          20h
```
Точно также тушим под с PostgreSQL.
```
$ kubectl -n wiki get pod
No resources found in wiki namespace.
```
Всё потушено, можно подменять базу.
  * Удаляем пустую базу

Возвращаемся в консоль сервера, поднимаемся уровнем выше и грохаем папку с базой:
```
# cd ..
# pwd
/var/lib/rancher/k3s/storage/pvc-89f82eb9-7845-41cd-9b24-07b2490ec362_wiki_postgres-pvc
# ll
total 12
drwxrwxrwx  3 root             root 4096 Jan 11 12:12 ./
drwx-----x  9 root             root 4096 Jan 11 12:14 ../
drwx------ 19 systemd-coredump root 4096 Jan 12 08:56 pgdata/
# rm -rf pgdata/
# ll
total 8
drwxrwxrwx 2 root root 4096 Jan 12 08:57 ./
drwx-----x 9 root root 4096 Jan 11 12:14 ../
#
```
  * Копируем бэкап на место убиенной базы

Я захожу на сервер k3s по ssh как юзер, потому в папку `/var/lib/rancher/k3s/storage/...` у меня записать не получится, скину в `/tmp`, потом перенесу.
Идём туда, где лежит наш бэкап и rsync'ом заливаем его на сервер.
В моём случае данные лежат в папке `/data/userdata/`, значит переходим в неё и копируем:
```
$ rsync -av . <k3s_host>:/tmp/1
```
Теперь идём назад на сервер k3s и копируем данные из `/tmp` в `/var/lib/rancher/...`
```
# pwd
/var/lib/rancher/k3s/storage/pvc-89f82eb9-7845-41cd-9b24-07b2490ec362_wiki_postgres-pvc
# cp -r /tmp/1 ./pgdata
# ll
total 12
drwxrwxrwx  3 root root 4096 Jan 12 09:50 ./
drwx-----x  9 root root 4096 Jan 11 12:14 ../
drwxr-xr-x 20 root root 4096 Jan 12 09:50 pgdata/
```
и возвращаем прежние права на папку:
```
# cd ..
# chown -R systemd-coredump:systemd-coredump pgdata/
```
После чего нужно запустить PostgreSQL, отскейлив его деплоймент в единицу, и если приложение просто переносится на другое место с сохранением всех настроек или разворачивается бэкап на то же самое место, то всё должно запуститься.
Если что-то меняется, например, другой кластер или вообще другая база, то дальше начинается самое интересное.
[Как лечить амнезию, читаем здесь](https://github.com/rs1977rs/Manuals/blob/master/WikiJS%20%D0%B2%20%D0%BA%D0%BB%D0%B0%D1%81%D1%82%D0%B5%D1%80%D0%B5%20k3s.%20%D0%A7%D0%B0%D1%81%D1%82%D1%8C%203%20-%20%D0%A8%D0%B5%D1%84%2C%20%D1%83%D1%81%D1%91%20%D0%BF%D1%80%D0%BE%D0%BF%D0%B0%D0%BB%D0%BE%20(%D0%BB%D0%B5%D1%87%D0%B8%D0%BC%20%D0%B0%D0%BC%D0%BD%D0%B5%D0%B7%D0%B8%D1%8E).md)
