# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


### Как добавить переменные окружения в kubernetes Secret

Закодируйте переменные окружения в base64:
```bash
echo -n 'my-secret-key' | base64
echo -n 'postgres://myuser:mypass@postgres-db-postgresql.default.svc.cluster.local:5432/mydatabase' | base64
echo -n '*' | base64
echo -n 'False' | base64
```

Заполните `secret.yaml`:

```shell
data:
  SECRET_KEY: bXktc2VjcmV0LWtleQ==
  DATABASE_URL: cG9zdGdyZXM6Ly9teXVzZXI6bXlwYXNzQHBvc3RncmVzLWRiLXBvc3RncmVzcWwuc2VydmljZS5jbHVzdGVyLmxvY2FsOjU0MzIvbXlkYXRhYmFzZQ==
  ALLOWED_HOSTS: Kg==
  DEBUG: RmFsc2U=
```


### Запуск манифеста

Чтобы открыть сайт запустите:

```shell
kubectl apply -f kubernetes/django-web-secret.yaml \
              -f kubernetes/django-web-deployment.yaml \
              -f kubernetes/django-web-service.yaml \
              -f kubernetes/django-web-ingress.yaml
```

Получите ip отсюда:
```shell
minikube ip
```

И добавьте его в hosts файл по [этой инструкции](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/rabota-s-dns-serverami/fayl-hosts-gde-nakhoditsya-i-kak-yego-izmenit)
Сайт будет доступен по адресу http://star-burger.test/


Также, для автоматической очистки БД от сессий запустите:

```shell
kubectl apply -f kubernetes/django-clearsessions.yaml
```

Для запуска миграций:

```shell
kubectl apply -f kubernetes/django-migrate.yaml
```

### БД внутри кластера

Установите [helm](https://artifacthub.io/packages/helm/bitnami/postgresql) и создайте БД внутри пода:
```
sudo -u postgres psql
postgres=# create database mydb;
postgres=# create user myuser with encrypted password 'mypass';
postgres=# grant all privileges on database mydb to myuser;
```

## Деплой в yandex cloud с использованием Lens Desktop

1. Создайте* Pod для образа [nginx](https://hub.docker.com/_/nginx), который будет иметь ваш `namespace`. Например, [так](envs/yc-sirius-test/nginx-pod.yaml).
2. Создайте Service для этого Pod'а. Например, вот [так](envs/yc-sirius-test/nginx-service.yaml).

*Для создания манифеста в Lens, нужно нажать на `+` (New tab) внизу на панели и выбрать "Create resource".

## Как подготовить dev окружение

Скачайте ssl-сертификат с [облака Яндекс](https://storage.yandexcloud.net/cloud-certs/RootCA.pem).
Создайте kubernetes secret:
```commandline
kubectl create secret generic ssl-cert-secret   --from-file=RootCA.pem=/путь/к/сертификату/RootCA.pem   --namespace=<namespace>
```
Замените `/путь/к/сертификату/RootCA.pem` и `<namespace>`.

Для тестирования подключения к базе данных postgres, создайте манифест вроде [этого](envs/yc-sirius-test/postgres-client.yaml).