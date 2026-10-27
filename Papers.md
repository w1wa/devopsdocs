 - В этой [статье](https://www.romaglushko.com/blog/k8s-gateway-api/) объясняется, почему **Ingress** заменяют на **Gateway API**, за что на самом деле отвечают новые ресурсы Gateway, Route и Policy, а также как выбрать gateway-контроллер перед миграцией. ([конспект на русском](papers/K8sGatewayAPI.md))

  - [Архитектура](https://devopscube.com/argo-cd-architecture) **Argo CD** ([конспект на русском](papers/ArgoCDArchitecture.md)):
    - Ключевые компоненты и их фактические роли
    - Как они взаимодействуют между собой и синхронизируют изменения
    - Как Argo CD хранит данные и как правильно настраивать бэкапы
    - Как запускать Argo CD в режиме высокой доступности
    - Безопасность и мониторинг (Prometheus + Grafana)

 - [Гайд](https://devopscube.com/terraform-module-best-practices/), который поможет разобраться с Terraform-модулями и тем, как правильно организовать их структуру. ([конспект на русском](papers/TerraformModuleBestPractices.md))

 - В этом [туториале](https://the-devops-engineer.medium.com/kubernetes-keda-autoscaling-scale-smarter-not-harder-d186da29175a) показано, как **KEDA** масштабирует ворклоады на основе глубины очереди, а не загрузки CPU, на полноценном примере с RabbitMQ: установка, настройка TriggerAuthentication, ScaledObject и нагрузочный тест, который подтверждает, что масштабирование работает. ([конспект на русском](papers/KEDAAutoscaling.md))

 -  Этом [гайде](https://newsletter.devopscube.com/p/conntrack-in-kubernetes) показано, как работает **conntrack** на реальных сценариях Kubernetes-сетей, и поймёте, почему он играет критически важную роль в работе Kubernetes Services, kube-proxy, NAT и DNS-трафика. ([конспект на русском](papers/ConntrackInKubernetes.md))
    - что такое conntrack и зачем он нужен;
    - почему Kubernetes Services зависят от него;
    - как посмотреть таблицу conntrack;
    - что происходит, когда таблица переполняется;
    - как диагностировать и устранять исчерпание conntrack в продакшене.

- [GitLab Architecture: A Complete Guide](https://devopscube.com/gitlab-architecture/) ([конспект на русском](papers/GitLabArchitecture.md))
    - Основные компоненты GitLab
    - Хранилище GitLab
    - Высокая доступность и масштабируемость
    - Аутентификация и авторизация
    - Мониторинг GitLab с помощью Prometheus и Grafana

- **In-Place Pod Resize** позволяет изменить CPU или memory для уже запущенного Pod без его перезапуска. [Подробный материал](https://devopscube.com/vpa-in-place-pod-resize) ([конспект на русском](papers/VPAInPlacePodResize.md)), в котором разобрали:
    - Что такое In-Place Pod Resize
    - Как это работает под капотом
    - Зачем нужен VPA для изменения ресурсов Pod без перезапуска
    - Что происходит, если на ноде не хватает ресурсов
    - Когда стоит использовать Resize Policy
    - С какими проблемами можно столкнуться при уменьшении ресурсов и многое другое
    - 

https://disk.yandex.ru/d/NFd7yOVZ5At3QQ\
