# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Развертывание Kubernetes кластера с помощью Minikube

Установите [minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/) и [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/).
Запустите minikube:
```
minikube start
```
Активируйте Ingress:
```
minikube addons enable ingress
```

Создайте образ Django-приложения в кластере командой:
```
minikube image build -t имя_образа backend_main_django/
```
Отредактируйте файл `test-django-configmap-example.yaml`, подставив свои значения переменных окружения и переименовав файл в `test-django-configmap.yaml`.
Запустите Django-приложение в кластере командой:
```
kubectl apply -f kubernetes/
```

Установите и настройте [Helm](https://helm.sh/):
```
sudo snap install helm --classic
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install postgres-15 bitnami/postgresql
helm list
```

Следуя инструкциям, появившимся после создания пода базы данных, создайте создайте базу данных и пользователя:
```
CREATE DATABASE yourdbname;
CREATE USER youruser WITH ENCRYPTED PASSWORD 'yourpass';
GRANT ALL PRIVILEGES ON DATABASE yourdbname TO youruser;
ALTER USER youruser SUPERUSER;
```

Для применения миграций базы данных используйте команду:
```
kubectl apply -f kubernetes/test-django-migrate.yaml
```

Для создания суперпользователя используйте команду:
```
kubectl run ... -ti -- bash
```

Узнайте IP-адрес minikube:
```
minikube ip
```

Приложение будет доступно по адресу `star-burger.test` после добавления в файл `/etc/hosts` строки:
```
your_minikube_ip star-burger.test
```

Раз в месяц будет запускаться автоматическая очистка сессий Django-приложения.
Если возникла необходимость очистить сессии Django-приложения не по расписанию, то это можно сделать в ручную:
```
kubectl create job --from=cronjob/test-django-clearsessions any_job_name
```

После внесения изменений в конфигурационный файл `test-django-configmap.yaml` введите следующие команды:
```
kubectl apply -f kubernetes/
kubectl rollout restart -f kubernetes/test-django-deployment.yaml
```

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
