# Django Site

Докеризированный сайт на Django для развертки в Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет
сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и
Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и
Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

---
# Переменные окружения

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

---

### Локальный запуск с использованием кластера Minikube

Для запуска необходимы:

- [См. документацию Docker Desktop](https://www.docker.com/get-started/)

- [См. документацию сubectl](https://kubernetes.io/docs/tasks/tools/)

- [См. документацию minikube](https://kubernetes.io/docs/tasks/tools/)


1. **Запустить кластер Minikube:**

    ```sh
    minikube start --driver=virtualbox
    ```
    * *Вместо docker можно использовать другой драйвер (например, docker), если он у вас настроен:*

        ```sh
        minikube start --driver=docker
        ```
2. **Передача чувствительных данных:**

   Создайте файл `local_deployment/secrets.yaml`, заменив значения переменных на свои (*см. раздел переменные окружения*):

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
    ```sh
    kubectl apply -f kubernetes/secrets.yaml
    ```
3. **Запуск базы данных PostgreSQL (*в Minikube*):**
    1. Установите Helm [см. документацию Helm](https://helm.sh/)
    2. Создайте файл `local_deployment/postgres-values.yaml` и укажите свои данные доступа::
        ```
        architecture: "standalone"

        auth:
            username: "username"
            password: "password"
            database: "nameDB"
        ```
    3. Установите PostgreSQL через Helm:
        ```sh
        helm install my-postgres oci://registry-1.docker.io/bitnamicharts/postgresql -f local_deployment/postgres-values.yaml  
        ```
4. **Запуск Django-приложения и сервиса:**
    ```sh
    kubectl apply -f local_deployment/django-deployment.yaml
    kubectl apply -f local_deployment/django-service.yaml
    ```
5. **Выполнить миграции:**
    ```sh    
    kubectl apply -f local_deployment/django-migrate.yaml
    ```
6. **Создать суперпользователя:**
    ```sh    
    kubectl apply -f local_deployment/django-superuser.yaml
    ```
7. **(*Опционально)* Настроить Ingress:**
    1. Установить ingress Controllers [см. таблицу доступных контролееров](https://docs.google.com/spreadsheets/d/191WWNpjJ2za6-nbG4ZoUMXMpUK8KlCIosvQB0f-oq3k/edit?gid=907731238#gid=907731238), например [Сontour](https://projectcontour.io/getting-started/):
        ```sh    
        kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
        ```
    2. Отредактируйте файл `local_deployment/ingress-hosts.yaml`, укажите свой домен:
        ```
        ...
        spec:
          rules:
            - host: your-domain.test
        ...
        ```
    3. При локальной разработке:
        - Узнайте внешний IP Minikube:
            ```sh
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
        ```sh
        kubectl apply -f local_deployment/ingress-hosts.yaml
        ```
8. **Настроить очистку сессий (*CronJob*):**
    ```sh
    kubectl apply -f local_deployment/django-clearsessions.yaml
    ```   
9. **Проверить доступность сайта:**

    http://your-domain.test/
    
