# Лабораторная работа: Запуск микросервисного приложения в Kubernetes

## Цель

Развернуть текущий проект мессенджера в Kubernetes-кластере, настроить хранение файлов через S3 CSI, организовать GitOps-деплой через Argo CD и подготовить `kustomize`-конфигурации для `dev` и `prod`.

# Лабораторная работа: Запуск микросервисного приложения в Kubernetes

## Описание проекта

Микросервисный мессенджер состоит из:
- `frontend` – веб-интерфейс (React + Nginx)
- `bff` – API-шлюз (Backend For Frontend)
- `user-service` – сервис управления пользователями
- `message-service` – сервис сообщений и загрузки файлов
- `postgres` – база данных (две БД: `messager_users`, `messager_messages`)
- `minio` – S3-совместимое хранилище для файлов (подключено через CSI)

---

## Структура репозитория

````
k8s/
├── base/ # Базовые манифесты
│ ├── namespace.yaml
│ ├── configmap.yaml
│ ├── secret.yaml
│ ├── postgres-pvc.yaml
│ ├── postgres.yaml
│ ├── migrate-users-job.yaml
│ ├── migrate-messages-job.yaml
│ ├── user-service.yaml
│ ├── message-service.yaml
│ ├── bff.yaml
│ ├── frontend.yaml
│ ├── minio.yaml
│ ├── s3-secret.yaml
│ ├── s3-pv-pvc.yaml
│ └── kustomization.yaml
├── overlays/
│ ├── dev/ # Конфигурация для разработки
│ │ ├── kustomization.yaml
│ │ └── patches/
│ │ ├── replicas.yaml
│ │ └── resources.yaml
│ └── prod/ # Конфигурация для production
│ ├── kustomization.yaml
│ └── patches/
│ ├── replicas.yaml
│ └── resources.yaml
argocd/
└── application-dev.yaml # Argo CD Application для dev
docs/ # Документация и скриншоты
````

# Messager — микросервисное приложение для обмена сообщениями

## Описание компонентов

| Компонент        | Назначение                             | Порт | Реплики (dev/prod) | Образ                           |
|----------------|----------------------------------------|------|--------------------|---------------------------------|
| frontend       | Веб-интерфейс (SPA)                    | 80   | 1 / 2              | mablinov2704/frontend           |
| bff            | API Gateway для фронтенда              | 8080 | 1 / 2              | mablinov2704/bff                |
| user-service   | Управление пользователями              | 8081 | 1 / 2              | mablinov2704/user-service       |
| message-service| Сообщения и файлы                      | 8082 | 1 / 2              | mablinov2704/message-service    |
| postgres       | PostgreSQL (единый инстанс)            | 5432 | 1                  | postgres:16-alpine              |
| minio          | S3-совместимое хранилище               | 9000 | 1                  | minio/minio:latest              |

## Сетевые связи

- **Внешний доступ**: User → (Ingress/port-forward) → frontend:80
- **Внутренние API**:
  - frontend → bff:8080 (BFF_INTERNAL_URL)
  - bff → user-service:8081
  - bff → message-service:8082
- **Базы данных**:
  - user-service → postgres:5432/messager_users
  - message-service → postgres:5432/messager_messages
- **Хранилище файлов**:
  - message-service → /app/uploads (S3 CSI) → minio:9000/messager-uploads

## Конфигурация окружения

### ConfigMap `messager-config`

| Ключ                          | Значение                                |
|-------------------------------|-----------------------------------------|
| FRONTEND_BFF_URL              | "" (same-origin)                        |
| FRONTEND_BFF_INTERNAL_URL     | http://bff:8080                         |
| BFF_HTTP_PORT                 | 8080                                    |
| USER_SERVICE_URL              | http://user-service:8081                |
| MSG_SERVICE_URL               | http://message-service:8082             |
| USER_HTTP_PORT                | 8081                                    |
| MSG_HTTP_PORT                 | 8082                                    |
| MSG_UPLOADS_DIR               | /app/uploads                            |
| POSTGRES_DB                   | messager                                |
| POSTGRES_HOST                 | postgres                                |
| POSTGRES_PORT                 | 5432                                    |

### Secret `messager-db-secret`

| Ключ                | Значение                                                              |
|---------------------|-----------------------------------------------------------------------|
| POSTGRES_USER       | postgres                                                              |
| POSTGRES_PASSWORD   | postgres123                                                           |
| USER_DB_DSN         | host=postgres user=postgres password=postgres123 dbname=messager_users sslmode=disable |
| MSG_DB_DSN          | host=postgres user=postgres password=postgres123 dbname=messager_messages sslmode=disable |

## Node Affinity

| Под               | Требование                                      | Тип               | Метка узла                 |
|-------------------|-------------------------------------------------|-------------------|----------------------------|
| postgres          | только workload=system                          | required          | workload=system            |
| minio             | только workload=system                          | required          | workload=system            |
| frontend          | только workload=app                             | required          | workload=app               |
| bff               | только workload=app                             | required          | workload=app               |
| user-service      | только workload=app                             | required          | workload=app               |
| message-service   | workload=app + предпочтение disk=fast           | required + preferred | workload=app, disk=fast  |

## S3 CSI

- **Драйвер**: ch.ctrox.csi.s3-driver
- **Бакет**: messager-uploads
- **Endpoint**: http://minio.messager:9000
- **PVC**: `message-uploads-pvc` (Bound, 10Gi, RWX)
- **Монтирование**: `/app/uploads` в message-service

## Kustomize: отличия dev/prod

| Параметр              | dev                  | prod                     |
|-----------------------|----------------------|--------------------------|
| Реплики               | 1                    | 2 (frontend, bff, user-service, message-service) |
| CPU requests          | 70–150m              | 150–250m                 |
| CPU limits            | 150–300m             | 300–500m                 |
| Memory requests       | 100–192Mi            | 128–256Mi                |
| Memory limits         | 150–256Mi            | 256–512Mi                |
| Теги образов          | latest               | v1.0.0                   |

## GitOps с Argo CD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: messager-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/DanLincoln24/IPR_Laba1.git
    targetRevision: HEAD
    path: k8s/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: messager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
````

## Проверки

- `kubectl kustomize k8s/overlays/dev` – без ошибок
- `kubectl kustomize k8s/overlays/prod` – без ошибок
- Все поды в `Running` (кроме миграций)
- Фронтенд доступен через port-forward
- S3 PVC в статусе `Bound`
- Argo CD Application создан

## Скриншоты

<img width="802" height="288" alt="image" src="https://github.com/user-attachments/assets/426bd757-f676-4457-bc32-4dd31c511c02" />
– `kubectl get pods -n messager`
<img width="1860" height="1038" alt="image" src="https://github.com/user-attachments/assets/6ea94b50-d66e-4678-98ec-7a9bab6a1f43" />
– браузер с `http://localhost:8080`
<img width="483" height="418" alt="image" src="https://github.com/user-attachments/assets/4786069f-4743-476c-bc2e-c61e272ed156" />
– Argo CD UI (Synced)

## Репозиторий

[https://github.com/DanLincoln24/IPR_Laba1](https://github.com/DanLincoln24/IPR_Laba1)
