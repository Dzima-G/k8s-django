# Django Site

Докеризированный сайт на Django для развертки в Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет
сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и
Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и
Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

---
# 🔐 Переменные окружения

Образ с Django считывает настройки из переменных окружения.  
Создайте файл `.env` в корневом каталоге и запишите туда данные в формате: `ПЕРЕМЕННАЯ=значение`

- **`SECRET_KEY`** — обязательная секретная настройка Django.  
  Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно.  
  [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key)
- **`DEBUG`** — настройка Django для включения отладочного режима.  
  Принимает значения `TRUE` или `FALSE`.  
  [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG)

- **`ALLOWED_HOSTS`** — настройка Django со списком разрешённых адресов.  
  Если запрос прилетит на другой адрес, то сайт ответит ошибкой **400 Bad Request**.  
  Можно перечислить несколько адресов через запятую, например: `127.0.0.1,192.168.0.1,site.test`
  [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts)
- **`DATABASE_URL`** — адрес для подключения к базе данных PostgreSQL.  
  Другие СУБД не поддерживаются.  
  [Формат записи](https://github.com/jacobian/dj-database-url#url-schema)
- 
Не обязательные переменные окружения:
- **`POSTGRES_SSL_SERT`** - сертификат для подключения к [Yandex Cloud Managed PostgreSQL](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect) в кодировке **base64**.
---

# Запустить сайт для локальной разработки
## 🐳 Запуск локально в Doker
1. **Подготовить окружение к локальной разработке:**

    Для запуска необходим установленный [Docker Desktop](https://www.docker.com/get-started/).
2. **Запустить базу данных и сайт:**
    ```shell
    cd local_deployment/docker_compose # перейти в каталог Compose-файлом
    docker compose up # собрать образ
    ```
 
3. Создать миграции и суперпользователя (в отдельном терминале):
    Перейти в каталог с `docker-compose`:
    ```shell
    cd local_deployment/docker_compose # перейти в каталог Compose-файлом
    docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
    docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
    ```
    
    После запуска сайт будет доступен по адресу: [http://127.0.0.1:8080](http://127.0.0.1:8080).

    Админка - [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).
    

## 🧩 Как вести разработку
Все файлы с кодом django смонтированы внутрь docker-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не
требовал постоянно пересборки docker-образа -- достаточно перезапустить сервисы Docker Compose.

1. **Обновить приложение:**

    ```shell
    git pull
    cd local_deployment/docker_compose # перейти в каталог Compose-файлом
    docker compose build # пересобирает образы
    ```

    После обновления — примените миграции:

    ```shell
    cd local_deployment/docker_compose # перейти в каталог Compose-файлом
    docker compose run --rm web ./manage.py migrate # обновить миграции
    ```
   
2. **Добавить библиотеку в зависимости:**

    Добавьте пакет в `requirements.txt` и пересоберите образ для веб-сервиса:

    ```shell
    docker compose build web # пересобирает образ для сервиса web
    ```
**Примечание:**
Каталог local_deployment/docker_compose служит для локального запуска проекта в Docker.

---
## ☸️ Запуск локально с использованием кластера Minikube

Для запуска необходимы:

- [См. документацию Docker Desktop](https://www.docker.com/get-started/)

- [См. документацию сubectl](https://kubernetes.io/docs/tasks/tools/)

- [См. документацию minikube](https://kubernetes.io/docs/tasks/tools/)


1. **Запустить кластер Minikube:**

    ```shell
    minikube start --driver=virtualbox
    ```
    * *Вместо docker можно использовать другой драйвер (например, docker), если он у вас настроен:*

        ```shell
        minikube start --driver=docker
        ```
2. **Передача чувствительных данных:**

   Создайте файл `local_deployment/k8s_minikube/secrets.yaml`, заменив значения переменных на свои (*см. раздел переменные окружения*):

    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: django-secrets
    type: Opaque
    stringData:
      DEBUG: "False"
      SECRET_KEY: "your_secret_key"
      DATABASE_URL: "your_database_url"
      ALLOWED_HOSTS: "allowed_hosts"
      DJANGO_SUPERUSER_USERNAME: "username"
      DJANGO_SUPERUSER_PASSWORD: "password"
      DJANGO_SUPERUSER_EMAIL: "your-email@example.com"
    ```

   Примените:
    ```shell
    cd local_deployment/k8s_minikube # перейти в каталог c манифестом
    kubectl apply -f kubernetes/secrets.yaml
    ```
3. **Запуск базы данных PostgreSQL (*в Minikube*):**
    1. Установите Helm [см. документацию Helm](https://helm.sh/)
    2. Создайте файл `local_deployment/k8s_minikube/postgres-values.yaml` и укажите свои данные доступа::
        ```
        architecture: "standalone"

        auth:
            username: "username"
            password: "password"
            database: "nameDB"
        ```
    3. Установите PostgreSQL через Helm:
        ```shell
        helm install my-postgres oci://registry-1.docker.io/bitnamicharts/postgresql -f local_deployment/k8s_minikube/postgres-values.yaml  
        ```
4. **Запуск Django-приложения и сервиса:**
    ```shell
    kubectl apply -f local_deployment/k8s_minikube/django-deployment.yaml
    kubectl apply -f local_deployment/k8s_minikube/django-service.yaml
    ```
5. **Выполнить миграции:**
    ```shell    
    kubectl apply -f local_deployment/k8s_minikube/django-migrate.yaml
    ```
6. **Создать суперпользователя:**
    ```shell    
    kubectl apply -f local_deployment/k8s_minikube/django-superuser.yaml
    ```
7. **(*Опционально)* Настроить Ingress:**
    1. Установить ingress Controllers [см. таблицу доступных контролееров](https://docs.google.com/spreadsheets/d/191WWNpjJ2za6-nbG4ZoUMXMpUK8KlCIosvQB0f-oq3k/edit?gid=907731238#gid=907731238), например [Сontour](https://projectcontour.io/getting-started/):
        ```shell    
        kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
        ```
    2. Отредактируйте файл `local_deployment/k8s_minikube/ingress-hosts.yaml`, укажите свой домен:
        ```
        ...
        spec:
          rules:
            - host: your-domain.test
        ...
        ```
    3. При локальной разработке:
        - Узнайте внешний IP Minikube:
            ```shell
            minikube ip
            ```
        пример ответа:
            ```
            192.168.59.100
            ```
        - Добавьте в файл `hosts` вашей ОС строку:

            ```
            192.168.59.100   your-domain.test
            ```
    4. Примените манифест Ingress::
        ```shell
        kubectl apply -f local_deployment/k8s_minikube/ingress-hosts.yaml
        ```
8. **Настроить очистку сессий (*CronJob*):**
    ```shell
    kubectl apply -f local_deployment/k8s_minikube/django-clearsessions.yaml
    ```   
9. **Проверить доступность сайта:**

    http://your-domain.test/

---
# ☸️ Запуск на облачном кластере

---
## Dev версия
Используйте манифесты из каталога `cloud_deployment/dev/`.

---
### 🔐 Секреты и конфигурация
#### 1. Сертификат PostgreSQL (Yandex Cloud Managed PostgreSQL)
**Файл:** `cloud_deployment/dev/postgres-ssl-cert.yaml`
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: postgres-ssl-cert
     namespace: <YOU-NAMESPACE>
   data:
     root.crt: |
      <значение POSTGRES_SSL_CERT в base64>
   ```
**Применение:**

```shell
  kubectl apply -f postgres-ssl-sert.yaml <YOU-NAMESPACE>
```
**Альтернатива:** 

```shell
  kubectl create secret generic postgres-ssl-cert \
    -n <YOUR-NAMESPACE> \
    --from-literal=root.crt='<POSTGRES_SSL_CERT в base64>'
```
#### 2. Django-секреты и конфиг
**Файл: `cloud_deployment/dev/django-secrets.yaml`**


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
  namespace: <YOUR-NAMESPACE>
type: Opaque
stringData:
  SECRET_KEY: "<SECRET_KEY>"
  DATABASE_URL: "postgres://<USER>:<PASS>@<HOST>:<PORT>/<DBNAME>"
```
**Файл: `cloud_deployment/dev/django-config.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
  namespace: <YOUR-NAMESPACE>
data:
  DJANGO_SETTINGS_MODULE: "webapp.settings"
  DEBUG: "<DEBUG>"
  ALLOWED_HOSTS: "127.0.0.1,localhost,<YOUR-DOMAIN>"
```
**Примените:**
```shell
  kubectl apply -f cloud_deployment/dev/django-secrets.yaml -n <YOUR-NAMESPACE>
  kubectl apply -f cloud_deployment/dev/django-config.yaml -n <YOUR-NAMESPACE>
```
---
### 🌐 Запуск Nginx:
```
kubectl apply -f nginx.yaml -n <YOU-NAMESPACE>
kubectl apply -f nginx-service.yaml -n <YOU-NAMESPACE>
```
---
### 🐳 Сборка и публикация Docker-образа

```shell
  git rev-parse --short HEAD
  docker build -f Dockerfile -t <DOCKER_USERNAME>/<IMAGE_NAME>:$(git rev-parse --short HEAD) .
  docker push <DOCKER_USERNAME>/<IMAGE_NAME>:$(git rev-parse --short HEAD)
```

### ⚙️ Запуск Django-приложения
Отредактируйте манифесты:
* `cloud_deployment/dev/django-services.yaml`
* `cloud_deployment/dev/django-deployment.yaml`

**Замените:**

```yaml
namespace: <YOUR-NAMESPACE>
image: dzimag/django-app-k8s:latest
```
**Примените:**

```shell
  kubectl apply -f django-services.yaml -n <YOUR-NAMESPACE>
  kubectl apply -f django-deployment.yaml -n <YOUR-NAMESPACE>
```
---
**Выполните миграции внутри pod’a**

Получите имя pod’a:
```shell
  kubectl get pods -n <YOUR-NAMESPACE>
```
Выполните миграции:
```shell
  kubectl exec -it <DJANGO-POD> -n <YOUR-NAMESPACE> -- python manage.py migrate
```
**Создайте суперпользователя**

```shell
  kubectl exec -it <DJANGO-POD> -n <YOUR-NAMESPACE> -- python manage.py createsuperuser
```
Подробнее см. [руководство по настройке](./getting-started.md)

[Тестовый вариант сайта развернут здесь](https://edu-dmitrij-gukalin.yc-sirius-dev.pelid.team/)

[Выделенные ресурсы облачной инфраструктуры](https://sirius-env-registry.website.yandexcloud.net/edu-dmitrij-gukalin.html)