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

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Инструкция по деплою через Minikube

Для деплоя понадобится:

- kubectl. [Установка и настройка](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/);
- Minikube. [Установка](https://minikube.sigs.k8s.io/docs/start/);
- [VirtualBox](https://www.virtualbox.org/);

Для запуска minikube используйте [драйвер VirtualBox](https://minikube.sigs.k8s.io/docs/drivers/virtualbox/).
Проверьте корректность запуска следующей:

```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.59.100:8443
CoreDNS is running at https://192.168.59.100:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'
```

1. Запустите СУБД с помощью Docker Compose:

```shell
$ docker-compose run -d db
```

2. Загрузите образ Django в кластер:

```shell
$ minikube image load <имя образа с тегом>
```

3. Проверьте, что образ загрузился:

```shell
$ minikube image ls
```

4. Создайте файл в папке `kubernetes` с разрешением `.yaml` со следующим содержимым:

```yaml
apiVersion : v1
kind : ConfigMap
metadata :
  name: django-config
  labels:
    env: prod
data :
  SECRET_KEY : 'Смотри раздел про переменные окружения'
  DATABASE_URL : 'postgres://POSTGRES_USER:POSTGRES_PASSWORD@<Внешний IP вашего ПК>:<PORT - 5432>/POSTGRES_DB'
  ALLOWED_HOSTS : 'IP вашего minikube (можно узнать командой minikube ip)'
```

5. Создайте `ConfigMap`:

```shell
$ kubectl apply -f ./kubernetes/<название файла>.yaml
```

6. Включите надстройку minikube ingress:

```shell
$ minikube addons enable ingress
```

8. Создайте объект `Ingress`:

```shell
$ kubectl apply -f ./kubernetes/ingress-hosts.yaml
```

8. Убедитесь, что IP-адрес установлен:

```shell
$ kubectl get ingress
```

Вы должны увидеть IPv4-адрес в столбце `ADDRESS`:

```shell
NAME             CLASS   HOSTS              ADDRESS          PORTS   AGE
django-ingress   nginx   star-burger.test   192.168.59.107   80      30s
```

9. Добавьте следующую строку в конец файла `hosts` на вашем компьютере (потребуется права администратора):

```shell
<minikube ip> star-burger.test
```

Путь к файлу `hosts`:

- Windows10 - `C:\Windows\System32\drivers\etc\hosts`
- Linux - `/etc/hosts`
- Mac OS X - `/private/etc/hosts`

10. Сайт доступен по адресу [http://star-burger.test](http://star-burger.test).

При изменении переменных окружения в `ConfigMap` выполните следующие команды:

```shell
$ kubectl apply -f ./kubernetes/django-config.yaml
$ kubectl delete pods -l project=dj-app
```

11. Запустите регулярное удаление сессий (опционально):

```shell
$ kubectl apply -f kubernetes/cronjob-django-clearsessions.yaml
```

Для принудительного удаления сессий:

```shell
$ kubectl create job --from=cronjob/django-clearsessions-cron django-clearsessions-job
```