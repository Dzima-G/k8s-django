# 🚀 Getting Started — запуск Django-приложения в облачном кластере

Это пошаговое руководство поможет запустить приложение в Yandex Cloud Managed Kubernetes.
Предполагается, что у вас уже есть:

- настроенный namespace в кластере;
- доступ к kubectl;
- Docker Hub-аккаунт для хранения образов;
- настроенный Ingress.

---

## 🧱 1. Dev-версия

Для запуска используйте готовые манифесты из каталога:

```bash
  cloud_deployment/dev/
```

---

## 🔐 2. Подготовка секретов и конфигураций

### 2.1 Сертификат PostgreSQL (Yandex Cloud Managed PostgreSQL)

Создайте файл `cloud_deployment/dev/postgres-ssl-cert.yaml` и вставьте значение `POSTGRES_SSL_CERT` в формате base64:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-ssl-cert
  namespace: <YOUR-NAMESPACE>
data:
  root.crt: |
    <значение POSTGRES_SSL_CERT в base64>
```

**Примените:**

```bash
  kubectl apply -f cloud_deployment/dev/postgres-ssl-cert.yaml -n <YOUR-NAMESPACE>
```

> 💡 Альтернатива — создание секрета без файла:
> ```bash
> kubectl create secret generic postgres-ssl-cert >   -n <YOUR-NAMESPACE> >   --from-literal=root.crt='<POSTGRES_SSL_CERT в base64>'
> ```

---

### 2.2 Django-секреты и конфиг

Создайте `cloud_deployment/dev/django-secrets.yaml`:

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

Создайте `cloud_deployment/dev/django-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
  namespace: <YOUR-NAMESPACE>
data:
  DJANGO_SETTINGS_MODULE: "webapp.settings"
  DEBUG: "True"
  ALLOWED_HOSTS: "127.0.0.1,localhost,<YOUR-DOMAIN>"
```

**Примените оба файла:**

```bash
  kubectl apply -f cloud_deployment/dev/django-secrets.yaml -n <YOUR-NAMESPACE>
  kubectl apply -f cloud_deployment/dev/django-config.yaml -n <YOUR-NAMESPACE>
```

---

## 🌐 3. Запуск Nginx

Nginx используется как обратный прокси для Django-приложения.

```bash
  kubectl apply -f nginx.yaml -n <YOUR-NAMESPACE>
  kubectl apply -f nginx-service.yaml -n <YOUR-NAMESPACE>
```

Проверить, что сервис создан:

```bash
  kubectl get svc -n <YOUR-NAMESPACE>
```

---

## 🐳 4. Сборка и публикация Docker-образа

1. Получите короткий хэш текущего коммита:
   ```bash
   git rev-parse --short HEAD
   ```

2. Соберите образ:
   ```bash
   docker build -f Dockerfile -t <DOCKER_USERNAME>/<IMAGE_NAME>:$(git rev-parse --short HEAD) .
   ```

3. Отправьте образ в Docker Hub:
   ```bash
   docker push <DOCKER_USERNAME>/<IMAGE_NAME>:$(git rev-parse --short HEAD)
   ```

> **💡 Альтернатива:**
>
> Вместо автоматического тега по хэшу коммита можно указать любой удобный тег вручную, например:
> ```bash
> docker build -f Dockerfile -t <DOCKER_USERNAME>/<IMAGE_NAME>:latest .
> docker push <DOCKER_USERNAME>/<IMAGE_NAME>:latest
> ```
> Тег latest удобно использовать для быстрых тестовых сборок, но для стабильных релизов рекомендуется указывать
> конкретные версии (v1.0, dev, test, prod) или хэш коммита — чтобы точно знать, какая версия кода развёрнута в кластере.

---

## ⚙️ 5. Запуск Django-приложения

Отредактируйте:

- `cloud_deployment/dev/django-services.yaml`
- `cloud_deployment/dev/django-deployment.yaml`

Замените:

```yaml
namespace: <YOUR-NAMESPACE>
image: <DOCKER_USERNAME>/<IMAGE_NAME>:<TAG>
```

Пример:

```yaml
image: dzimag/django-app-k8s:latest
```

**Примените манифесты:**

```bash
  kubectl apply -f django-services.yaml -n <YOUR-NAMESPACE>
  kubectl apply -f django-deployment.yaml -n <YOUR-NAMESPACE>
```

---

## ⚙️ 6. Применение миграций Django

После запуска контейнера необходимо применить миграции, чтобы база данных соответствовала моделям.

1. **Получи имя pod’a с Django**
    ```bash
    kubectl get pods -n <YOUR-NAMESPACE>
    ```
    Найди pod, где запущено приложение (обычно называется django-...).

2. **Выполните миграции внутри pod**

    ```bash
    kubectl exec -it <DJANGO-POD> -n <YOUR-NAMESPACE> -- python manage.py migrate
    ```
---

## ⚙️ 7. Создание суперпользователя (админа) в Django-приложении
1. **Получи имя pod’a с Django**
    ```bash
    kubectl get pods -n <YOUR-NAMESPACE>
    ```
    Найди pod, где запущено приложение (обычно называется django-...).

2. **Выполни команду внутри pod’a**

    Запусти интерактивную команду для создания суперпользователя:
    ```bash
    kubectl exec -it <DJANGO-POD> -n <YOUR-NAMESPACE> -- python manage.py createsuperuser
    ```
3. **Введи данные пользователя**
    ```
    Username: admin
    Email address: admin@example.com
    Password: ********    
    ```
4. **Проверка**

    После успешного создания — открой /admin на домене или IP проекта:
    
    ```
    http://<EXTERNAL-IP>/admin
    ```
    и войди под этими учётными данными.


---

## 🧩 8. Проверка и диагностика

### Проверить деплой:

```bash
  kubectl get deployments -n <YOUR-NAMESPACE>
  kubectl get pods -n <YOUR-NAMESPACE>
  kubectl get svc -n <YOUR-NAMESPACE>
```

Все поды должны иметь статус `Running`.

### Проверить версию образа:

```bash
  kubectl describe pod <DJANGO-POD> -n <YOUR-NAMESPACE> | grep Image
```

### Проверить доступность приложения:

```bash
  curl http://<EXTERNAL-IP>
```

---

## ⚠️ Где искать трейсбеки

Просмотреть логи приложения:

```bash
  kubectl logs <DJANGO-POD> -n <YOUR-NAMESPACE>
```

Если используется Gunicorn:

```bash
kubectl logs <DJANGO-POD> -n <YOUR-NAMESPACE> -c django
```

Для Nginx:

```bash
kubectl logs <NGINX-POD> -n <YOUR-NAMESPACE>
```

---

После выполнения шагов приложение будет развернуто в вашем namespace и доступно по IP или domain из сервиса Nginx.

---