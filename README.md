# asweb03
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
<img width="616" alt="2023-06-02 22_14_15-site" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/768fa155-1fb6-4967-852e-22a4cd8e29c9">


Контейнер Apache HTTP Server

Для начала создаю конфигурационный файл для Apache HTTP Server. Для этого выполняю следующие команды в консоли:

```shell
# команда скачивает образ httpd и запускает на его основе контейнер с именем httpd
docker run -d --name httpd  httpd:2.4

# копируем конфигурационный файл из контейнера в папку .\files\httpd
docker cp httpd:/usr/local/apache2/conf/httpd.conf .\files\httpd\httpd.conf

<img width="696" alt="2023-06-02 22_27_59-Editing asweb03_Updated_README md at main · AlexVor550_asweb03_Updated · GitHub " src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/60a96a9f-3ed5-4b10-8cbc-9300e32a1924">

# останавливаем контейнер httpd
docker stop httpd

# удаляем контейнер
docker rm httpd

<img width="255" alt="2023-06-02 22_26_56-asweb03 containers vhosts ru (2) md - Visual Studio Code" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/199de0ee-43c3-4e37-96e8-528f88cd2703">
```


В созданом файле `.\files\httpd\httpd.conf` раскоментирую строки, содержащие подключение расширений `mod_proxy.so`, `mod_proxy_http.so`, `mod_proxy_fcgi.so`.
<img width="469" alt="2023-06-02 22_31_16-httpd conf - Visual Studio Code" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/f259fe0a-4c9c-47ab-9d99-448573f99398">


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

<img width="234" alt="2023-06-02 22_36_13-httpd conf - Visual Studio Code" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/f3b638b2-7c56-4a54-ba6e-a2cc2c715fe4">


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

<img width="530" alt="2023-06-02 22_37_10-asweb03" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/98bf4174-d7b8-4fb8-9da3-547b8d988086">


## Запуск и тестирование

В командной строке открываю директорию с работой и выполняю команду:

```shell 
docker-compose build
```
<img width="851" alt="2023-06-02 22_05_03-ÐœÐµÐ»ÑŒÐ½Ð¸Ðº_ÐÐ½Ñ‚Ð¾Ð½Ð¸Ð½Ð°_Ð»Ð°Ð±3 docx и еще 8 страниц — Личный_ Microso" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/4971aa76-835f-4bde-a4b0-ac330f7119c6">


На основе созданных определений docker построит образы сервисов.За 5,3 секунды построились образы

Выполняю слудующию команду:

```shell
docker-compose up -d
```
<img width="842" alt="2023-06-02 22_05_29-ÐœÐµÐ»ÑŒÐ½Ð¸Ðº_ÐÐ½Ñ‚Ð¾Ð½Ð¸Ð½Ð°_Ð»Ð°Ð±3 docx и еще 8 страниц — Личный_ Microso" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/689c3b35-16f7-4583-b62e-76f34d80ac31">


На основе образов запускаются контейнеры. 

Открываю в браузере страницу: http://wordpress.localhost и произвожу установку сайта. 

Имя пользователя базы данных, его пароль и название базы данных беру из файла `docker-compose.yml`

<img width="680" alt="2023-06-01 16_00_54-WordPress › Setup Configuration File и еще 6 страниц — Личный_ Microsoft​ Edge" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/c8e56e28-6a00-4d5f-9d6f-335186de7600">


Далее прописываю имя сайта,имя пользователя и пароль для входа.
<img width="561" alt="2023-06-01 16_01_50-WordPress › Installation и еще 6 страниц — Личный_ Microsoft​ Edge" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/e62a0a13-e643-440c-8d9d-80c6950ec070">


После регистрации ввожу данные с регистрации для входа и вижу приветсвенную страницу "Hello world"
<img width="871" alt="2023-06-01 16_03_27-AlexVor и еще 6 страниц — Личный_ Microsoft​ Edge" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/1fdd040c-5d25-4e6c-9cf3-039ffea0b245">


Выполняю последовательно следующие команды

```shell
# остановить контейнеры
docker-compose down
# удалить контейнеры
docker-compose rm
```

Открываю в браузере страницу: http://wordpress.localhost и вижу ошибку подключения .

Снова запускаю кластер контейнеров:

```shell
docker-compose up -d
```
<img width="871" alt="2023-06-01 16_03_27-AlexVor и еще 6 страниц — Личный_ Microsoft​ Edge" src="https://github.com/AlexVor550/asweb03_Updated/assets/107479058/c6c2ad4e-e4be-47fb-b715-b630a5017f40">


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


