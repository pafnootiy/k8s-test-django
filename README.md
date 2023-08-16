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

## Развертывание Django приложения в Minikube

Запустите Minikube с помощью следующей команды:
```shell-session
apiVersion: v1
kind: ConfigMap
metadata:
  name: ваш-configmap
data:
  DJANGO_DEBUG: "false"
  DJANGO_SECRET_KEY: "ваш_секретный_ключ"
```
Примените манифест ConfigMap к Minikube кластеру:
```shell-session
kubectl apply -f configmap.yaml

```

Создайте файл с именем configmap.yaml для ConfigMap. Добавьте следующее содержимое:
```shell-session
minikube start
```

Создайте файл манифеста для развертывания вашего Django приложения. Создайте файл с именем deployment.yaml и добавьте следующее содержимое:

```shell-session
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-django-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: your-django-app
  template:
    metadata:
      labels:
        app: your-django-app
    spec:
      containers:
        - name: your-django-app-container 
          image: your-django-app-image:latest
          envFrom:
            - configMapRef:
                name: your-configmap
          ports:
            - containerPort: 8000
```

Примените манифест развертывания к Minikube кластеру:

```shell-session
kubectl apply -f deployment.yaml
```
Создайте файл манифеста для сервиса и назовите его service.yaml. Добавьте следующее содержимое:

```shell-session
apiVersion: v1
kind: Service
metadata:
  name:  django-svc
spec:
  selector:
    app: your-django-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

Примените манифест сервиса к Minikube кластеру:
```shell-session
kubectl apply -f service.yaml

```

Создайте файл манифеста для Ingress и назовите его ingress.yaml. Добавьте следующее содержимое:
```shell-session
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: you-django-app
spec:
  rules:
    - host: your-host-ingress.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django-svc
                port:
                  number: 8000
```
Примените манифест Ingress к Minikube кластеру:
```shell-session
kubectl apply -f ingress.yaml
```
Получите IP Minikube с помощью следующей команды:
```shell-session
minikube ip
```
Добавьте запись в файл /etc/hosts на вашей хостовой системе, чтобы ассоциировать IP Minikube с вашим выбранным хостом. 
```shell-session
<Minikube-IP> your-host-ingress.local
```
Теперь ваше Django приложение должно быть доступно по адресу http://you-host-ingress.local в вашем веб-браузере.





## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
