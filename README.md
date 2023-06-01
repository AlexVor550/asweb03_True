# asweb03_True
3 лабораторная работа на Docker для допуска к сессии

Цель работы
Целью данной работы является ознакомление с запуском сайтов (виртуальных хостов) в контейнере. Также студент ознакомится со способом создания кластера контейнеров и рассмотрит установку сайтов на базе Wordpress (1).

В результате выполнения данной работы будет получен кластер из 3-х контейнеров:

 - __Apache HTTP server__ - контейнер принимает запросы и переправляет их серверу PHP-FPM;
 - __PHP-FPM__ - контейнер с сервером PHP-FPM;
 - __MariaDB__ - контейнер с серером баз данных.

Подготовка

Проверьте, что у вас установлен и запущен _Docker Desktop_.

Создайте папку `asweb03`. В ней будет выполняться вся работа.

Создайте папки `database` - для базы данных, `files` - для хранения конфигураций и `site` - в данной папке будет расположен сайт.

Лабораторная работа выполняется при подключении к сети Internet, так как скачиваются образы из репозитория hub.docker (2).

Выполнение

Скачиваю [CMS Wordpress](https://wordpress.org/) и распаковаваю в папку `site`.В папке `site` появляется папка `wordpress` с исходным кодом сайта.

Контейнер Apache HTTP Server

Для начала создаю конфигурационный файл для Apache HTTP Server. Для этого выполняю следующие команды в консоли:

```shell
# команда скачивает образ httpd и запускает на его основе контейнер с именем httpd
docker run -d --name httpd  httpd:2.4

# копируем конфигурационный файл из контейнера в папку .\files\httpd
docker cp httpd:/usr/local/apache2/conf/httpd.conf .\files\httpd\httpd.conf

# останавливаем контейнер httpd
docker stop httpd

# удаляем контейнер
docker rm httpd
```

В созданом файле `.\files\httpd\httpd.conf` раскоментирую строки, содержащие подключение расширений `mod_proxy.so`, `mod_proxy_http.so`, `mod_proxy_fcgi.so`.

В конфигурационном файле объявление параметра `ServerName`. Под ним добавляю следующие строки:

```
# определение доменного имени сайта
ServerName wordpress.localhost:80
# перенаправление php запросов контейнеру php-fpm
ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php-fpm:9000/var/www/html/$1
# индексный файл
DirectoryIndex /index.php index.php
```

Также нахожу определение параметра `DocumentRoot` и задаю ему значение `/var/www/html`, как и в следующей за параметром строке.

Создаю файл `Dockerfile.httpd` со следующим содержимым:

```dockerfile
FROM httpd:2.4

RUN apt update && apt upgrade -y

COPY ./files/httpd/httpd.conf /usr/local/apache2/conf/httpd.conf
```

### Контейнер PHP-FPM

Создаю файл `Dockerfile.php-fpm` со следующим содержимым:

```dockerfile
FROM php:7.4-fpm

RUN apt-get update && apt-get upgrade -y && apt-get install -y \
		libfreetype6-dev \
		libjpeg62-turbo-dev \
		libpng-dev
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
	&& docker-php-ext-configure pdo_mysql \
	&& docker-php-ext-install -j$(nproc) gd mysqli
```

### Контейнер MariaDB

Создаю файл `Dockerfile.mariadb` со следующим содержимым:

```dockerfile
FROM mariadb:10.8

RUN apt-get update && apt-get upgrade -y
```

### Сборка решения

Создаю файл `docker-compose.yml` со следующим содержимым:

```yaml
version: '3.9'

services:
  httpd:
    build:
      context: ./
      dockerfile: Dockerfile.httpd
    networks:
      - internal
    ports:
      - "8080:80"
    volumes:
      - "./site/wordpress/:/var/www/html/"
  php-fpm:
    build:
      context: ./
      dockerfile: Dockerfile.php-fpm
    networks:
      - internal
    volumes:
      - "./site/wordpress/:/var/www/html/"
  mariadb:
    build: 
      context: ./
      dockerfile: Dockerfile.mariadb
    networks:
      - internal
    environment:
     MARIADB_DATABASE: sample
     MARIADB_USER: sampleuser
     MARIADB_PASSWORD: samplepassword
     MARIADB_ROOT_PASSWORD: rootpassword
    volumes:
      - "./database/:/var/lib/mysql"
networks:
  internal: {}
```

Данный файл объявляет структуру из трех контейнеров: http как точка входа, контейнер php-fpm и контейнер с базой данных. Для взаимодействия контейнеров объявляется также сеть `internal` с настройками по умолчанию.

## Запуск и тестирование

В командной строке открываю директорию с работой и выполняю команду:

```shell 
docker-compose build
```

На основе созданных определений docker построит образы сервисов.За 5,3 секунды построились образы

Выполняю слудующию команду:

```shell
docker-compose up -d
```

На основе образов запускаются контейнеры. 

Открываю в браузере страницу: http://wordpress.localhost и произвожу установку сайта. 

Имя пользователя базы данных, его пароль и название базы данных беру из файла `docker-compose.yml`

Далее прописываю имя сайта,имя пользователя и пароль для входа.

После регистрации ввожу данные с регистрации для входа и вижу приветсвенную страницу "Hello world"

Выполняю последовательно следующие команды

```shell
# остановить контейнеры
docker-compose down
# удалить контейнеры
docker-compose rm
```

Открываю в браузере страницу: http://wordpress.localhost и вижу ошибку подключения . Снова запускаю кластер контейнеров:

```shell
docker-compose up -d
```

Благодаря повторному запуску кластера страница: http://wordpress.localhost снова функционирует

Отчет

В данной лабораторной работе я познакомился с одним из главных функуионалов Docker,а именно контейнеризация и создание кластера из контейнеров.Я узнал о возможности установки wordpress имея только Docker и прописанный код для создания контейнеров

В результате выполнения данной работы был создан кластер из 3-х контейнеров:

Apache HTTP server - контейнер принимает запросы и переправляет их серверу PHP-FPM;
PHP-FPM - контейнер с сервером PHP-FPM;
MariaDB - контейнер с серером баз данных.

Ответы на вопросы:
Как скопировать файл из контейнера в хостовый компьютер?

Чтобы скопировать файл из контейнера в хостовый компьютер необходимо использовать команду docker cp.

Синтаксис команды выглядит следующим образом:

docker cp [OPTIONS] CONTAINER: path_to_file_inside_container path_on_host_machine

За что отвечает директива `DocumentRoot` в конфигурационном файле Apache HTTP Server?

Директива DocumentRoot в конфигурационном файле Apache HTTP Server отвечает за определение каталога, в котором находятся файлы, доступные для обслуживания по HTTP протоколу. 

В файле `docker-compose.yml` папка `database` хоста монтируется в папку `/var/lib/mysql` контейнера `mariadb`.

Для чего монтируют к контейнеру базы данных папку?

Монтирование папки database хоста в папку /var/lib/mysql контейнера mariadb позволяет сохранить данные базы данных на хосте, а не внутри контейнера. Это помогает, если нужно сохранить данные даже после удаления контейнера, а также для резервного копирования и восстановления данных.Также, при монтировании папки хоста в контейнер, база данных в контейнере может работать с данными на хосте, что может улучшить производительность, поскольку работа с данными на хосте может быть эффективнее, чем работа с данными внутри контейнера.


