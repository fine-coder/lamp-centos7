# Установка LAMP на CentOS 7


### 1. Установим репозиторий EPEL

```
sudo yum install epel-release
sudo yum update
```


### 2. Установим Apache

```
sudo yum update httpd
```
пишет
```
Package(s) httpd available, but not installed.
No packages marked for update
```

значит пакет Apache отсутствует  
установим
```
sudo yum install httpd
```


### 3. Запустим firewall

проверим состояние брандмауэра
```
sudo systemctl status firewalld
```

если служба отключена, то необходимо включить
```
sudo systemctl start firewalld
```
затем
```
sudo systemctl enable firewalld
```

теперь нужно посмотреть, запущен ли Firewalld
```
sudo firewall-cmd --state
```


### 4. Откроем порт 80 и 443 в брандмауэре

```
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```


### 5. Запускаем веб-сервер

```
sudo systemctl start httpd
```

проверим текущее состояние
```
sudo systemctl status httpd
```

получаем
```
Active: active (running)
```


### 6. Папка проекта

создаем папку проекта
```
sudo mkdir -p /var/www/domain.org/html
```

дополнительная директория для хранения файлов журнала для сайта
```
sudo mkdir -p /var/www/domain.org/log
```

затем назначим права владения для директории html с помощью переменной среды $USER
```
sudo chown -R $USER:$USER /var/www/domain.org/html
```

убедимся, что наша корневая директория имеет набор разрешений по умолчанию
```
sudo chmod -R 755 /var/www
```

затем создадим в качестве примера страницу index.html
```
sudo nano /var/www/domain.org/html/index.html
```
что-нибудь напишем  
сохраним `ctrl + o`  
выйдем `ctrl + x`


### 7. Настройка виртуальных хостов

создаем директории
```
sudo mkdir /etc/httpd/sites-available /etc/httpd/sites-enabled
```

затем мы должны попросить Apache выполнить поиск виртуальных хостов в директории sites-enabled, для этого необходимо изменить главный файл конфигурации Apache и добавить строку, объявляющую опциональную директорию для дополнительных файлов конфигурации:
```
sudo nano /etc/httpd/conf/httpd.conf
```

добавим в конец файла строку
```
IncludeOptional sites-enabled/*.conf
```

теперь, когда у нас есть директории виртуального хоста, мы можем создать наш файл виртуального хоста  
начнем с создания нового файла в директории sites-available:
```
sudo nano /etc/httpd/sites-available/domain.org.conf
```

добавим следующий блок:
```
<VirtualHost *:80>
    ServerName domain.org
    ServerAlias www.domain.org
    DocumentRoot /var/www/domain.org/html
    ErrorLog /var/www/domain.org/log/error.log
    CustomLog /var/www/domain.org/log/requests.log combined
</VirtualHost>
```

теперь, когда мы создали файлы виртуального хоста, нам нужно активировать их, чтобы Apache смог предоставлять их посетителям  
для этого нужно создать символьную ссылку для каждого виртуального хоста в директории sites-enabled:
```
sudo ln -s /etc/httpd/sites-available/domain.org.conf /etc/httpd/sites-enabled/domain.org.conf
```


### 8. Настройка разрешений SELinux для виртуальных хостов (рекомендуется)

сначала включим SELinux
```
sudo nano /etc/sysconfig/selinux
```
затем изменим директиву `SELinux=disabled` на
```
SELinux=enforcing
```

перезапустим операционную систему
```
shutdown -r now
```

проверим состояние работы SELinux
```
sestatus
```

изменение политик Apache для директории
```
sudo semanage fcontext -a -t httpd_log_t "/var/www/domain.org/log(/.*)?"
```

если будет ошибка `semanage command not found`, то используем
```
sudo yum provides /usr/sbin/semanage
```
находим в выводе примерно такую строку
```
policycoreutils-python-2.5-34.el7.x86_64
```

устанавливаем этот пакет
```
sudo yum install policycoreutils-python
```

снова
```
sudo semanage fcontext -a -t httpd_log_t "/var/www/domain.org/log(/.*)?"
```

затем воспользуемся командой restorecon для применения этих изменений и их сохранения между перезагрузками:
```
sudo restorecon -R -v /var/www/domain.org/log
```

введем команду
```
sudo ls -dZ /var/www/domain.org/log/
```
должно вывестись что-то подобное
```
root root system_u:object_r:httpd_log_t:s0 /var/www/domain.org/log/
```

после обновления контекста SELinux с помощью любого метода Apache сможет осуществлять запись в директорию `/var/www/domain.org/log`


### 9. Тестирование виртуального хоста

перезапустим службу Apache
```
sudo systemctl restart httpd
```

сформируем список содержимого директории `/var/www/domain.org/log`, чтобы убедиться, что Apache создал файлы журнала:
```
ls -lZ /var/www/domain.org/log
```
видим, что Apache удалось создать файлы error.log и requests.log


### 10. Подключаем домен

наш сайт сейчас открывается по ip

у домена пропишем ресурсные записи
```
@ A ip
* A ip
```

ждем, пока применятся

можно заходить на сайт по доменному имени


### 11. Остановим веб-сервер

```
sudo systemctl stop httpd
```

проверим текущее состояние
```
sudo systemctl status httpd
```


### 12. Ставим MySQL

```
sudo yum install mariadb mariadb-server
```

проверим статус
```
sudo systemctl status mariadb
```

запустим сервер MySQL
```
sudo systemctl start mariadb
```

запустим скрипт начальной настройки
```
sudo mysql_secure_installation
```
указываем новый пароль: `Set root password? [Y/n]` y  
удаляем анонимного пользователя: `Remove anonymous users? [Y/n]` y  
запрещаем вход администратора удаленно: `Disallow root login remotely? [Y/n]` y  
удаляем тестовую база данных: `Remove test database and access to it? [Y/n]` y  
применим сделанные изменения: `Reload privilege tables now? [Y/n]` y

попробуем подключиться к MySQL без пароля
```
mysql -u root
```
не пускает

попробуем подключиться к MySQL с паролем
```
mysql -u root -p
```
пустил

```
MariaDB [(none)]> show databases;
```
получаем список таблиц

```
MariaDB [(none)]> quit
```
выходим


### 13. Ставим PHP

добавляем еще один репозиторий
```
sudo rpm -Uvh http://rpms.remirepo.net/enterprise/remi-release-7.rpm
```

затем необходимо отредактировать конфиг репозитория, чтобы он начал работать  
открываем его через редактор nano:
```
sudo nano /etc/yum.repos.d/remi-php74.repo
```

находим строку `enabled=0` и меняем значение на `1`

обновляем кэш:
```
sudo yum update
```

и ставим нужную версию PHP:
```
sudo yum install php php-fpm php-gd php-mysql php-imagick php-dom php-opcache php-zip php-mbstring php-curl php-session php-json php-xml php-openssl
```

перейдем к настройке PHP, например, изменим лимиты
```
sudo nano /etc/php.ini
```

```
memory_limit = ...
post_max_size = ...
upload_max_filesize = ...
```


### 14. Установка phpMyAdmin (актуальная версия 5.1.1)

```
sudo yum --enablerepo=remi install phpmyadmin
```

далее
```
sudo nano /etc/httpd/conf.d/phpMyAdmin.conf
```
ищем блок
```
<Directory /usr/share/phpMyAdmin/>
```
в нем комментируем все `Require` и добавляем свой
```
Require all granted
```

запускаем веб-сервер
```
sudo systemctl start httpd
```

открываем phpMyAdmin по адресу:
```
http://SERVER_IP/phpmyadmin
```

изменим адрес phpMyAdmin, открываем
```
sudo nano /etc/httpd/conf.d/phpMyAdmin.conf
```
комментируем
```
Alias /phpMyAdmin /usr/share/phpMyAdmin
Alias /phpmyadmin /usr/share/phpMyAdmin
```
добавляем свой, например
```
Alias /pmaNotLook /usr/share/phpMyAdmin
```

чтобы активировать изменения, перезапустите веб-сервис
```
sudo systemctl restart httpd
```


### 15. Защитим страницу входа phpMyAdmin дополнительной авторизацией

откроем
```
sudo nano /etc/httpd/conf.d/phpMyAdmin.conf
```

В блок настроек каталога `/usr/share/phpMyAdmin` (но вне других внутренних блоков) нужно добавить директиву AllowOverride:
```
<Directory /usr/share/phpMyAdmin/>
AllowOverride All
```

перезапустим сервис, чтобы активировать изменения
```
sudo systemctl restart httpd
```

теперь директива AllowOverride будет направлять Apache в файл .htaccess в каталоге `/usr/share/phpMyAdmin`  
если веб-сервер находит такой файл, он выполняет указанные в нём директивы

создадим в каталоге `/usr/share/phpMyAdmin` файл .htaccess
```
sudo nano /usr/share/phpMyAdmin/.htaccess
```

внесем следующие данные:
```
AuthType Basic
AuthName "Admin Login"
AuthUserFile /etc/httpd/pma_pass
Require valid-user
```

теперь нужно создать файл паролей, указанный в настройках .htaccess, и внести в него пароли пользователей, у которых должен быть доступ к phpMyAdmin

для этого используется утилита htpasswd  
запустим команду, передав местонахождение файла и имя:
```
sudo htpasswd -c /etc/httpd/pma_pass striker
```
после запуска команды будет предложено ввести и подтвердить пароль этого пользователя

создав файл паролей, можно протестировать шлюз авторизации  
попробуем открыть страницу входа phpMyAdmin:
```
http://server_domain_or_IP/pmaNotLook
```


### 16. Изменим SSH-порт

создадим резервную копию текущего файла конфигурации демона ssh
```
date_format=`date +%Y_%m_%d:%H:%M:%S`
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config_$date_format
```

сделали копию, проверим
```
ls /etc/ssh/sshd_config*
```
получаем
```
/etc/ssh/sshd_config  /etc/ssh/sshd_config_2021_07_02:18:18:07
```

изменим порт службы SSH
```
sudo nano /etc/ssh/sshd_config
```
найдем строку
```
#Port 22
```
расскомментируем и изменим порт
```
Port 28351
```

разрешим новый SSH-порт в SELinux

сначала посмотрим текущий порт
```
semanage port -l | grep ssh
```
получаем
```
ssh_port_t                     tcp      22
```

если мы хотим разрешить привязку sshd к настроенному сетевому порту, нам нужно изменить тип порта на ssh_port_t
```
sudo semanage port -a -t ssh_port_t -p tcp 28351
```

убедимся, что новый порт был добавлен в список разрешенных портов для ssh
```
semanage port -l | grep ssh
```
получаем
```
ssh_port_t                     tcp      28351, 22
```

откроем порт SSH на Firewalld
```
sudo firewall-cmd --add-port=28351/tcp --permanent
sudo firewall-cmd --reload
```

теперь мы можем удалить службу SSH
```
sudo firewall-cmd --remove-service=ssh --permanent
sudo firewall-cmd --reload
```

перезапустим службу sshd
```
sudo systemctl restart sshd
```

установим пакет net-tools (в нем netstat)
```
sudo yum install net-tools
```

проверим адрес прослушивания для SSH
```
netstat -tunl | grep 28351
```
получаем
```
tcp        0      0 0.0.0.0:28351           0.0.0.0:*               LISTEN
tcp6       0      0 :::28351                :::*                    LISTEN
```

можем проверить так
```
ss -tnlp | grep ssh
```
получим
```
LISTEN     0      128          *:28351                    *:*                   users:(("sshd",pid=8037,fd=3))
LISTEN     0      128       [::]:28351                 [::]:*                   users:(("sshd",pid=8037,fd=4))
```


### 17. Изменим пароль для root

используем команду
```
passwd
```


### 18. Установим сертификат Let's Encrypt для домена

установим Acme.sh
```
curl https://get.acme.sh | sh
source ~/.bashrc
```

```
acme.sh --register-account -m info@domain.org
```

затем
```
acme.sh --issue -d domain.org -d *.domain.org --dns  --yes-I-know-dns-manual-mode-enough-go-ahead-please
```

добавим 2 TXT записи в домен

немного подождем и затем
```
acme.sh --renew -d domain.org -d *.domain.org --dns  --yes-I-know-dns-manual-mode-enough-go-ahead-please --force
```


### 19. Создание поддомена

создадим папку
```
mkdir /var/www/domain.org/html/subdomain
```

откроем
```
sudo nano /etc/httpd/sites-available/domain.org.conf
```

добавим следующий блок:
```
<VirtualHost *:80>
    ServerName subdomain.domain.org
    ServerAlias www.subdomain.domain.org
    DocumentRoot /var/www/domain.org/html/subdomain
    ErrorLog /var/www/domain.org/log/subdomain.error.log
    CustomLog /var/www/domain.org/log/subdomain.requests.log combined
</VirtualHost>
```

затем
```
sudo systemctl restart httpd
```


### 20. Установим zip

```
sudo yum install zip unzip
```

теперь можем загрузить на сервер zip-архив с сайтом и распаковать командой
```
unzip /var/www/domain.org/html/subdomain/www.zip -d /var/www/domain.org/html/subdomain
```
