# KEDA: масштабирование по событиям, а не по CPU

> Конспект на русском по статье [Kubernetes KEDA Autoscaling: Scale Smarter, Not Harder](https://the-devops-engineer.medium.com/kubernetes-keda-autoscaling-scale-smarter-not-harder-d186da29175a) (The DevOps Engineer, Medium). Это пересказ, а не перевод: пошаговые манифесты и команды — в оригинале. Иллюстрация подгружается с Medium. Раздел «Шпаргалка» в конце — дополнение по документации KEDA, а не часть статьи.

## Проблема

HPA обычно масштабирует по CPU и памяти. Но у современных приложений узкое место чаще в другом: глубина очереди (Kafka, RabbitMQ, SQS), насыщение пула соединений с БД, промахи кэша, очередь HTTP-запросов, задачи по расписанию. Когда в Kafka скопились сотни тысяч сообщений, а CPU на 12%, HPA ничего не делает.

## Что такое KEDA

KEDA (Kubernetes Event-Driven Autoscaling) — лёгкий open-source компонент, выпускной (graduated) проект CNCF, изначально созданный Microsoft и Red Hat. Он выполняет две роли:

- **Metrics Adapter** — отдаёт в Kubernetes Metrics API внешние метрики из 60+ источников (Kafka, RabbitMQ, Redis, Prometheus, SQS и др.);
- **оператор** — следит за ресурсами `ScaledObject` и управляет HPA. Кроме того, KEDA сам масштабирует до нуля и обратно, чего обычный HPA не умеет.

![KEDA](https://cdn-images-1.medium.com/max/1024/1*CksPuI3t6WlwK3LLESth7g.jpeg)

| Возможность | HPA | KEDA |
|---|---|---|
| CPU / память | да | да |
| Кастомные метрики | сложно настроить | из коробки |
| Внешние события | нет | 60+ скейлеров |
| Масштабирование до нуля и с нуля | нет | да |
| Kubernetes Job | нет | да |

## Пример: распродажа

Сервис обработки заказов читает очередь RabbitMQ. Обычно в очереди около 20 сообщений, и хватает 2 подов. На распродаже за минуту приходит 80 000 заказов.

- **HPA по CPU (порог 70%)** реагирует с опозданием: CPU — запаздывающий индикатор. Пока он отразит нагрузку, проходит несколько минут, очередь растёт, клиенты ждут.
- **KEDA с целью «500 сообщений на под»** реагирует на причину (длину очереди), а не на симптом (CPU), и сразу поднимает нужное число подов. После разбора очереди поды уходят вплоть до нуля.

## Ресурсы KEDA

1. **ScaledObject** — масштабирование Deployment и StatefulSet: что масштабировать, по какому источнику (trigger), минимум и максимум реплик, порог.
2. **ScaledJob** — вместо масштабирования долгоживущих реплик создаёт новые Job на пачки событий. Подходит для батч-обработки.
3. **TriggerAuthentication / ClusterTriggerAuthentication** — как KEDA аутентифицируется во внешних системах (секреты, строки подключения, IAM-роли), чтобы не держать учётные данные в ScaledObject.

KEDA не заменяет HPA, а снабжает его более точными данными: планировщик, PodDisruptionBudget и логика выкатки продолжают работать как обычно.

## Практика: RabbitMQ

Шаги из статьи:

1. Установить KEDA через Helm (появятся три пода KEDA).
2. Развернуть RabbitMQ.
3. Сохранить строку подключения к RabbitMQ в Secret.
4. Развернуть consumer-приложение.
5. Создать `TriggerAuthentication`, ссылающийся на Secret.
6. Создать `ScaledObject` с триггером `rabbitmq` на нужную очередь.
7. Проверить: KEDA создаёт HPA для Deployment, а в статусе ScaledObject должно быть `Active: True` и `Ready: True`.
8. Отправить в очередь пачку сообщений (через UI RabbitMQ или скрипт) и наблюдать, как поды поднимаются за секунды и опускаются до нуля после разбора очереди.

Набросок ScaledObject (упрощённо, без привязки к примеру из статьи):

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker
spec:
  scaleTargetRef:
    name: worker            # Deployment
  minReplicaCount: 0
  maxReplicaCount: 30
  cooldownPeriod: 60
  triggers:
    - type: rabbitmq
      metadata:
        queueName: tasks
        mode: QueueLength
        value: "100"        # целевое число сообщений на под
      authenticationRef:
        name: rabbitmq-auth # TriggerAuthentication
```

## Несколько триггеров

В одном ScaledObject можно указать несколько триггеров (например, длину очереди и число HTTP-запросов). KEDA выбирает тот, который требует больше реплик.

## Советы для продакшена

1. Всегда задавайте `resources.requests` и `limits`, иначе масштабирование будет непредсказуемым.
2. Подбирайте `cooldownPeriod` (по умолчанию 300 с): для всплесков, которые быстро повторяются, — 30–60 с, для подов с долгим стартом — больше.
3. Общие учётные данные для нескольких namespace выносите в `ClusterTriggerAuthentication`.
4. `minReplicaCount: 0` экономит ресурсы, но добавляет холодный старт. Для чувствительных к задержке сервисов держите хотя бы один под.
5. Мониторьте метрики KEDA в Prometheus: `keda_scaler_metrics_value`, `keda_scaler_active`, `keda_scaled_object_errors`.

## Какие бывают скейлеры

- **Очереди**: Kafka, RabbitMQ, AWS SQS, Azure Service Bus, GCP Pub/Sub.
- **Базы данных**: Redis, MongoDB, PostgreSQL.
- **Наблюдаемость**: Prometheus, Datadog, Graphite, New Relic.
- **Облака**: CloudWatch, Azure Monitor, Stackdriver.
- **HTTP**: отдельный HTTP Add-on.
- **Cron**: например, поднять поды к 8:00 и опустить к 18:00. Полезно для заблаговременного прогрева перед известным пиком.

## Выводы

- HPA реагирует на симптомы, KEDA — на причины.
- Масштабирование до нуля напрямую снижает расходы на событийные нагрузки.
- 60+ скейлеров и комбинирование триггеров подходят почти для любой архитектуры.
- Порог входа низкий: описываете ScaledObject, остальное делает KEDA.

## Шпаргалка

### Установка

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace

kubectl get pods -n keda
# keda-operator, keda-operator-metrics-apiserver, keda-admission-webhooks
```

### Поля ScaledObject

| Поле | По умолчанию | Смысл |
|---|---|---|
| `scaleTargetRef.name` | — | что масштабировать (Deployment, StatefulSet или ресурс с `/scale`) |
| `minReplicaCount` | 0 | минимум реплик; 0 — масштабирование до нуля |
| `maxReplicaCount` | 100 | максимум реплик |
| `pollingInterval` | 30 с | как часто опрашивать источник |
| `cooldownPeriod` | 300 с | сколько ждать после последней активности перед уходом в 0 |
| `idleReplicaCount` | — | число реплик в простое (сейчас допускается только 0) |
| `fallback` | — | сколько реплик держать, если источник метрик недоступен |
| `advanced.horizontalPodAutoscalerConfig.behavior` | — | тонкая настройка скорости масштабирования HPA |
| `triggers` | — | источники событий; при нескольких побеждает требующий больше реплик |

### TriggerAuthentication из Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-conn
stringData:
  host: amqp://user:password@rabbitmq.default.svc:5672/
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-auth
spec:
  secretTargetRef:
    - parameter: host        # имя параметра, которое ждёт скейлер
      name: rabbitmq-conn    # Secret
      key: host              # ключ в Secret
```

### ScaledJob: по Job на пачку сообщений

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-resizer
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: worker
            image: example/resizer:1.0
        restartPolicy: Never
  pollingInterval: 15
  maxReplicaCount: 20
  triggers:
    - type: rabbitmq
      metadata:
        queueName: images
        mode: QueueLength
        value: "10"
      authenticationRef:
        name: rabbitmq-auth
```

### Другие частые триггеры

```yaml
# по расписанию: 5 реплик в рабочие часы
- type: cron
  metadata:
    timezone: Europe/Moscow
    start: 0 8 * * 1-5
    end: 0 19 * * 1-5
    desiredReplicas: "5"

# по запросу Prometheus
- type: prometheus
  metadata:
    serverAddress: http://prometheus.monitoring.svc:9090
    query: sum(rate(http_requests_total{app="api"}[1m]))
    threshold: "100"     # целевое значение на одну реплику

# по лагу консьюмер-группы Kafka
- type: kafka
  metadata:
    bootstrapServers: kafka.kafka.svc:9092
    consumerGroup: orders
    topic: orders
    lagThreshold: "500"
```

### Диагностика

```bash
kubectl get scaledobject,scaledjob -A
kubectl describe scaledobject <имя>     # conditions Ready / Active / Fallback
kubectl get hpa                         # KEDA создаёт HPA с именем keda-hpa-<имя>
kubectl logs -n keda deploy/keda-operator
```

### HPA или KEDA

| Ситуация | Что взять |
|---|---|
| нагрузка действительно зависит от CPU или памяти | HPA |
| обработка очередей и событий | KEDA |
| нужно масштабирование до нуля | KEDA |
| пакетные задачи на событие | KEDA `ScaledJob` |
| плановый прогрев перед пиком | KEDA `cron` |

Не создавайте собственный HPA для того же Deployment, которым управляет ScaledObject: два автоскейлера будут конфликтовать.
