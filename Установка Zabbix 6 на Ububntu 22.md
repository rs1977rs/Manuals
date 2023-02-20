## Установка Zabbix 6 на Ubuntu 22 (Apache, MySQL)

За основу взята [статья](https://www.server-world.info/en/note?os=Ubuntu_22.04&p=zabbix60&f=1). Она помогает установить необходимое ПО (Apache, PHP и MySQL), но в ней есть неточность на этапе раскатки скрипта - ошибка в пути.
Поэтому пользуемся [официальным мануалом](https://www.zabbix.com/ru/download?zabbix=6.0&os_distribution=ubuntu&os_version=22.04&components=server_frontend_agent&db=mysql&ws=apache). Кроме правильного пути до скрипта там есть еще нюансы, например отдельное создание пользователя перед выдачей ему привелегий и установки переменной до раскатки скрипта.

### Устанавливаем Apache
```
# apt -y install apache2
```

### Устанавливаем PHP и PHP-FPM
```
# apt -y install php8.1 php8.1-mbstring php-pear

# php -v
PHP 8.1.2 (cli) (built: Apr  7 2022 17:46:26) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.1.2, Copyright (c) Zend Technologies
    with Zend OPcache v8.1.2, Copyright (c), by Zend Technologies
```
```
# apt -y install php-fpm
```

### Устанавливаем MariaDB
```
# apt -y install mariadb-server

# mysql_secure_installation

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Switch to unix_socket authentication [Y/n] n
 ... skipping.

Change the root password? [Y/n] n
 ... skipping.

Remove anonymous users? [Y/n] y
 ... Success!

Disallow root login remotely? [Y/n] y
 ... Success!

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reload privilege tables now? [Y/n] y
 ... Success!

```

### Устанавливаем Zabbix и Zabbix-agent
```
# wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu22.04_all.deb

# dpkg -i zabbix-release_6.0-4+ubuntu22.04_all.deb

# apt update

# apt -y install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent2 php-mysql php-gd php-bcmath php-net-socket
```

### Создаем и настраиваем БД
```
# mysql

MariaDB [(none)]> create database zabbix character set utf8mb4 collate utf8mb4_bin;
Query OK, 1 row affected (0.00 sec)

# замените [password] на свой пароль
MariaDB [(none)]> create user zabbix@localhost identified by 'password';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> grant all privileges on zabbix.* to zabbix@localhost;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> set global log_bin_trust_function_creators = 1;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> exit
Bye

# zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -p zabbix
Enter password:   # пароль, который вы установили ранее для пользователя [zabbix]
                  # процесс установки не быстрый, у меня заняло около 10 минут

# mysql
MariaDB [(none)]> set global log_bin_trust_function_creators = 0;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> exit
Bye                 
```

### Настраиваем сервер Zabbix
```
# vim /etc/zabbix/zabbix_server.conf

    # line 105 : проверьте имя БД
DBName=zabbix
    # line 121 : проверьте имя пользователя
DBUser=zabbix
    # line 130 : добавьте пароль пользователя
DBPassword=zabbix

# systemctl restart zabbix-server
# systemctl enable zabbix-server
```

### Настраиваем PHP
```
# vim /etc/php/8.1/fpm/pool.d/www.conf

    # добавьте в конце файла
php_value[max_execution_time] = 300
php_value[memory_limit] = 128M
php_value[post_max_size] = 16M
php_value[upload_max_filesize] = 2M
php_value[max_input_time] = 300
php_value[max_input_vars] = 10000
php_value[always_populate_raw_post_data] = -1
php_value[date.timezone] = Europe/Moscow

# systemctl restart apache2 php8.1-fpm
```

## Установка закончена!
Заходим по адресу [http://host.ip/zabbix]()

Задаём пароль админа и после уведомления об успешной установки логинимся как `Admin` с паролем, который задали на шаге установки админского пароля.
