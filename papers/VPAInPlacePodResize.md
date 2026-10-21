# In-Place Pod Resize и VPA в Kubernetes

> Конспект на русском по статье [How to Use In-Place Pod Resize with VPA in Kubernetes](https://devopscube.com/vpa-in-place-pod-resize) (Aswin Vijayan, DevOpsCube). Это пересказ, а не перевод — полные манифесты в оригинале. Схема подгружается с сайта DevOpsCube. Раздел «Шпаргалка» в конце — дополнение по документации Kubernetes и VPA, а не часть статьи.

## Что такое In-Place Pod Resize

Функция стабильна с Kubernetes 1.35.

Раньше изменить CPU или память у работающего пода без перезапуска было нельзя. In-Place Pod Resize позволяет менять requests и limits CPU и памяти у работающего пода, не удаляя и не пересоздавая его. Kubelet напрямую обновляет cgroup контейнера в Linux; контейнер не перезапускается, если перезапуск явно не запрошен.

Изменение делается через подресурс `resize` — он сообщает Kubernetes, что меняются только изменяемые на лету поля (CPU и память), и процесс завершать не нужно:

```bash
kubectl patch pod my-pod --subresource resize --patch \
  '{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"300m"}}}]}}'
```

Ограничение: так работает только для отдельного пода. Если изменить ресурсы в Deployment, Kubernetes пересоздаст поды. Поэтому для рабочих нагрузок изменения на месте делают через VPA.

## Как работает связка с VPA

![VPA In-Place Pod Resize](https://storage.ghost.io/c/5f/2f/5f2f4d20-2abf-4534-8d40-7aa233aedd43/content/images/2026/04/image-44.png)

1. **Metrics Server** постоянно собирает потребление CPU и памяти подами.
2. **VPA Recommender** анализирует метрики и, если ресурсов слишком мало или слишком много, рассчитывает новые значения.
3. **VPA Updater** не выселяет под, а отправляет в API Kubernetes патч через подресурс `/resize`.
4. **Kubelet** на ноде получает изменение и обновляет cgroup контейнера.
5. Под получает новые ресурсы без перезапуска.

## Зачем это нужно

Обычный VPA применяет рекомендацию так: выселяет под, поднимает новый с обновлёнными ресурсами и удаляет старый. Для многих нагрузок это приемлемо. Но для stateful-приложений, CMS вроде WordPress, долгих задач и всего, что плохо переносит рестарт, выселение создаёт проблемы.

Режим VPA **`InPlaceOrRecreate`** использует In-Place Pod Resize и меняет ресурсы подов целевого Deployment без перезапуска.

Если на ноде не хватает CPU или памяти для изменения, VPA выселяет под и пытается разместить его на ноде с достаточными ресурсами. Если такой нет, под остаётся в Pending.

## resizePolicy

Иногда перезапуск после изменения ресурсов всё же нужен. Пример — JVM: максимальный размер heap вычисляется один раз при старте исходя из лимита памяти. Если поднять лимит cgroup без рестарта, JVM лишнюю память не использует.

Для этого у каждого контейнера можно задать `resizePolicy` — что делать kubelet'у при изменении конкретного ресурса. Перезапускается только контейнер, не под.

| Значение | Поведение |
|---|---|
| `NotRequired` (по умолчанию) | обновить cgroup, контейнер продолжает работать |
| `RestartContainer` | перезапустить контейнер после изменения |

Значения можно комбинировать по ресурсам. Пример ниже: при изменении памяти контейнер перезапускается, при изменении CPU — нет.

```yaml
resizePolicy:
  - resourceName: cpu
    restartPolicy: NotRequired
  - resourceName: memory
    restartPolicy: RestartContainer
```

## Требования

- Кластер Kubernetes 1.35 и `kubectl`.
- Ноды на **cgroup v2**. Проверка на ноде: `stat -fc %T /sys/fs/cgroup/` должна вернуть `cgroup2fs`. Если вывод `tmpfs`, это cgroup v1, и изменение на месте не работает.

## Практика по шагам

1. **Metrics Server.** VPA берёт из него данные о потреблении. Установите, если его ещё нет, и проверьте, что под запущен.
2. **VPA.** Установите с включённой поддержкой in-place. Должны работать поды `vpa-recommender`, `vpa-updater` и `vpa-admission-controller`.
3. **Deployment с `resizePolicy`.** Создайте тестовый nginx и убедитесь, что под запущен, а ресурсы соответствуют заданным.

   **Важно:** если не задать начальные requests и limits, под получит QoS-класс BestEffort. Когда VPA добавит requests, класс должен смениться на Burstable, но QoS работающего пода изменить нельзя — под будет удалён и создан заново.

4. **Объект VPA** для этого Deployment в режиме `InPlaceOrRecreate` с минимальными значениями (в статье — 100m CPU и 200Mi памяти). Через пару минут Recommender соберёт метрики и обновит ресурсы. В демо автора CPU вырос с 50m до 100m, память — с 64Mi до 250Mi: VPA выставил больше минимума, потому что приложению это требовалось.
5. **Проверка.** Requests и limits пода обновились по рекомендации VPA, а сам под не выселялся.

## Уменьшение ресурсов

- **CPU** — сжимаемый ресурс: при снижении лимита ядро просто ограничивает процесс, не убивая его.
- **Память** — несжимаемый ресурс: если контейнер уже использует больше нового лимита, он упадёт с OOMKilled.

Для уменьшения памяти используйте `resizePolicy` с перезапуском.

## Лучшие практики

- В продакшене начинайте с режима `Off`. В других режимах ресурсы будут меняться вслед за нагрузкой, вплоть до выселения подов.
- Всегда задавайте `minAllowed` и `maxAllowed`: без `maxAllowed` VPA может порекомендовать значения, которые исчерпают ресурсы ноды.
- Для JVM-приложений ставьте `RestartContainer` для памяти, иначе потолок heap не изменится.

## Очистка

Удалите тестовый Deployment и объект VPA, а затем удалите сам VPA скриптом из клонированного репозитория autoscaler.

## Итог

In-Place Pod Resize в связке с VPA даёт подам нужные ресурсы без выселения. В режиме `InPlaceOrRecreate` VPA патчит работающий под напрямую: UID пода остаётся тем же, контейнер продолжает работать, меняются только значения ресурсов. Но это не гарантия: если изменить ресурсы без перезапуска невозможно, под будет пересоздан.

## Шпаргалка

### Режимы VPA (`updatePolicy.updateMode`)

| Режим | Что делает |
|---|---|
| `Off` | только считает рекомендации, ничего не меняет |
| `Initial` | ставит ресурсы только при создании пода |
| `Recreate` | применяет рекомендации, выселяя поды |
| `InPlaceOrRecreate` | сначала пробует изменить на месте, при неудаче — пересоздаёт |
| `Auto` | устаревший синоним `Recreate` |

### Объект VPA

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: InPlaceOrRecreate
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
        controlledResources: ["cpu", "memory"]
```

В некоторых версиях VPA режим `InPlaceOrRecreate` нужно отдельно включить feature gate в компонентах VPA — смотрите документацию своей версии.

### Установка

```bash
# Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# VPA из репозитория autoscaler
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh       # удаление: ./hack/vpa-down.sh

kubectl get pods -n kube-system | grep vpa
```

### Проверка и диагностика

```bash
kubectl describe vpa web-vpa                 # рекомендации: Target, Lower Bound, Upper Bound
kubectl top pod                              # фактическое потребление

# ресурсы, применённые к контейнеру (могут отличаться от spec во время изменения)
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].resources}'

# ход изменения: условия PodResizePending (Deferred / Infeasible) и PodResizeInProgress
kubectl get pod <pod> -o jsonpath='{.status.conditions}'

# число рестартов контейнера — убедиться, что изменение прошло без перезапуска
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'
```

| Статус изменения | Что значит |
|---|---|
| `PodResizePending`, причина `Deferred` | сейчас на ноде не хватает ресурсов, kubelet попробует позже |
| `PodResizePending`, причина `Infeasible` | изменение невозможно на этой ноде в принципе |
| `PodResizeInProgress` | kubelet применяет изменение |

### Чек-лист

- [ ] Kubernetes 1.35+, ноды на cgroup v2.
- [ ] Metrics Server и VPA установлены.
- [ ] У контейнеров заданы начальные requests и limits (QoS не BestEffort).
- [ ] `resizePolicy` с `RestartContainer` для памяти у JVM и похожих приложений.
- [ ] `minAllowed` и `maxAllowed` в VPA.
- [ ] Сначала режим `Off` и анализ рекомендаций, затем `InPlaceOrRecreate`.
- [ ] VPA и HPA не управляют одним и тем же ресурсом (CPU/память) одного Deployment.
