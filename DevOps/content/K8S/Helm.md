
Теги: `#kubernetes` `#helm` `#package-manager` `#devops`

  

**Helm** — пакетный менеджер для Kubernetes. Он объединяет связанные Kubernetes-манифесты в один пакет (**Chart**) и позволяет устанавливать, настраивать, обновлять и откатывать приложение как единый релиз.

  

Helm не заменяет Kubernetes: он генерирует обычные YAML-манифесты и отправляет их в Kubernetes API.

  

```text

Chart + Values → Helm → Kubernetes-манифесты → Kubernetes API

```

  

---

  

## 1. Базовые понятия

  

- **Chart** — пакет с шаблонами Kubernetes-манифестов и значениями по умолчанию.

- **Release** — конкретный установленный экземпляр Chart в кластере.

- **Values** — параметры, которые Helm подставляет в шаблоны.

- **Repository** — хранилище готовых Charts.

- **Revision** — версия состояния Release после `install`, `upgrade` или `rollback`.

  

```text

Chart = пакет приложения

Release = установленный экземпляр пакета

```

  

Один Chart можно установить несколько раз с разными настройками:

  

```bash

helm install app-dev ./my-chart -f values-dev.yaml

helm install app-prod ./my-chart -f values-prod.yaml

```

  

В результате появятся два независимых Release: `app-dev` и `app-prod`.

  

---

  

## 2. Структура Helm Chart

  

Создать базовый Chart:

  

```bash

helm create my-chart

```

  

Структура:

  

```text

my-chart/

├── Chart.yaml # Метаданные и версия Chart

├── values.yaml # Значения по умолчанию

├── templates/ # Шаблоны Kubernetes-манифестов

├── charts/ # Зависимые Charts

└── .helmignore # Файлы, исключаемые из пакета

```

  

Пример `values.yaml`:

  

```yaml

replicaCount: 2

  

image:
repository: nginx

tag: "1.27"

  
service:
type: ClusterIP
port: 80

```

  

Использование Values в `templates/deployment.yaml`:

  

```yaml

spec:

replicas: {{ .Values.replicaCount }}

template:

spec:

containers:

- name: nginx

image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

```

  

После рендеринга Helm подставит конкретные значения и получит обычный Kubernetes YAML.

  

---

  

## 3. Конфигурации для разных окружений

  

Один Chart можно использовать для нескольких окружений:

  

```text

my-chart/

values-dev.yaml

values-stage.yaml

values-prod.yaml

```

  

Пример установки production-конфигурации:

  

```bash

helm install my-app ./my-chart -f values-prod.yaml

```

  

Значение можно переопределить через CLI:

  

```bash

helm install my-app ./my-chart \

-f values-prod.yaml \

--set image.tag=2.0.0

```

  

Приоритет значений:

  

```text

values.yaml < файл через -f < параметр через --set

```

  

Важно: обычно шаблоны Chart не редактируют для каждого окружения. Различия выносят в отдельные Values-файлы.

  

---

  

## 4. Основные команды (must-know)

  

### Репозитории

  

```bash

# Добавить репозиторий

helm repo add <name> <url>

  

# Обновить локальный список Charts

helm repo update

  

# Найти Chart

helm search repo <name>

  

# Посмотреть стандартные Values готового Chart

helm show values <repo/chart>

```

  

### Установка и просмотр

  

```bash

# Установить Chart

helm install <release-name> <chart>

  

# Установить в отдельный Namespace

helm install my-app ./my-chart \

--namespace production \

--create-namespace

  

# Показать Release в текущем Namespace

helm list

  

# Показать Release во всех Namespace

helm list --all-namespaces

  

# Посмотреть состояние Release

helm status my-app

```

  

### Обновление и откат

  

```bash

# Обновить существующий Release

helm upgrade my-app ./my-chart -f values-prod.yaml

  

# Установить Release, а если он существует — обновить

helm upgrade --install my-app ./my-chart -f values-prod.yaml

  

# Посмотреть историю ревизий

helm history my-app

  

# Откатиться к ревизии 2

helm rollback my-app 2

  

# Удалить Release

helm uninstall my-app

```

  

`helm upgrade --install` особенно часто используется в CI/CD, потому что одна команда подходит и для первой установки, и для последующих обновлений.

  

---

  

## 5. Проверка и отладка Chart

  

```bash

# Проверить Chart на распространённые ошибки

helm lint ./my-chart

  

# Локально сгенерировать итоговые YAML без установки

helm template my-app ./my-chart -f values-prod.yaml

  

# Проверить установку без сохранения изменений

helm install my-app ./my-chart --dry-run --debug

  

# Посмотреть Values установленного Release

helm get values my-app

  

# Посмотреть сгенерированные манифесты Release

helm get manifest my-app

```

  

Для отладки полезнее всего сначала выполнить:

  

```bash

helm lint ./my-chart

helm template my-app ./my-chart -f values-prod.yaml

```

  

---

  

## 6. Что Helm делает и не делает

  

Helm умеет:

  

- объединять множество YAML-манифестов в Chart;

- параметризовать шаблоны через Values;

- устанавливать и обновлять приложения;

- хранить историю Revision;

- откатывать Release;

- управлять зависимостями Charts.

  

Helm самостоятельно не умеет:

  

- создавать Kubernetes-кластер;

- писать логику пользовательских контроллеров;

- автоматически масштабировать Pod;

- заменять `Deployment`, `Service`, `Ingress` и другие ресурсы;

- постоянно синхронизировать кластер с Git как GitOps-контроллер.

  

Например, Helm может установить `HorizontalPodAutoscaler`, но само масштабирование выполняет Kubernetes.

  

---

  

## 7. Как Helm используется в Production

  

- Для сторонних приложений (`Prometheus`, `Grafana`, ingress-controller) часто используют готовые Charts.

- Для собственных приложений компания может написать отдельный Chart.

- Для похожих микросервисов часто создают общий внутренний Chart и передают разные Values.

- Chart и Values обычно хранят в Git и применяют через CI/CD или GitOps-инструменты.

  

```text

Код → Docker Image → новый image.tag в Values → Helm upgrade → Kubernetes

```

  

Важно: Helm не отменяет необходимость знать Kubernetes-манифесты. Чтобы написать или отладить Chart, нужно понимать ресурсы, которые находятся в `templates/`.

  

---

  

## Краткая шпаргалка

  

| Задача | Команда |
|:---|:---|
| Создать Chart | `helm create my-chart` |
| Проверить Chart | `helm lint ./my-chart` |
| Показать итоговый YAML | `helm template my-app ./my-chart` |
| Установить Release | `helm install my-app ./my-chart` |
| Установить или обновить | `helm upgrade --install my-app ./my-chart` |
| Показать Release | `helm list` |
| Показать историю | `helm history my-app` |
| Откатить Release | `helm rollback my-app 2` |
| Удалить Release | `helm uninstall my-app` |

  

Главная схема:

  

```text

Chart + Values = Release

```

  

**Helm превращает набор связанных Kubernetes-манифестов в единый настраиваемый и версионируемый пакет.**