# Домашнее задание к занятию "13.1 контейнеры, поды, deployment, statefulset, services, endpoints"
Настроив кластер, подготовьте приложение к запуску в нём. Приложение стандартное: бекенд, фронтенд, база данных (пример можно найти в папке 13-kubernetes-config).

## Задание 1: подготовить тестовый конфиг для запуска приложения
Для начала следует подготовить запуск приложения в stage окружении с простыми настройками. Требования:
* под содержит в себе 3 контейнера — фронтенд, бекенд, базу;
* регулируется с помощью deployment фронтенд и бекенд;
* база данных — через statefulset.

## Решение 1

Для начала соберем контейнеры из dockerfile и запушим их в свой докерхаб
[frontend](https://hub.docker.com/r/hatsker/frontend)
[backend](https://hub.docker.com/r/hatsker/backend)

создадим namespace "stage"
```commandline
alexp@cp1:~$sudo kubectl create namespace stage
namespace/stage created
```
создадим yaml файл для деплоя

```commandline
alexp@cp1:~$ touch stage.yaml
```
содерижимое файла

```commandline
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-and-backend
  namespace: stage
  labels:
    app: testapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: testapp
  template:
    metadata:
      labels:
        app: testapp
    spec:
      containers:
      - name: frontend
        image: hatsker/frontend
        ports:
        - containerPort: 80
      - name: backend
        image: hatsker/backend
        ports:
        - containerPort: 9000
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db-test
  namespace: stage
spec:
  selector:
    matchLabels:
      app: db
  serviceName: "db-test"
  replicas: 1
  template:
    metadata:
      labels:
        app: db
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: db
        image: postgres:13-alpine
        ports:
        - containerPort: 5432
        env:
          - name: POSTGRES_PASSWORD
            value: postgres
          - name: POSTGRES_USER
            value: postgres
          - name: POSTGRES_DB
            value: news
```
результат 

![1](/img/dz_13_1_1.png)


## Задание 2: подготовить конфиг для production окружения
Следующим шагом будет запуск приложения в production окружении. Требования сложнее:
* каждый компонент (база, бекенд, фронтенд) запускаются в своем поде, регулируются отдельными deployment’ами;
* для связи используются service (у каждого компонента свой);
* в окружении фронта прописан адрес сервиса бекенда;
* в окружении бекенда прописан адрес сервиса базы данных.

## Решение 2

Создадим namespace "Prod"
```commandline
alexp@cp1:~$ sudo kubectl create namespace prod
namespace/prod created
```
создадим yaml файл для деплоя

```commandline
alexp@cp1:~$ touch prod.yaml
```
содерижимое файла

```commandline
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db-prod
  namespace: prod
spec:
  selector:
     matchLabels:
       app: db
  serviceName: "db-svc"
  replicas: 1
  template:
     metadata:
       labels:
         app: db
     spec:
       terminationGracePeriodSeconds: 10
       containers:
       - name: db
         image: postgres:13-alpine
         ports:
         - containerPort: 5432
         env:
           - name: POSTGRES_PASSWORD
             value: postgres
           - name: POSTGRES_USER
             value: postgres
           - name: POSTGRES_DB
             value: news

# Сервис для базы данных
---
apiVersion: v1
kind: Service
metadata:
  name: db-svc
  namespace: prod
spec:
  selector:
     app: db
  ports:
     - protocol: TCP
       port: 5432
       targetPort: 5432>              

# Деплоймент бэкенда в отдельный под с сервисом БД в окружении
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
  namespace: prod
  labels:
     app: backend
spec:
  replicas: 1
  selector:
     matchLabels:
       app: backend
  template:
     metadata:
       labels:
         app: backend
     spec:
       containers:
       - name: backend
         image: hatsker/backend
         ports:
         - containerPort: 9000
         env:
           - name: DATABASE_URL
             value: postgres://postgres:postgres@db-svс:5432/news

# Сервис бэкенда
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: prod
spec:
  selector:
     app: backend
  ports:
     - protocol: TCP
       port: 9000
       targetPort: 9000

# Деплоймент фронтенда в отдельный под с сервисом бэкенда в окружении
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
  namespace: prod
  labels:
     app: frontend
spec:
  replicas: 1
  selector:
     matchLabels:
       app: frontend
  template:
     metadata:
       labels:
         app: frontend
     spec:
       containers:
       - name: frontend
         image: hatsker/frontend
         ports:
         - containerPort: 80
         env:
           - name: BASE_URL
             value: http://backend:9000

# Сервис фронтенда
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: prod
spec:
  selector:
     app: frontend
  ports:
     - name: web
       protocol: TCP
       port: 8000
       targetPort: 80
```
задеплоим

```commandline
alexp@cp1:~$ sudo kubectl create -f prod.yaml -n prod
statefulset.apps/db-prod created
service/db-svc created
deployment.apps/backend-deploy created
service/backend-svc created
deployment.apps/frontend-deploy created
service/frontend-svc created
```
### Результат

![2](/img/dz_13_1_2.png)




## Задание 3 (*): добавить endpoint на внешний ресурс api
Приложению потребовалось внешнее api, и для его использования лучше добавить endpoint в кластер, направленный на это api. Требования:
* добавлен endpoint до внешнего api (например, геокодер).

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

В качестве решения прикрепите к ДЗ конфиг файлы для деплоя. Прикрепите скриншоты вывода команды kubectl со списком запущенных объектов каждого типа (pods, deployments, statefulset, service) или скриншот из самого Kubernetes, что сервисы подняты и работают.

---
