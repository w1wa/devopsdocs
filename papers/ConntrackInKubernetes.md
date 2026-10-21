# Conntrack в Kubernetes: почему переполняется и что делать

> Конспект на русском по выпуску рассылки [Conntrack Exhaustion in Kubernetes: Causes and Fixes](https://newsletter.devopscube.com/p/conntrack-in-kubernetes) (DevOpsCube Newsletter). Это пересказ, а не перевод — подробности в оригинале. Иллюстрации подгружаются с сайта рассылки. Раздел «Шпаргалка» в конце — дополнение по документации ядра Linux и kube-proxy, а не часть статьи.

## Что это

Conntrack (connection tracking) — часть netfilter в ядре Linux: таблица в памяти ядра со всеми активными соединениями. Для каждого соединения она хранит:

- IP и порт источника;
- IP и порт назначения;
- куда соединение на самом деле перенаправлено после NAT;
- состояние (NEW, ESTABLISHED и т.д.);
- таймаут, по истечении которого запись удаляется.

![Что хранит conntrack](https://media.beehiiv.com/cdn-cgi/image/fit=scale-down,quality=80,format=auto,onerror=redirect/uploads/asset/file/c002f568-2e92-476b-8e12-5eb9317d259f/image.png?t=1779961997)

## Роль в Kubernetes

Больше всего conntrack использует kube-proxy при маршрутизации трафика через Service.

ClusterIP сервиса — виртуальный адрес: его нет ни на одном интерфейсе, ни у одного пода или ноды. Путь запроса:

1. Под-клиент отправляет запрос на IP сервиса.
2. Пакет попадает в ядро ноды, где его перехватывают iptables-правила kube-proxy.
3. Ядро делает **DNAT**: подменяет адрес назначения на IP конкретного пода.
4. Пакет доходит до пода.

Под ничего не знает о сервисе и отвечает от своего IP. Клиент же обращался к сервису, и ответ от незнакомого адреса он отбросит. Здесь и нужен conntrack: при DNAT он запомнил соответствие «исходный запрос → куда перенаправлен» и на обратном пути подменяет адрес источника обратно на IP сервиса. Для клиента всё выглядит так, будто ответил сам сервис.

![Трафик через Service и conntrack](https://media.beehiiv.com/cdn-cgi/image/fit=scale-down,quality=80,format=auto,onerror=redirect/uploads/asset/file/34b71efc-e9fe-4345-8a12-46389f0b7347/image.png?t=1780380643)

В кластерах kubeadm параметры conntrack задаются в `KubeProxyConfiguration`:

![conntrack в KubeProxyConfiguration](https://media.beehiiv.com/cdn-cgi/image/fit=scale-down,quality=80,format=auto,onerror=redirect/uploads/asset/file/c75406db-c349-40c9-a438-76d8efe1d027/image.png?t=1779969736)

## Как посмотреть таблицу

Утилита `conntrack` по умолчанию не установлена (в Ubuntu — пакет `conntrack`). Команда `conntrack -L` выводит записи. В каждой видно:

- протокол и состояние соединения;
- исходное направление — от клиента к сервису;
- обратное направление — от пода к клиенту, с переставленными адресами;
- флаг `ASSURED`: пакеты шли в обе стороны, соединение полностью установлено.

## Состояния соединений

| Состояние | Значение |
|---|---|
| `NEW` | первый пакет соединения, которое conntrack ещё не видел |
| `ESTABLISHED` | обе стороны обменялись пакетами, соединение активно |
| `RELATED` | новое соединение, связанное с существующим (например, канал данных FTP) |
| `INVALID` | пакет не относится ни к одному известному соединению; обычно отбрасывается |
| `TIME_WAIT` | соединение закрывается, запись ждёт удаления |

## Переполнение таблицы

Размер таблицы ограничен: часто около 131 072 записей, но значение зависит от ядра и объёма памяти. Kubernetes создаёт много NAT-трафика:

- ClusterIP-сервисы;
- kube-proxy в режиме iptables;
- пробы liveness и readiness;
- service mesh.

В нагруженном кластере с сотнями подов и тысячами соединений таблица заполняется быстро. Симптомы переполнения:

- случайные таймауты соединений;
- периодические сбои DNS;
- запросы к API, которые падают без понятной ошибки;
- поведение «то работает, то нет»;
- сервисы выглядят здоровыми, но соединения случайно обрываются.

Чтобы проверить, насколько таблица заполнена, сравните текущее число записей (`nf_conntrack_count`) с лимитом (`nf_conntrack_max`). Если они близки — проблема в conntrack.

## Как исправить

### 1. Увеличить размер таблицы

Самая быстрая и распространённая мера: поднять `net.netfilter.nf_conntrack_max` через `sysctl` и закрепить значение в `/etc/sysctl.conf`. На рабочих нодах Ubuntu часто ставят 262 144 и выше, в крупных кластерах — ещё больше.

Это временная мера, а не решение: большая таблица занимает больше памяти, поиск в ней медленнее, а настоящая причина (лишний трафик) остаётся скрытой.

### 2. Уменьшить таймауты TCP

По умолчанию ядро хранит простаивающее ESTABLISHED-соединение 5 дней, а TIME_WAIT — 120 секунд. В нагруженном кластере накапливаются тысячи «мёртвых» записей. Уменьшение таймаутов быстрее освобождает таблицу.

Осторожно: если приложение держит долгоживущие простаивающие соединения (например, пул соединений к БД), при слишком маленьком `timeout_established` запись удалится, и следующий пакет будет помечен как INVALID.

### 3. Масштабировать ноды горизонтально

Лимит `nf_conntrack_max` действует на ноду. Если распределить те же поды на большее число нод, на каждую придётся меньше соединений: 1000 подов на 5 нодах дают гораздо большее давление на таблицу, чем на 20.

### 4. NodeLocal DNSCache

По умолчанию не включён и разворачивается вручную как DaemonSet (на некоторых managed-платформах, например GKE Autopilot, уже включён).

- **Без кэша:** каждый DNS-запрос идёт под → ClusterIP kube-dns → DNAT → под CoreDNS и занимает запись conntrack.
- **С кэшем:** под → локальный кэш на той же ноде; при промахе — запрос в CoreDNS по TCP.

Для локального DNS-трафика используются правила `NOTRACK`, поэтому большая часть DNS-запросов вообще не попадает в conntrack.

![NodeLocal DNSCache](https://media.beehiiv.com/cdn-cgi/image/fit=scale-down,quality=80,format=auto,onerror=redirect/uploads/asset/file/288bed5f-37d0-4e3f-8eca-560af7fbc669/image.png?t=1780981695)

В статье также есть ссылка на разбор реального инцидента из-за переполнения conntrack.

## Выводы

Conntrack — одна из самых важных и при этом недооценённых частей сетевой подсистемы Kubernetes. Даже CNI на eBPF, заменяющие kube-proxy (например, Cilium), не избавляют от него полностью: внешний SNAT, NAT ядра и другие stateful-операции всё равно опираются на отслеживание соединений. Переполнение возможно:

- с nftables;
- в кластерах на eBPF;
- в service mesh;
- при интенсивном DNS-трафике.

Поэтому в продакшене обязательно мониторьте заполненность через метрики node-exporter `node_nf_conntrack_entries` и `node_nf_conntrack_entries_limit` — так видно перегруженные ноды.

## Шпаргалка

### Команды

```bash
sudo apt install -y conntrack                 # установить утилиту (Ubuntu)

sudo conntrack -L                             # все записи
sudo conntrack -L -p tcp --dport 53           # отфильтровать (здесь DNS по TCP)
sudo conntrack -C                             # число записей
sudo conntrack -S                             # статистика по CPU: drop, insert_failed, early_drop

cat /proc/sys/net/netfilter/nf_conntrack_count   # сейчас
cat /proc/sys/net/netfilter/nf_conntrack_max     # лимит

dmesg | grep -i "nf_conntrack: table full"    # явный признак переполнения
```

### Параметры sysctl

| Параметр | Значение по умолчанию ядра | Смысл |
|---|---|---|
| `net.netfilter.nf_conntrack_max` | зависит от памяти | максимум записей |
| `net.netfilter.nf_conntrack_buckets` | зависит от памяти | размер хэш-таблицы |
| `net.netfilter.nf_conntrack_tcp_timeout_established` | 432000 (5 дней) | сколько хранить простаивающее соединение |
| `net.netfilter.nf_conntrack_tcp_timeout_time_wait` | 120 | TIME_WAIT |
| `net.netfilter.nf_conntrack_tcp_timeout_close_wait` | 60 | CLOSE_WAIT |

```bash
# временно
sudo sysctl -w net.netfilter.nf_conntrack_max=262144
# постоянно
echo "net.netfilter.nf_conntrack_max=262144" | sudo tee /etc/sysctl.d/90-conntrack.conf
sudo sysctl --system
```

### Настройки в kube-proxy

kube-proxy при старте сам выставляет часть этих параметров, поэтому ручной `sysctl` может быть перезаписан. Лучше задавать их в конфигурации kube-proxy:

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
conntrack:
  maxPerCore: 32768            # лимит = maxPerCore × число ядер, но не меньше min
  min: 131072
  tcpEstablishedTimeout: 24h   # kube-proxy по умолчанию ставит 24 ч вместо 5 дней ядра
  tcpCloseWaitTimeout: 1h
```

### Алерт Prometheus

```yaml
- alert: ConntrackTableAlmostFull
  expr: node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Таблица conntrack на {{ $labels.instance }} заполнена более чем на 80%"
```

### Порядок диагностики

1. Симптомы: случайные таймауты, сбои DNS, ошибки без явной причины.
2. Сравнить `nf_conntrack_count` и `nf_conntrack_max`; посмотреть `conntrack -S` и `dmesg`.
3. Найти, кто создаёт соединения: `conntrack -L` с группировкой по адресам и портам.
4. Срочно: поднять `nf_conntrack_max`.
5. Надолго: NodeLocal DNSCache, разумные таймауты, переиспользование соединений в приложениях, больше нод.
6. Включить алерт на заполненность.
