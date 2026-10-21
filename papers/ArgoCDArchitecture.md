# Архитектура Argo CD

> Конспект на русском по статье [Argo CD Architecture: A Complete Guide](https://devopscube.com/argo-cd-architecture) (Aswin Vijayan, DevOpsCube). Это пересказ, а не дословный перевод — подробности в оригинале. Иллюстрации подгружаются с сайта DevOpsCube. Раздел «Шпаргалка» в конце — дополнение по документации Argo CD, а не часть статьи.

## Компоненты

Argo CD состоит из семи компонентов: API Server, Application Controller, ApplicationSet Controller, Repo Server, Dex (опционально), Redis и Notification Controller. Все они разворачиваются как Deployment, кроме Application Controller — это StatefulSet, что важно для масштабирования.

![Общая архитектура Argo CD](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2026/03/image--1-.webp)

| Компонент | Тип | Роль |
|---|---|---|
| API Server (`argocd-server`) | Deployment | REST/gRPC API для UI, CLI и CI/CD |
| Application Controller | StatefulSet | сверяет кластер с Git и синхронизирует |
| ApplicationSet Controller | Deployment | генерирует Application из шаблона |
| Repo Server | Deployment | клонирует Git и рендерит манифесты |
| Dex | Deployment | SSO через внешние провайдеры |
| Redis | Deployment | кэш |
| Notification Controller | Deployment | уведомления о событиях |

### API Server

Точка входа для UI, CLI и CI/CD-инструментов. Отвечает за:

- аутентификацию и проверку прав;
- хранение пользователей и паролей в Secret, настроек аккаунтов и прав — в ConfigMap;
- создание и изменение ресурсов Application, ApplicationSet и AppProject, которые хранятся в etcd как обычные объекты Kubernetes (это важно для бэкапов).

![Argo CD Server](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2026/01/architecture6.2.png)

### Application Controller

Ядро Argo CD — Kubernetes-контроллер, который:

- читает ресурсы Application;
- запрашивает у Repo Server отрендеренные манифесты;
- сравнивает желаемое состояние из Git с фактическим в кластере и показывает статус (`Synced` / `OutOfSync`, `Healthy` / `Degraded` и т.д.);
- применяет изменения при синхронизации — вручную или автоматически.

«Приложение» (Application) в Argo CD — любой развёрнутый набор манифестов: сервис, инструмент, инфраструктурный компонент.

![Application Controller](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2026/01/architecture5.2.png)

### ApplicationSet Controller

Из одного ресурса ApplicationSet генерирует набор Application, которые затем обрабатывает Application Controller. Типичные сценарии:

- одно приложение в нескольких окружениях или кластерах;
- все сервисы системы одним ресурсом (например, по каталогам в Git).

![ApplicationSet Controller](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2026/01/architecture4.2.png)

### Repo Server

- По запросу Application Controller клонирует репозиторий и рендерит манифесты: обычный YAML, Helm-чарты или Kustomize.
- Кэширует результат в Redis на 24 часа.
- Проверяет Git на изменения раз в 3 минуты. Чтобы реагировать сразу, настройте webhook из Git: тогда Git сам отправит POST-запрос при новом коммите.

![Repo Server](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2026/03/07.png)

### Dex

OIDC-провайдер для входа через корпоративные системы идентификации: AWS SSO, Okta, Google, Microsoft Entra ID (Azure AD), GitHub и другие. Хорошая практика — не давать всем полный доступ, а сочетать SSO-группы с RBAC. Например:

- DevOps-инженеры создают, удаляют и синхронизируют приложения;
- разработчики только смотрят приложения и логи в UI.

![Dex](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2026/01/architecture8.3.png)

### Redis

Кэш, который ускоряет ответы и снижает нагрузку на остальные компоненты и на Git. Хранит отрендеренные манифесты, данные о созданных ресурсах, статусы приложений и синхронизации. Учётные данные и конфигурацию не хранит — они в Secret и ConfigMap, поэтому потеря Redis не означает потерю данных.

![Redis в Argo CD](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2026/01/architecture9.2.png)

### Notification Controller

Следит за событиями Application и отправляет уведомления по шаблонам: смена статуса синхронизации или здоровья, выкладка новой версии и т.п. Каналы: Slack, e-mail, Microsoft Teams, PagerDuty, AWS SQS и другие.

![Notification Controller](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2026/01/architecture10.3.png)

## Хранение и бэкапы

Argo CD не использует Persistent Volume, а Redis — только кэш. Всё состояние — обычные объекты Kubernetes:

| Где | Что |
|---|---|
| CRD (Application, ApplicationSet, AppProject) | приложения и проекты |
| ConfigMap в `argocd` | настройки сервера и UI, RBAC (изменения через UI/CLI тоже попадают сюда) |
| Secret в `argocd` | пароли, доступы к кластерам и репозиториям |

![Хранение данных Argo CD](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2025/11/1756465118036.jpeg)

Восстанавливать это вручную долго, поэтому бэкапьте namespace Argo CD вместе с CRD:

- `argocd admin export` — встроенный способ для быстрых бэкапов;
- **Velero** — рекомендуется для продакшена: полный бэкап namespace (Secret, ConfigMap, CRD) по расписанию.

## Высокая доступность и масштабирование

Одна реплика каждого компонента — единая точка отказа. Для продакшена есть отдельный HA-манифест.

| Компонент | Что делать в HA |
|---|---|
| API Server, Repo Server | минимум 2 реплики. Из-за pod anti-affinity нужны минимум 3 ноды — на одной ноде HA не заработает |
| Redis | HA-режим с тремя Sentinel: три нужны для кворума при выборе лидера |
| Application Controller | по умолчанию 1 реплика даже в HA. При большом числе кластеров увеличить реплики и включить шардирование |

Шардирование Application Controller:

- кластеры делятся между репликами, например 30 кластеров на 3 репликах — по 10 на каждую;
- число шардов должно совпадать с числом реплик;
- по умолчанию кластеры распределяются по хэшу, можно выбрать round-robin или другой алгоритм.

## Безопасность

- **Админ.** После установки есть только `admin` со случайным паролем — смените его после первого входа. Пароль хранится в `argocd-secret` как bcrypt-хэш (`$2a$10$…`).
- **Учётные данные** (включая доступ к Git, который использует Repo Server) хранятся в Kubernetes Secret.
- **TLS.** Компоненты (`argocd-server`, `repo-server`, `application-controller`) общаются между собой по TLS. Трафик к Redis по умолчанию не шифруется, но TLS можно включить.
- **RBAC.** Встроены роли `readonly` и `admin`; можно создавать свои с нужными действиями (просмотр, деплой, изменение, удаление).
- **Права самого Argo CD.** Сузьте ClusterRole для `argocd-server` и `application-controller` до нужных namespace, чтобы Argo CD не видел чужие ресурсы.

## Мониторинг

Argo CD показывает статусы приложений, но мониторинг самого Argo CD строится на Prometheus и Grafana. Метрики по умолчанию выключены и включаются для каждого компонента; все отдаются на `/metrics`:

| Компонент | Порт |
|---|---|
| Application Controller | 8082 |
| API Server | 8083 |
| Repo Server | 8084 |
| ApplicationSet Controller | 8080 |
| Notification Controller | 9001 |
| Dex | 5558 |

Ключевые метрики: здоровье и статус синхронизации приложений, запросы к Git, число управляемых кластеров. Используйте готовые дашборды Grafana от сообщества и алерты на долю неудачных синхронизаций и глубину очереди контроллера.

## Шпаргалка

### Установка и первый вход

```bash
kubectl create namespace argocd
# обычная установка
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# или HA-вариант
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml

# начальный пароль admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

argocd login <адрес-argocd-server>
argocd account update-password
```

### Повседневные команды CLI

```bash
argocd app list
argocd app get <app>            # статус синхронизации и здоровья
argocd app diff <app>           # отличие Git от кластера
argocd app sync <app>
argocd app history <app>
argocd app rollback <app> <id>
argocd cluster list
argocd repo list
```

### Минимальный Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/deploy.git
    targetRevision: main
    path: apps/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true      # удалять ресурсы, которых больше нет в Git
      selfHeal: true   # откатывать ручные изменения в кластере
    syncOptions:
      - CreateNamespace=true
```

### RBAC (`argocd-rbac-cm`)

```yaml
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:deployer, applications, get,  */*, allow
    p, role:deployer, applications, sync, */*, allow
    g, my-org:devops, role:admin
    g, my-org:developers, role:deployer
```

### Где что настраивается

| ConfigMap / Secret | Что там |
|---|---|
| `argocd-cm` | общие настройки, SSO/Dex, интервал опроса Git (`timeout.reconciliation`) |
| `argocd-rbac-cm` | политики RBAC |
| `argocd-cmd-params-cm` | параметры компонентов (например, алгоритм шардирования) |
| `argocd-notifications-cm` | шаблоны и триггеры уведомлений |
| `argocd-secret` | пароль admin, ключи webhook |

### Бэкап и восстановление

```bash
# встроенный экспорт
argocd admin export -n argocd > argocd-backup.yaml
argocd admin import -n argocd - < argocd-backup.yaml

# Velero: namespace + кластерные ресурсы (CRD)
velero backup create argocd-backup --include-namespaces argocd --include-cluster-resources=true
```

### Чек-лист продакшена

- [ ] HA-манифест, минимум 3 ноды.
- [ ] Шардирование Application Controller при большом числе кластеров.
- [ ] Пароль admin сменён; вход через SSO; права через RBAC.
- [ ] Webhook из Git вместо одного только опроса.
- [ ] Регулярные бэкапы namespace и CRD (Velero).
- [ ] Метрики включены, дашборды и алерты настроены.
- [ ] Уведомления о неудачных синхронизациях и деградации.
