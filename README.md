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

## Развёрытвание с помощью Minikube
Перед началом работы убедитесь что у вас установлены следующие инструменты:
- [VirtualBox](https://virtualbox.org)
- [Minikube](https://minikube.sigs.k8s.io)
- [Kubernetes kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)

### Запускаем кластер Minikube
Запустите Minikube:
```shell
minikube start
```
### Настройка и запуск базы данных
Установите инструмент для управления приложениями Kubernetes [Helm](https://helm.sh/)

Добавьте Helm Chart Repository:

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

В директории проекта создайте файл postgres-value.yaml с конфигурацией базы данных :
Пример:
```
auth:
  enablePostgresUser: true
  postgresPassword: {your database password}
  username: {your username}
  password: {your user`s password}
  database: {your database`s name}
```

Установите PostgreSQL, используя созданный конфигурационный файл:

```
helm install postgres bitnami/postgresql -f postgres-config.yaml
```

### Настройка и запуск Django в Kubernetes
Будем использоваться готовый image с Django. Если вы хотите использовать готовый образ с Django, отредактируйте файл kubernetes/django-service.yml, измените значение image на адрес нужного образа на Docker Hub или созданного локально.

Откройте файл kubernetes/django-config.yaml и укажите нужные значения для Django. В DATABASE_URL укажите данные в формате postgres://username:password@db_host:db_port/database_name, где username, password, database_name - данные из файла postgres-config.yaml, db_host - значение, указанное в команде helm install, db_port - по умолчанию 5432.

Регистрируем конфиг:
```shell
kubectl apply -f kubernetes/configmap.yaml
```
Регистрируем secret файл:
```shell
kubectl apply -f kubernetes/django-secret.yaml
```
Запускаем Django и сервис:
```shell
kubectl apply -f kubernetes/deployment.yaml
```
```shell
kubectl apply -f kubernetes/django-service.yaml
```


Создаём миграции:
```shell
kubectl apply -f kubernetes/migrate.yaml
```
Создаём службу для очистки устаревших сессий:
```shell
kubectl apply -f kubernetes/clearsessions.yaml
```
Запускаем Ingress:
Добавьте выбранный хост(в примере используется star-burger.test) в /etc/hosts(На Linux и macOS, файл hosts обычно находится в /etc/hosts.
На Windows, файл hosts обычно находится в C:\Windows\System32\Drivers\etc\hosts.). Ip можно узнать с помощью команды minikube ip.
```shell
minikube ip
```
Выполните команду:
```shell
kubectl apply -f kubernetes/ingres.yaml
```

## Настройка и запуск Django в Yandex Cloud

Работающая версия сайта находится по адресу: [edu-trusting-darwin.sirius-k8s.dvmn.org](https://edu-trusting-darwin.sirius-k8s.dvmn.org/)

Страничка с выделенными ресурсами: [https://sirius-env-registry.website.yandexcloud.net/edu-trusting-darwin.html](https://sirius-env-registry.website.yandexcloud.net/edu-trusting-darwin.html)

### Установка и подключение к Yandex Cloud (CLI)
1.Следуйте инструкциям на официальном сайте [Yandex Cloud](https://cloud.yandex.com/en/docs/cli/quickstart), чтобы установить CLI для вашей операционной системы.

2.После установки CLI выполните команду `yc init`, чтобы авторизоваться и выбрать нужный проект.

### Деплой кода

#### Загрузка Docker Image на [DockerHub](https://hub.docker.com/)
1.Зарегистрируйте аккаунт на [DockerHub](https://hub.docker.com/)
2.Локально соберите Docker-образ командой:

```
docker build -t <image-name>:<tag> 
```

3.Загрузите образ на [DockerHub](https://hub.docker.com/), предварительно заменив <YOUR-USERNAME> на ваш идентификатор Docker ID:

```
docker tag <image-name>:<tag> YOUR-USERNAME/<image-name>:<tag>
```
```
docker push YOUR-USERNAME/<image-name>:<tag>
```
#### Подготовка dev окружения
1.В файле configmap.yaml подставьте свои значения переменных окружения.

Создайте config-файл:

```
kubectl -n <namespace> apply -f configmap.yaml
```

2.Получите SSL-сертификат для подключения к PostgreSQL:

Linux/Bash или macOS(Zsh):
```
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt
```
Windows(PowerShell):
```
mkdir $HOME\.postgresql; curl.exe -o $HOME\.postgresql\root.crt https://storage.yandexcloud.net/cloud-certs/CA.pem
```
Создайте Secret с сертификатом:

```
kubectl create secret generic postgresql-ssl -n <namespace> --from-file=/path_to/root.crt
```
`/path_to/root.crt` путь где лежит root.crt
#### Запуск приложения
1.Разверните Django-приложение в кластере:

```
kubectl -n <namespace> apply -f deployment.yaml
```
2.В файле service.yaml замените значение nodePort: 30371 на свое, согласно настройкам ALB-роутера.

Запустите сервис:

```
kubectl -n <namespace> apply -f service.yaml
```

3.Примените миграции:
```
kubectl -n <namespace> apply -f migrate.yaml
```

4.Для очистки сессий, запустите clearsessions:
```
kubectl -n <namespace> apply -f clearsessions.yaml
```
Теперь ваш сайт доступен по ссылке [edu-trusting-darwin.sirius-k8s.dvmn.org](https://edu-trusting-darwin.sirius-k8s.dvmn.org/).






## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

