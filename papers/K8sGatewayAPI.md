# Kubernetes Gateway API: почему Ingress уступает место Gateway API

> Конспект на русском по статье [Kubernetes Gateway API](https://www.romaglushko.com/blog/k8s-gateway-api/) (Roman Glushko). Это пересказ, а не дословный перевод: полные манифесты, ссылки и детали — в оригинале. Иллюстрации подгружаются с сайта автора. Раздел «Шпаргалка» в конце — дополнение по документации Gateway API, а не часть статьи.

## Контекст

В ноябре 2025 года Kubernetes объявил, что NGINX Ingress Controller (стоит более чем в 40% кластеров) перестанет поддерживаться в марте 2026. После этого Gateway API окончательно закрепился как преемник Ingress API.

## Как появился Ingress

Исходно сервисы публиковались через типы `Service`:

- **ClusterIP** — стабильное DNS-имя только внутри кластера;
- **NodePort** — порт на каждой ноде, через который можно попасть в сервис снаружи;
- **LoadBalancer** — внешний L4-балансировщик (создаётся Cloud Controller Manager) поверх NodePort;
- **ExternalName** — CNAME-алиас на внешний сервис в DNS кластера.

NodePort слишком низкоуровневый: балансировку между нодами, маршрутизацию по путям и синхронизацию портов приходится делать самому.

![NodePort Service](https://www.romaglushko.com/blog/k8s-gateway-api/nodeport-service.png)

LoadBalancer на каждый сервис — это много публичных IP, лишние расходы и отсутствие единой точки контроля входящего трафика (а именно её атакующие прощупывают первой).

![LoadBalancer Service](https://www.romaglushko.com/blog/k8s-gateway-api/loadbalancer-service.png)

Логичное решение — один LoadBalancer, за которым стоит reverse proxy (например, NGINX), маршрутизирующий запросы по хосту и пути. Из этой схемы и выросли Ingress API и Ingress-контроллеры.

![LoadBalancer + reverse proxy](https://www.romaglushko.com/blog/k8s-gateway-api/loadbalancer-proxy.png)

## Ingress API

Появился в 2015 году и состоит из двух ресурсов. Реализации «из коробки» нет — нужен Ingress-контроллер.

![Kubernetes Ingress API](https://www.romaglushko.com/blog/k8s-gateway-api/k8s-ingress-api.png)

- **IngressClass** — кластерный ресурс: какой контроллер обрабатывает набор Ingress и с какими параметрами (через ссылку на CRD реализации). Позволяет держать в кластере несколько контроллеров одновременно.
- **Ingress** — правила маршрутизации (по хосту и пути, типы `Exact`/`Prefix`) плюс настройки TLS.

![Несколько Ingress-контроллеров в кластере](https://www.romaglushko.com/blog/k8s-gateway-api/multiple-ingress-controllers.png)

### Проблемы Ingress

1. **Слишком узкий API.** Таймауты, CORS, лимит размера тела, sticky-сессии, canary, rate limiting, маршрутизация по заголовкам — ничего этого в спецификации нет. Всё настраивается аннотациями конкретного контроллера (или его CRD, как у Traefik), поэтому чем больше функций — тем хуже переносимость. Поддержка gRPC, TCP и UDP тоже целиком зависит от реализации.
2. **Негибкая расширяемость.** Аннотации — это строки «ключ-значение», сложную конфигурацию приходится сериализовать в строку.
3. **Нет механизма политик.** Команда приложения может аннотацией отключить, например, rate limiting, который платформенная команда считает обязательным. Остаётся только внешний policy engine (Kyverno, OPA Gatekeeper).
4. **Размытая ответственность.** В одном ресурсе смешаны маршруты (зона команды приложения), хосты, порты и TLS (зона платформенной команды) и аннотации (обе зоны). У ресурса больше одной причины для изменения — признак того, что его стоит разделить.
5. **Нет безопасных ссылок между namespace.** Backend Ingress не может указывать на сервис в другом namespace. Обходной путь через `ExternalName`-сервис фактически открывает дорогу атаке *confused deputy* (см. раздел про ReferenceGrant).

![Обход через ExternalName](https://www.romaglushko.com/blog/k8s-gateway-api/k8s-ingress-external-name-bridge.png)

## Gateway API

Следующее поколение Ingress API:

- гораздо больше возможностей для управления входящим трафиком;
- ресурсы разделены по зонам ответственности команд;
- есть штатный механизм расширения (политики) и явные ссылки между namespace.

![Kubernetes Gateway API](https://www.romaglushko.com/blog/k8s-gateway-api/k8s-gateway-api.png)

### GatewayClass

Аналог IngressClass: указывает контроллер (`controllerName`) и может ссылаться на CRD реализации с параметрами — как публиковать прокси, сколько реплик, как раскатывать и масштабировать.

### Gateway

Динамически создаваемая точка входа в кластер. В мире Ingress эту роль играл сам контроллер — по сути, «статический» шлюз. Gateway ссылается на GatewayClass, а его главная часть — список **listeners**.

- **Listener** — отдельная точка входа: порт + протокол + hostname, опционально TLS (`Terminate` или `Passthrough`). Это самый мелкий уровень, к которому привязываются маршруты.
- **ListenerSet** — позволяет вынести listeners в namespace команд. Нужен, когда у каждого сервиса свой HTTPS-хост и сертификат: у одного Gateway лимит в 64 listener'а, и он быстро становится узким местом согласования между командами.
- **allowedRoutes** — ограничение на listener'е: из каких namespace (по селектору) и каких типов (`kinds`) маршруты могут к нему подключаться. Например, к TLS-passthrough-порту базы данных не должны цепляться HTTPRoute из чужих namespace.

Упрощённый пример:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public
  namespace: infra
spec:
  gatewayClassName: envoy
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-cert
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              expose: "true"
```

### Routes

Основная гибкость Gateway API — в маршрутах: `HTTPRoute`, `GRPCRoute`, `TLSRoute`, `TCPRoute`, `UDPRoute`. Структура у всех похожа:

- **parentRefs** — к чему привязан маршрут: Gateway, ListenerSet, Service (для service mesh) или CRD реализации. Через `sectionName` (и `port` для TCP/UDP) можно привязаться к конкретному listener'у;
- **hostnames** — маршрутизация по хосту на уровне маршрута (если протокол позволяет);
- **rules** — `matches` (какие запросы), `filters` (обработка запроса/ответа) и `backendRefs` (куда отправлять, с весами). `backendRefs` необязательны, если ответ формирует сам фильтр, например редирект.

#### HTTPRoute

- **Сопоставление**: путь и хост (как в Ingress), а также заголовки, query-параметры, HTTP-метод. Применяется самое специфичное правило, порядок не важен. Пример: canary-версию видят только внутренние клиенты с определённым заголовком.
- **Фильтры**: `URLRewrite`, `RequestHeaderModifier`/`ResponseHeaderModifier`, отдельный фильтр `CORS`, `RequestRedirect` (включая HTTP→HTTPS), `RequestMirror` — зеркалирование трафика для tap-and-compare-тестирования при рефакторинге.
- **Веса backend'ов** — для blue-green и A/B.
- **sessionPersistence** — sticky-сессии по cookie или заголовку.
- **ExternalAuth** (экспериментально) — перед отправкой в backend заголовки запроса уходят в сервис аутентификации по HTTP (ожидается ответ 200) или gRPC (API `ext_authz` Envoy).
- **Таймауты и ретраи** (коды, число попыток, backoff). Глобальный бюджет ретраев против retry storm задаётся через `BackendTrafficPolicy`.

Пример с весовым разделением трафика:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop
  namespace: shop
spec:
  parentRefs:
    - name: public
      namespace: infra
      sectionName: https
  hostnames: ["shop.example.com"]
  rules:
    - matches:
        - path: { type: PathPrefix, value: /api }
      backendRefs:
        - name: shop-v1
          port: 8080
          weight: 90
        - name: shop-v2
          port: 8080
          weight: 10
```

#### GRPCRoute

Технически gRPC работает поверх HTTP/2, но отдельный ресурс оправдан:

- не все HTTP-фильтры имеют смысл для gRPC (например, переписывание URL);
- у gRPC своя семантика ошибок: HTTP 200 при ошибке в `grpc-status` — это важно для ретраев и метрик;
- маршруты удобнее описывать в терминах `service`/`method`.

Тело запроса (protobuf) не разбирается, маршрутизировать по полям сообщения нельзя. Поддерживаются зеркалирование, веса и работа с заголовками.

#### TCPRoute и UDPRoute

Просто пересылают трафик с порта listener'а в backend — без matchers и фильтров, только с весами.

#### TLSRoute

Маршрутизация по SNI из TLS-рукопожатия, обычно в режиме passthrough (TLS терминирует сам backend). `hostnames` обязательны и должны пересекаться с hostname listener'а. Некоторые реализации (Envoy Gateway, kgateway) умеют и терминировать TLS, но это расширение.

#### Расширения и backend'ы

- HTTPRoute и GRPCRoute поддерживают фильтр `ExtensionRef` — ссылку на CRD реализации. Например, в Envoy Gateway так можно вернуть прямой ответ 503 «на обслуживании» вовсе без backend'а.
- Стандартные backend'ы — `Service` и `ServiceImport` (мультикластер). Реализации добавляют свои: в Envoy Gateway ресурс `Backend` указывает на внешний FQDN, IP или unix-сокет.

### ReferenceGrant

Ссылки между namespace в Gateway API — штатная возможность, поэтому к ним нужен контроль. Сценарий атаки *confused deputy*: в скомпрометированном namespace злоумышленник создаёт Ingress, сервис без селектора и вручную EndpointSlice с IP защищённых подов из другого namespace — получается «второй вход» в закрытый API. То же можно сделать проще, через ExternalName.

![Атака confused deputy](https://www.romaglushko.com/blog/k8s-gateway-api/k8s-ingress-confused-deputy-attack.png)

**ReferenceGrant** — механизм двустороннего доверия: namespace-владелец явно разрешает, из каких namespace можно ссылаться на его Service и Secret. Контроллер проверяет ReferenceGrant при каждой такой ссылке.

![ReferenceGrant](https://www.romaglushko.com/blog/k8s-gateway-api/k8s-gateway-refgrants.png)

![Как ReferenceGrant мешает атаке](https://www.romaglushko.com/blog/k8s-gateway-api/k8s-gateway-refgrants-mitigation.png)

ReferenceGrant лишь делает ссылку легитимной и не контролирует, куда backend проксирует трафик дальше. Поэтому дополнительно нужны точные NetworkPolicy.

## Политики (Policy attachment)

Способ иерархически навесить поведение на Gateway, ListenerSet или Route, не меняя сами ресурсы: политика содержит `targetRefs` (к чему применяется) и конфигурацию, специфичную для реализации.

![Политики Gateway API](https://www.romaglushko.com/blog/k8s-gateway-api/k8s-gateway-api-policies.png)

Типичный пример — аутентификация: включить её на весь Gateway (включая будущие маршруты) с исключениями для публичных маршрутов. Владеть ею может отдельная команда, которая не трогает Gateway и Route. В Envoy Gateway для этого есть `SecurityPolicy` (например, проверка JWT).

Политики — выразительная замена аннотациям. В стандарте пока две:

- **BackendTLSPolicy** — подключаться к backend'у по TLS;
- **BackendTrafficPolicy** — бюджет ретраев для backend'а (работает похоже на circuit breaker).

## Кто чем владеет

- **Платформенная команда**: GatewayClass, Gateway, ListenerSet, балансировщики, TLS и глобальные политики.
- **Команды приложений**: свои Route и политики уровня маршрута.

Возможны и другие модели — например, все маршруты в одном namespace платформенной команды.

![Владение ресурсами](https://www.romaglushko.com/blog/k8s-gateway-api/k8s-gateway-resource-ownership.png)

## Типичная архитектура

Обычно реализация разделена на две плоскости:

- **control plane** — Gateway Controller (оператор Kubernetes): следит за ресурсами Gateway API и собирает из них конфигурацию;
- **data plane** — прокси, который обслуживает трафик (Envoy, NGINX, Traefik и др.). Одни реализации поднимают отдельный прокси на каждый Gateway, другие делят data plane между несколькими.

Такое разделение устраняет часть фундаментальных проблем безопасности NGINX Ingress Controller.

![Типичная архитектура](https://www.romaglushko.com/blog/k8s-gateway-api/k8s-gateway-common-arch.png)

## Варианты миграции с NGINX Ingress Controller

1. **Ничего не делать.** Допустимо разве что в домашней лаборатории. В enterprise стандарты ISO 27001, SOC 2, HIPAA, PCI DSS и FedRAMP требуют поддерживаемого и обновляемого ПО. История уязвимостей: около 20 CVE за пять лет, включая критическую IngressNightmare (CVE-2025-1974). Она позволяла любому workload'у с сетевым доступом к admission controller получить RCE и прочитать доступные контроллеру Secrets. Инъекции конфигурации продолжились и в 2026 году. Причины структурные: огромная гибкость (сложно валидировать), control и data plane в одном поде, широкий доступ к Secrets.
2. **Поддерживаемый форк** (например, от Chainguard) — краткосрочное решение с исправлением CVE «по возможности».
3. **Другой Ingress-контроллер** (HAProxy, Istio, Traefik и др.). Сам Ingress API не устарел, но аннотации придётся переносить.
4. **Переход на Gateway API** — единственный долгосрочный вариант:
   - установить CRD и реализацию Gateway API;
   - настроить GatewayClass, Gateway и при необходимости политики;
   - перевести Ingress в Route. Утилита `ingress2gateway` делает это автоматически, но аннотации — таймауты, лимиты размера тела и загрузки — нужно проверить вручную, иначе можно незаметно сломать приложение.

   Переключать трафик можно поэтапно: оставить Ingress-контроллер точкой входа и часть запросов отправлять в новый Gateway.

## Как выбрать реализацию

Автор рассматривает типовой API Gateway: публичный north-south трафик, своя инфраструктура, аутентификация и гибкий rate limiting «из коробки». Его критерии:

- **Conformance.** Выбирайте реализацию, прошедшую официальные тесты (если отчёта нет на сайте Gateway API — запросите у мейнтейнеров). Тесты проверяют функциональность, но не надёжность и производительность.
- **Публичные бенчмарки** — например, сравнение от John Howard (контрибьютор Istio, сотрудник Solo.io, компании за kgateway). Это ещё одна полезная точка зрения.
- **OSS как отправная точка.** Многие функции, ради которых раньше покупали проприетарные решения (OAuth2, OIDC), уже есть в открытых проектах, и их качество можно оценить самому.

| Реализация | Data plane | Кратко |
|---|---|---|
| Envoy Gateway | Envoy | Лёгкий, сфокусирован на API Gateway. Богатая безопасность (ext auth, JWT, OIDC, API-ключи, IP allowlist), гибкий глобальный rate limiting, расширения через gRPC, Lua и WASM, ресурс `Backend`. Отличный выбор, если нужен только API Gateway. |
| Istio | Envoy | Полный пакет: Ingress, Gateway API, service mesh, мультикластер, mTLS. Проверен в бою, но тяжёлый и со своей кривой обучения. Лучший выбор, когда в первую очередь нужен mesh. |
| kgateway | Envoy / agentgateway | Полная conformance, трансформации на MiniJinja, иерархические маршруты, вызов AWS Lambda. Независимых отзывов пока мало. |
| Traefik | собственный | Простой и удобный, популярен в homelab. Расширение через middleware, JWT/OIDC — только платные плагины. Нет разделения control/data plane (риск единой точки отказа); обязательно нагрузочное тестирование. |
| NGINX Gateway Fabric | NGINX | Хорошая conformance, с версии 2.0 разделены control и data plane. JWT/OIDC, cookie-сессии и обновления без reload — только в NGINX Plus; rate limiting только локальный. Snippets упрощают перенос старой конфигурации NGINX. |
| Cilium | Envoy | Удобно, если Cilium уже используется как CNI, но возможностей сверх стандарта мало (нет JWT/OIDC, IP allowlist, плагинов). |
| Kong | NGINX | Сомнения в развитии OSS-версии: готовые образы новых версий не публикуются. Брать, если готовы к enterprise-лицензии или к самостоятельной сборке. |

## Итог

Статья прослеживает путь от Service через Ingress к Gateway API, подробно разбирает ресурсы нового API и сравнивает реализации. За рамками остались GAMMA (Gateway API для service mesh) и Inference Extension.

## Шпаргалка

### Ресурсы одним взглядом

| Ресурс | Область | Кто владеет | Зачем |
|---|---|---|---|
| `GatewayClass` | кластер | платформа | какой контроллер и с какими параметрами |
| `Gateway` | namespace | платформа | точка входа, listeners, TLS |
| `ListenerSet` | namespace | команда / платформа | вынести listeners (хосты и сертификаты) в namespace команд |
| `HTTPRoute` / `GRPCRoute` | namespace | команда приложения | L7-маршрутизация, фильтры, веса |
| `TLSRoute` | namespace | команда приложения | маршрутизация по SNI, passthrough |
| `TCPRoute` / `UDPRoute` | namespace | команда приложения | L4-пересылка порта в backend |
| `ReferenceGrant` | namespace цели | владелец цели | разрешить ссылки из других namespace |
| `BackendTLSPolicy` / `BackendTrafficPolicy` | namespace | платформа / команда | TLS до backend'а, бюджет ретраев |

### Ingress → Gateway API: что куда переезжает

| В Ingress | В Gateway API |
|---|---|
| `IngressClass` | `GatewayClass` |
| сам Ingress-контроллер | `Gateway` (+ прокси, который создаёт контроллер) |
| `spec.tls` | `listeners[].tls` в `Gateway` / `ListenerSet` |
| `rules[].host` | `hostnames` в Route (и `hostname` listener'а) |
| `rules[].http.paths` | `rules[].matches[].path` в `HTTPRoute` |
| аннотации таймаутов, CORS, редиректов, rewrite | поля `timeouts`, фильтры `CORS`, `RequestRedirect`, `URLRewrite` |
| аннотации canary | веса в `backendRefs` или match по заголовку |
| аннотации аутентификации, rate limit | политики реализации (`SecurityPolicy` и т.п.) или `ExternalAuth` |
| ExternalName-«мост» в другой namespace | прямая ссылка + `ReferenceGrant` |

### ReferenceGrant: пример

Разрешить HTTPRoute из namespace `shop` ссылаться на сервисы в namespace `payments`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-shop-routes
  namespace: payments          # namespace, на ресурсы которого ссылаются
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: shop
  to:
    - group: ""
      kind: Service
```

### Редирект HTTP → HTTPS: пример

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-redirect
  namespace: infra
spec:
  parentRefs:
    - name: public
      sectionName: http         # listener на 80 порту
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```

### Полезные команды

```bash
# установить стандартные CRD Gateway API (подставьте актуальную версию)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/<версия>/standard-install.yaml

# что есть в кластере
kubectl get gatewayclass
kubectl get gateway,httproute,grpcroute,referencegrant -A

# почему маршрут не работает: смотрите conditions Accepted и ResolvedRefs
kubectl describe httproute <имя> -n <namespace>

# конвертация Ingress в Gateway API (результат обязательно проверить вручную)
ingress2gateway print --providers ingress-nginx
```

### Чек-лист миграции

- [ ] Выбрать реализацию; проверить её conformance-отчёт и нужные функции (аутентификация, rate limiting).
- [ ] Установить CRD и контроллер, создать `GatewayClass` и `Gateway`.
- [ ] Определить, кто владеет `Gateway`, а кто — маршрутами; настроить `allowedRoutes`.
- [ ] Сконвертировать Ingress через `ingress2gateway`.
- [ ] Вручную перенести аннотации: таймауты, лимиты тела и загрузки, CORS, sticky-сессии, rate limiting.
- [ ] Для ссылок между namespace создать `ReferenceGrant` и проверить NetworkPolicy.
- [ ] Переключать трафик поэтапно и сравнивать метрики и ошибки до и после.
- [ ] Удалить старые Ingress и контроллер после полного переключения.
