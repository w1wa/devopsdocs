# Лучшие практики разработки Terraform-модулей

> Конспект на русском по статье [Terraform Module Development Best Practices](https://devopscube.com/terraform-module-best-practices/) (Alexander Sharov, DevOpsCube). Это сжатый пересказ, а не перевод: примеры конфигов (`.gitignore`, pre-commit, Dependabot, Renovate, CI) — в оригинале. Иллюстрации подгружаются с исходного сайта. Раздел «Шпаргалка» в конце — дополнение по документации Terraform/OpenTofu, а не часть статьи.

Всё описанное применимо и к OpenTofu — форку Terraform под лицензией MPL-2.0. Внедрять стоит то, что подходит вашему проекту.

## Конфигурация модуля

**Входы и выходы**
- понятные имена; всегда указывайте `type` (не `any`);
- задавайте значения по умолчанию, где это возможно, и используйте `validation`;
- секреты помечайте `sensitive = true`;
- документируйте переменные (удобно генерировать через `terraform-docs`);
- добавление переменной со значением по умолчанию обратно совместимо, удаление — нет;
- если значение никогда не будет меняться — это `locals`, а не переменная.

**Структура**
- Один модуль — один репозиторий. Подмодули допустимы, только если они прямо связаны с основным модулем.
- Стандартные файлы: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `README.md`. Большой `main.tf` делите по смыслу: `network.tf`, `ec2.tf`, `s3.tf`.
- Каталоги:
  - `modules/` — вложенные подмодули;
  - `examples/` — примеры использования;
  - `docs/` — документация и схемы;
  - `tests/` — тесты;
  - `scripts/` — скрипты, которые вызывает сам Terraform;
  - `helpers/` — вспомогательные скрипты для автоматизации;
  - `files/` — статические файлы и шаблоны;
  - `wrappers/` — обёртки для Terragrunt.

**Подмодули (`modules/<name>`)**
- По умолчанию считаются приватными, если в документации не сказано иное.
- Terraform не отслеживает рефакторинг: переименование или перенос ресурса в подмодуль приведёт к пересозданию. Используйте блоки `moved` и описывайте такие изменения в CHANGELOG как breaking.

**Скрипты (`scripts/`)** — крайняя мера: созданное скриптом не попадает в state. Сначала попробуйте:
- функции провайдера;
- альтернативный провайдер (например, `awscc` для AWS);
- TerraCurl.

Если без скрипта никак — задокументируйте причину и план отказа, а вызывайте через провижионер (`local-exec`).

**Вспомогательные скрипты (`helpers/`)** — для рутинных задач. Нужны описание, проверка аргументов и `--help`.

**Статические файлы (`files/`)**
- Длинные heredoc выносите в файлы и подключайте через `file()`.
- Шаблоны для `templatefile()` называйте `.tftpl`/`.tpl` и кладите в `files/templates/`.
- Сюда же — например, исходники Lambda-функций.

**Terragrunt-обёртка (`wrappers/`)** — для случаев, когда нативный `for_each` не подходит: несколько экземпляров модуля с разными параметрами. Обёртка — это `main.tf` с `for_each` по переменной `items`, где каждый вход берётся из элемента или из `defaults`, плюс `variables.tf` и `outputs.tf`.

**Провайдеры и backend** в переиспользуемых модулях не настраиваются — это задача корневого модуля. В `versions.tf` указывайте только минимальные версии провайдеров и Terraform (если не уверены — `>= 1.0`).

## Тестирование

- **Статический анализ** — линтеры, валидация, dry-run (см. раздел про качество кода).
- **Интеграционные тесты модуля** — развернуть модуль в тестовом окружении и проверить результат. Инструменты: встроенные `terraform test` и `check`, Terratest (Go), terraform-compliance (BDD), инструменты Google, InSpec.
- **End-to-end** — развернуть всё окружение целиком и проверить через API. Самый надёжный и самый дорогой способ.

Для разработки модулей актуальны первые два уровня: как и где модуль будут использовать, заранее не известно. Устаревшие инструменты (например, Kitchen-Terraform) не используйте.

## Документация

- Как можно больше примеров в `examples/`.
- Таблицы входов и выходов — автоматически через `terraform-docs`.
- Документируйте подмодули.
- В `README.md` — обзор, примеры, входы/выходы, совместимость версий и ограничения, бейджи (Terraform/OpenTofu).

## Git

- **`.gitignore`**: каталог `.terraform`, state-файлы, lock-файлы, лишние `tfvars`, артефакты сборки, логи.
- **`.gitattributes`**: единые окончания строк.
- **`.gitconfig`**: `autocrlf = input`, `longpaths = true` (пути длиннее 260 символов в Windows).

**Trunk-based development** (рекомендация DORA):
- `main` всегда чистая, защищённая и готова к использованию;
- ветки `feat/…` и `fix/…`, при необходимости `release/vX.Y.Z`;
- слияние через pull request, перед слиянием — rebase.

## Возможности GitHub

Файлы в `.github/` и корне репозитория делают проект удобнее для контрибьюторов.

![Репозиторий tofuutils/tenv, где используются почти все возможности GitHub](https://lh7-rt.googleusercontent.com/docsz/AD_4nXcF0LTdDgUQDzsqIj1GpHHaD0tTe5zndUHwvBbT1b0DHwneHbWsE_Zutr-eFMHY_eI4j3S9_FdZ9ebZ_8H7VltgMYS5rez8vPuVooiBJHhEy-sGwMMkSQtltYeBrCa-r8IQdJU-wqF_foouJguwVkU?key=jwhzJb-i8N1tskUDoQ63vl5X)

- **PULL_REQUEST_TEMPLATE.md** — шаблон описания PR с вопросами для ревью.
- **ISSUE_TEMPLATE/** — шаблоны issue. `config.yml` с `blank_issues_enabled: false` запрещает issue без шаблона.

![Выбор шаблона issue](https://lh7-rt.googleusercontent.com/docsz/AD_4nXeUGRjza0BVV0siZX-cmqwjLc8npxkAOn9RZ7bcep-vylQQynA33PGNq-FqheEUUr5VSPq2r6o-8Kwloovyc20V3uZa3mgOc89SScwnaWjHyOlZ2EF9ZfzLIXPFPox_TsMj6jjJcGElAutJA4IGIe0?key=jwhzJb-i8N1tskUDoQ63vl5X)

- **CONTRIBUTING.md** — как поднять окружение, правила PR и коммитов, ожидания от ревью.
- **LICENSE** — MIT (разрешительная, хороший вариант по умолчанию), Apache 2.0 (плюс защита патентов) или GPL (копилефт). Помогает выбрать choosealicense.com.
- **CODE_OF_CONDUCT.md**, **FUNDING.yml**, **CODEOWNERS** (автоназначение ревьюеров), **MAINTAINERS.md**, **SECURITY.md** (как сообщать об уязвимостях), **SUPPORT.md**.

## CI

Минимум для каждого push и PR: линтинг, форматирование, валидация, проверки соответствия политикам, проверка PR. Кроме этого стоит автоматизировать:

- **Версионирование** — SemVer (MAJOR — несовместимые изменения, MINOR — новая функциональность, PATCH — исправления). Автор использует GitHub Action `kvendingoldo/semver-action`: он ставит теги, создаёт release-ветки и GitHub Releases. Альтернатива — semantic-release. Для публичных модулей GitHub Releases обязательны.

![GitHub Release, созданный semver-action](https://lh7-rt.googleusercontent.com/docsz/AD_4nXfhPDubMJ3uGsf4X5xPGVGB8euUbjI--Qe5g2wbrDPBhRfzy-VIyTqWrOmTzqqrOtANM2YNXq-VU1bWSl04ThKqXhJmjFCyF17dttAtt3fEE4UF5S8f2HFPWK-hqAyOYEp7nWXfzsDmyHCZrUhRuyc?key=jwhzJb-i8N1tskUDoQ63vl5X)

- **CHANGELOG.md** — обязателен для публичных проектов. Изменения описываются в каждом PR, а обновление автоматизируется: Release Drafter, Conventional Changelog, go-changelog. Формат — по keepachangelog.com.
- **Обновление зависимостей**:
  - Dependabot — встроен в GitHub, есть алерты безопасности;
  - Renovate — гибче: группировка обновлений, обновление `versions.tf` и lock-файла, лучше работает вне GitHub (например, в GitLab).
- **Снижение рутины** — закрывать неактивные issue и PR, проверять формат заголовков PR. Всё, что делается вручную больше одного раза, стоит автоматизировать.
- **Готовые Actions** из Marketplace лучше своих: свои писать долго и сложно поддерживать.

## Качество кода

- Поиск секретов в репозитории: Gitleaks, git-secret, Talisman (pre-commit).
- Оценка стоимости инфраструктуры: Infracost.
- `tflint` и `terraform validate`; форматирование `terraform fmt` через pre-commit.
- Сканеры безопасности и соответствия:
  - Checkov;
  - Terrascan (политики на OPA);
  - Trivy (в него влился tfsec);
  - KICS.
- Политики как код: Open Policy Agent (Rego).
- В GitLab — встроенный IaC Scanning, доступный и на бесплатном плане.
- Для YAML, JSON и Markdown — GitHub Super-Linter, или `yamllint`/`jsonlint`, если он не подходит.

## Окружение разработчика

- **`.editorconfig`** — единый стиль во всех редакторах. Дополнительно можно хранить `.vscode/settings.json` и `.vscode/extensions.json` с рекомендуемыми плагинами.
- **pre-commit** — проверки до коммита (`pre-commit run --all-files`). Для старта подойдёт шаблонный репозиторий AWS для Terraform и готовые наборы хуков для Terraform и OpenTofu.

## Шаблонный репозиторий модулей

Если модулей десятки или сотни, поддерживать CI и линтеры копированием невозможно. Автор предлагает **Terraform Module Template Repository** — шаблон со всеми пайплайнами и конфигами, из которого новые модули создаются кнопкой «Use this template».

- В шаблонном репозитории обновления сложно доносить до дочерних: они технически независимы.
- В форке backmerge часто приводит к конфликтам.

В обоих случаях автор советует отдельный CI-job, который по расписанию раскатывает изменения из шаблона в дочерние репозитории. Для пары модулей это избыточно — проще копировать конфиги. Если же модулей ожидается много, шаблон стоит сделать с самого начала.

Во второй части серии автор обещает готовый шаблон, реализующий эти практики, и пример job'а для распространения обновлений.

## Шпаргалка

### Структура репозитория модуля

```text
terraform-aws-example/
├── .github/              # шаблоны PR/issue, CODEOWNERS, workflows
├── docs/
├── examples/
│   ├── basic/
│   └── complete/
├── files/
│   └── templates/        # *.tftpl
├── helpers/
├── modules/
│   └── <submodule>/
├── scripts/
├── tests/                # *.tftest.hcl
├── wrappers/             # Terragrunt-обёртка
├── .editorconfig
├── .gitignore
├── .pre-commit-config.yaml
├── CHANGELOG.md
├── LICENSE
├── README.md
├── main.tf
├── outputs.tf
├── variables.tf
└── versions.tf
```

### Переменная «по правилам»

```hcl
variable "environment" {
  description = "Окружение: dev, stage или prod"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "stage", "prod"], var.environment)
    error_message = "environment должен быть dev, stage или prod."
  }
}

variable "db_password" {
  description = "Пароль базы данных"
  type        = string
  sensitive   = true
}
```

### versions.tf в переиспользуемом модуле

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"   # только нижняя граница; верхнюю фиксирует корневой модуль
    }
  }
}
# блоков provider и backend здесь нет — они в корневом модуле
```

### Рефакторинг без пересоздания ресурсов

```hcl
# ресурс переехал в подмодуль — сообщаем Terraform, что это тот же объект
moved {
  from = aws_s3_bucket.logs
  to   = module.logging.aws_s3_bucket.this
}
```

### Встроенный тест (`tests/basic.tftest.hcl`)

```hcl
run "bucket_name_has_prefix" {
  command = plan

  variables {
    environment = "dev"
  }

  assert {
    condition     = startswith(aws_s3_bucket.this.bucket, "dev-")
    error_message = "Имя бакета должно начинаться с окружения."
  }
}
```

### Команды

```bash
terraform fmt -recursive          # форматирование
terraform validate                # проверка синтаксиса и ссылок
tflint --recursive                # линтер
terraform test                    # встроенные тесты
terraform-docs markdown table --output-file README.md .   # таблицы входов/выходов
trivy config .                    # сканер безопасности (бывший tfsec)
checkov -d .                      # ещё один сканер
infracost breakdown --path .      # оценка стоимости
pre-commit run --all-files        # все хуки разом
```

### Инструменты по задачам

| Задача | Инструменты |
|---|---|
| Форматирование и валидация | `terraform fmt`, `terraform validate` |
| Линтинг | tflint |
| Безопасность и соответствие | Checkov, Trivy, Terrascan, KICS, OPA |
| Секреты в коде | Gitleaks, git-secret, Talisman |
| Тесты | `terraform test`, `check`, Terratest, terraform-compliance |
| Документация | terraform-docs |
| Стоимость | Infracost |
| Зависимости | Dependabot, Renovate |
| Версии и релизы | semver-action, semantic-release |
| CHANGELOG | Release Drafter, Conventional Changelog, go-changelog |

### SemVer для модулей

| Изменение | Версия |
|---|---|
| удалена или переименована переменная, изменился тип, ресурс пересоздаётся | MAJOR |
| новая переменная со значением по умолчанию, новый output | MINOR |
| исправление без изменения интерфейса | PATCH |

### Чек-лист перед релизом модуля

- [ ] У всех переменных есть `type`, `description`, где нужно — `default` и `validation`.
- [ ] Секреты помечены `sensitive`.
- [ ] Нет блоков `provider` и `backend`; в `versions.tf` только минимальные версии.
- [ ] Переименования оформлены через `moved`.
- [ ] `fmt`, `validate`, `tflint`, сканер безопасности и тесты проходят в CI.
- [ ] README и таблицы входов/выходов обновлены, есть примеры в `examples/`.
- [ ] CHANGELOG дополнен, версия выбрана по SemVer, создан GitHub Release.
