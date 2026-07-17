# Домашнее задание к занятию «Микросервисы: подходы» - Юрочкин В.А.

## Задача 1: Обеспечить разработку

### Предлагаемое решение: **GitLab (облачная версия — GitLab.com)**

GitLab — это комплексная DevOps-платформа, покрывающая в одном продукте хранение кода, CI/CD, управление секретами и реестр контейнеров. Облачная версия (gitlab.com) не требует поддержки собственной инфраструктуры для самой платформы, при этом позволяет подключать собственные раннеры.

#### Соответствие требованиям

| Требование | Реализация в GitLab |
|---|---|
| Облачная система | GitLab.com — SaaS, доступен без развёртывания |
| Система контроля версий Git | Git — нативный протокол GitLab |
| Репозиторий на каждый сервис | Отдельный проект (Project) в GitLab на каждый микросервис; группы (Groups) для организации |
| Запуск сборки по событию из VCS | Триггеры на push, merge request, tag через встроенные вебхуки GitLab CI |
| Запуск сборки по кнопке с параметрами | `when: manual` в `.gitlab-ci.yml` + переменные через `variables:` с `description:` — UI для ввода при запуске |
| Привязка настроек к каждой сборке | CI/CD Variables на уровне проекта, группы или окружения (`environment:`) |
| Шаблоны для различных конфигураций | `include:` (подключение шаблонных `.yml`-файлов) + `extends:` (наследование блоков) |
| Безопасное хранение секретов | CI/CD Variables с флагами **Masked** и **Protected**; интеграция с HashiCorp Vault |
| Несколько конфигураций из одного репозитория | Несколько `job`-ов / `stage`-ов в одном `.gitlab-ci.yml`; `rules:` для условного включения; matrix-сборки через `parallel: matrix:` |
| Кастомные шаги при сборке | Секция `script:` / `before_script:` / `after_script:` произвольные shell-команды |
| Собственные Docker-образы для сборки | Директива `image:` на уровне job или глобально; образы из GitLab Container Registry |
| Агенты сборки на собственных серверах | **GitLab Runner** — self-hosted, регистрируется через токен, поддерживает executor: shell, docker, kubernetes |
| Параллельный запуск нескольких сборок | Несколько раннеров + параллельные `stages`; `parallel:` keyword |
| Параллельный запуск тестов | `parallel: N` — GitLab разбивает тест-сьют на N параллельных job-ов; поддержка `pytest-split`, `rspec --bisect` и т.д. |

#### Пример структуры `.gitlab-ci.yml`

```yaml
include:
  - project: 'infra/ci-templates'
    file: '/docker-build.yml'

variables:
  DOCKER_REGISTRY: registry.gitlab.com/myorg

stages:
  - build
  - test
  - deploy

build:
  extends: .docker-build-template
  image: docker:24
  script:
    - docker build -t $DOCKER_REGISTRY/$CI_PROJECT_NAME:$CI_COMMIT_SHA .
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

test:parallel:
  stage: test
  image: python:3.12
  parallel: 4
  script:
    - pytest --split=$CI_NODE_INDEX --splits=$CI_NODE_TOTAL

deploy:staging:
  stage: deploy
  when: manual
  environment: staging
  variables:
    TARGET_ENV: staging
  script:
    - ./deploy.sh
```

#### Обоснование выбора

- **Единая платформа**: Git + CI/CD + Container Registry + Secrets — один инструмент вместо связки GitHub + Jenkins + Vault.
- **Гибкость раннеров**: self-hosted GitLab Runner позволяет использовать собственные мощные серверы для тяжёлых сборок, не платя за минуты в облаке.
- **Зрелая экосистема шаблонов**: механизм `include` + `extends` позволяет создать централизованный репозиторий CI-шаблонов для всей организации.
- **Встроенная безопасность**: переменные с маскированием, protected branches, SAST/DAST сканеры из коробки (GitLab Ultimate).

---

## Задача 2: Логи

### Предлагаемое решение: **ELK-стек (Elasticsearch + Logstash/Filebeat + Kibana)**

Классический и хорошо проверенный в продакшене стек для централизованного сбора, хранения и анализа логов.

#### Архитектура решения

```
[Микросервисы / контейнеры]
        │  stdout / stderr
        ▼
[Filebeat — агент на каждом хосте]
        │  beats protocol (TLS)
        ▼
[Logstash — парсинг, обогащение, фильтрация]
        │  HTTP / native
        ▼
[Elasticsearch — индексирование и хранение]
        │
        ▼
[Kibana — UI: поиск, фильтры, дашборды, ссылки]
```

#### Соответствие требованиям

| Требование | Реализация |
|---|---|
| Сбор со всех хостов | **Filebeat** устанавливается как DaemonSet (Kubernetes) или системный сервис на каждом хосте |
| Минимальные требования к приложениям, stdout | Filebeat читает stdout/stderr контейнеров напрямую из `/var/log/containers/*.log` или Docker socket — приложение просто пишет в stdout |
| Гарантированная доставка | Filebeat хранит **registry** с позицией чтения; при недоступности Logstash — буферизует локально; Logstash + persistent queue гарантируют at-least-once доставку |
| Поиск и фильтрация | Elasticsearch — полнотекстовый поиск, фильтрация по полям, range-запросы по времени через Kibana Discover |
| Пользовательский интерфейс | **Kibana** — UI с настройкой прав (role-based access); разработчики получают доступ к нужным индексам через Space/Role |
| Ссылка на сохранённый поиск | Kibana сохраняет поисковые запросы (Saved Searches) и генерирует постоянные URL — можно отправить коллеге |

#### Конфигурация Filebeat для сбора из stdout (пример)

```yaml
# filebeat.yml
filebeat.inputs:
  - type: container
    paths:
      - /var/log/containers/*.log
    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

output.logstash:
  hosts: ["logstash:5044"]
  ssl.enabled: true
```

#### Обоснование выбора

- **Нулевые требования к приложениям**: Filebeat работает на уровне ОС/оркестратора — сервисы ничего не знают о системе логирования.
- **Масштабируемость**: Elasticsearch горизонтально масштабируется (шарды, реплики); Logstash — добавлением узлов.
- **Гибкость парсинга**: Logstash Grok-фильтры позволяют разобрать любой формат (JSON, plaintext, nginx access log и т.д.).
- **Kibana Spaces**: изоляция данных между командами; разработчики видят только свои сервисы.
- **Saved Searches + Deep Links**: каждый поиск в Kibana имеет уникальный URL — удобно для передачи в тикет или алерт.

> **Альтернатива**: Grafana Loki + Promtail + Grafana — более легковесное решение, особенно актуальное если Grafana уже используется для мониторинга (см. Задача 3). Loki не индексирует содержимое логов, что снижает стоимость хранения, но ограничивает полнотекстовый поиск.

---

## Задача 3: Мониторинг

### Предлагаемое решение: **Prometheus + Grafana + Node Exporter + cAdvisor**

De-facto стандарт для мониторинга микросервисных систем, особенно в Kubernetes-окружениях.

#### Архитектура решения

```
[Хосты / VM]                    [Контейнеры / Поды]           [Сервисы]
     │                                  │                          │
[Node Exporter]              [cAdvisor / kubelet]       [Prometheus клиент]
(CPU, RAM, Disk, Net)        (CPU, RAM, Net per pod)    (/metrics endpoint)
     │                                  │                          │
     └──────────────────────────────────┴──────────────────────────┘
                                        │  HTTP scrape
                                        ▼
                                  [Prometheus]
                               (сбор, хранение, алерты)
                                        │
                                        ▼
                                   [Grafana]
                         (дашборды, запросы, алерты UI)
                                        │
                              [Alertmanager]
                          (маршрутизация алертов → Slack/PagerDuty)
```

#### Соответствие требованиям

| Требование | Реализация |
|---|---|
| Сбор метрик со всех хостов | **Node Exporter** — агент на каждом хосте, scrape из Prometheus |
| CPU, RAM, HDD, Network хостов | Node Exporter: `node_cpu_*`, `node_memory_*`, `node_disk_*`, `node_network_*` |
| CPU, RAM, HDD, Network на сервис | **cAdvisor** (встроен в kubelet): `container_cpu_usage_seconds_total`, `container_memory_usage_bytes`, `container_network_*` |
| Метрики, специфичные для сервиса | Prometheus-клиент в приложении (Go/Python/Java/Node библиотеки) → кастомные метрики на `/metrics`; Prometheus scrape по аннотациям pod-ов |
| UI с запросами и агрегацией | **Grafana Explore** — произвольные PromQL-запросы, агрегации, функции (`rate()`, `sum by()`, `histogram_quantile()`) |
| Настраиваемые панели | **Grafana Dashboards** — drag-and-drop, JSON-экспорт/импорт, библиотека готовых дашбордов (grafana.com/dashboards) |

#### Пример PromQL-запросов

```promql
# CPU utilization по сервису (контейнеру)
rate(container_cpu_usage_seconds_total{namespace="production", container!=""}[5m])

# RAM потребление сервиса
container_memory_usage_bytes{namespace="production", pod=~"api-.*"}

# 95-й перцентиль времени ответа (кастомная метрика сервиса)
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{job="order-service"}[5m]))

# Доступное место на диске, %
(node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
```

#### Обоснование выбора

- **Pull-модель Prometheus**: надёжна и безопасна — Prometheus сам ходит за метриками, агенты не нужно настраивать на push-эндпоинт.
- **PromQL**: мощный язык запросов с поддержкой агрегаций, функций скользящего окна, многомерных меток.
- **Kubernetes-native**: cAdvisor встроен в kubelet; Prometheus Operator автоматизирует конфигурацию через CRD (`ServiceMonitor`, `PodMonitor`).
- **Grafana**: богатый UI, ролевая модель доступа, поддержка множества источников данных (Prometheus, Loki, Elasticsearch одновременно).
- **Экосистема**: тысячи готовых экспортёров для PostgreSQL, Redis, Nginx, RabbitMQ и других сервисов.
- **Alertmanager**: гибкая маршрутизация алертов с группировкой, дедупликацией и подавлением (silences).

---

## Итоговая схема инфраструктуры

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitLab (Cloud)                           │
│  Git repos │ CI/CD Pipelines │ Container Registry │ Secrets     │
└────────────────────────────┬────────────────────────────────────┘
                             │ deploy
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                            │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │Service A │  │Service B │  │Service C │  │Service N │       │
│  │/metrics  │  │/metrics  │  │/metrics  │  │/metrics  │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│       │ stdout      │ stdout      │ stdout      │ stdout        │
│       ▼             ▼             ▼             ▼               │
│  ┌─────────────────────────────────────────────────┐           │
│  │          Filebeat (DaemonSet)                   │           │
│  └──────────────────────┬──────────────────────────┘           │
│                         │                                       │
│  ┌──────────────────────▼──────────────────────────┐           │
│  │  Node Exporter + cAdvisor (DaemonSet)           │           │
│  └──────────────────────┬──────────────────────────┘           │
└───────────────────────┬─┴────────────────────────┬─────────────┘
                        │                          │
             ┌──────────▼────────┐      ┌──────────▼────────┐
             │   ELK Stack       │      │  Prometheus Stack  │
             │ Logstash          │      │  Prometheus        │
             │ Elasticsearch     │      │  Alertmanager      │
             │ Kibana (UI/поиск) │      │  Grafana (UI)      │
             └───────────────────┘      └───────────────────┘
```

## Используемые технологии

| Область | Инструмент | Версия (актуальная) |
|---|---|---|
| Source Control + CI/CD | GitLab | SaaS (latest) |
| CI Runner | GitLab Runner | 17.x |
| Сбор логов (агент) | Filebeat | 8.x |
| Обработка логов | Logstash | 8.x |
| Хранение логов | Elasticsearch | 8.x |
| UI логов | Kibana | 8.x |
| Сбор метрик (хост) | Node Exporter | 1.8.x |
| Сбор метрик (контейнер) | cAdvisor | 0.49.x |
| Хранение метрик | Prometheus | 2.x |
| Алерты | Alertmanager | 0.27.x |
| UI мониторинга | Grafana | 11.x |
