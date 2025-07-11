# Пример GitLab CI/CD пайплайна для сборки, тестирования и отправки Helm чарта в GitLab Package Repository
GitLab CI/CD пайплайн автоматизирует процесс сборки, тестирования и отправки Helm чарта в GitLab Package Repository. Используя команду `helm lint` на этапе тестирования, вы можете убедиться, что ваш Helm чарт соответствует стандартам качества перед его отправкой в репозиторий

## Инструкция по созданию GitLab репозитория
Создание репозитория в GitLab — это первый шаг для начала работы с системой контроля версий и автоматизации CI/CD процессов. Следуйте этой инструкции, чтобы создать новый репозиторий в GitLab.

## 1. Регистрация и вход в GitLab
Перейдите по ссылке https://gitlab-devops-dtu.ip-dynamic.org/ на сайт GitLab.
Зарегистрируйтесь или войдите в свою учетную запись.

## 2. Создание нового проекта
После входа в систему нажмите на кнопку "New project" на главной странице или в меню навигации.
Выберите "Create blank project" для создания нового пустого проекта.

## 3. Настройка проекта
- `Project name`: Введите имя вашего проекта. Например, my-helm-chart.
- `Project slug`: Это автоматически сгенерированное поле на основе имени проекта. Оно будет использоваться в URL вашего репозитория.
- `Project description`: (Необязательно) Введите описание вашего проекта.
- `Visibility level`: Выберите уровень видимости проекта:
    - `Private`: Проект виден только вам и пользователям, которым вы предоставите доступ.
    - `Internal`: Проект виден всем зарегистрированным пользователям GitLab.
    - `Public`: Проект виден всем, включая незарегистрированных пользователей.

Нажмите кнопку `Create project` для создания нового проекта.
После создания проекта вы будете перенаправлены на страницу репозитория. Вам нужно клонировать репозиторий на локальную машину для дальнейшей работы.

На странице репозитория нажмите на кнопку `Clone` и скопируйте URL репозитория.
Откройте терминал на вашей локальной машине и выполните команду для клонирования репозитория:
```bash
git clone <URL вашего репозитория>
```
Например:
```
git clone https://gitlab-devops-dtu.ip-dynamic.org/your-username/my-helm-chart.git
```
## Добавление файлов в репозиторий
Перейдите в каталог вашего репозитория:
```bash
cd my-helm-chart
```
Добавьте файлы вашего проекта в репозиторий. Например, создайте файл README.md:
```bash
echo "# My Helm Chart" > README.md
```

###  Установите Helm
Если у вас еще не установлен Helm, установите его, следуя инструкциям на официальном сайте Helm. https://helm.sh/docs/intro/install/

### Создайте новый Helm Chart
Используйте команду `helm create` для создания нового Helm Chart:
Эта команда создаст структуру каталогов и файлов для вашего Helm Chart.
```bash
helm create my-helm-chart
```
Эта команда создаст структуру каталогов и файлов для вашего Helm Chart.

### Структура Helm Chart
После выполнения команды helm create, у вас будет следующая структура каталогов:

```
my-helm-chart/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── service.yaml
│   └── serviceaccount.yaml
└── .helmignore

```

### Внесите изменения в Helm Chart

Файл `Chart.yaml` содержит метаданные о вашем Helm Chart. Добавьте в чарт поле `maintainers` :
```yaml
maintainers: # (optional)
  - name: The maintainers name (required for each maintainer)
    email: The maintainers email (optional for each maintainer)
    url: A URL for the maintainer (optional for each maintainer)
``` 
измените версию вашего чарта:
```yaml
version: 0.1.0
```

Пример содержимого:

```yaml
apiVersion: v2
name: my-helm-chart
description: A Helm chart for Kubernetes

# A chart can be either an 'application' or a 'library' chart.
type: application

# This is the chart version. This version number should be incremented each time you make changes to the chart.
version: 0.1.0

# This is the version number of the application being deployed. This version number should be incremented each time you make changes to the application.
appVersion: 1.0.0
maintainers: # (optional)
  - name: Your Name
    email: your_email@example.com
```

Теперь у вас есть пример Helm Chart, который можно использовать для развертывания простого приложения в Kubernetes. Вы можете настроить файлы `Chart.yaml`, `values.yaml` и шаблоны в каталоге `templates/` в соответствии с вашими требованиями. Этот Helm Chart можно использовать в вашем GitLab CI/CD пайплайне для автоматизации сборки, тестирования и отправки в GitLab Package Repository.

Закоммитьте и отправьте `helm chart` в репозиторий:
```bash
git add . 
git commit -m "Add Helm Chart"
git push origin main
```

# Настройка GitLab CI/CD
Создайте файл `.gitlab-ci.yml` в корне вашего репозитория и добавьте в него конфигурацию пайплайна. Например:
**Файл .gitlab-ci.yml:**
```yaml
stages:
  - build
  - test
  - deploy

variables:
  HELM_PACKAGE_NAME: "demo-chart"
  HELM_PACKAGE_VERSION: "0.1.0"
  HELM_PACKAGE_REPO: "https://$CI_SERVER_HOST/api/v4/projects/$CI_PROJECT_ID/packages/helm"

build:
  stage: build
  script:
    - echo "Packaging Helm chart..."
    - helm package ./demo-chart --version $HELM_PACKAGE_VERSION --app-version $HELM_PACKAGE_VERSION
  artifacts:
    paths:
      - "*.tgz"

test:
  stage: test
  script:
    - echo "Linting Helm chart..."
    - helm lint ./demo-chart
  dependencies:
    - build

deploy:
  stage: deploy
  script:
    - echo "Uploading Helm chart to GitLab Package Repository..."
    - 'curl --fail-with-body --request POST
     --form "chart=@$HELM_PACKAGE_NAME-$HELM_PACKAGE_VERSION.tgz"
     --user gitlab-ci-token:$CI_JOB_TOKEN
     "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/helm/api/stable/charts"'

  dependencies:
    - build
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

```

## Описание:
`stages`: Определяет этапы пайплайна: build, test, deploy.
`variables`: Определяет переменные для имени пакета Helm, версии и URL репозитория.
`build`:
    **stage**: Указывает, что это этап сборки.
    **script**: Выполняет команду helm package для упаковки Helm чарта.
    **artifacts**: Сохраняет артефакты сборки (упакованный Helm чарт) для последующих этапов.
`test`:
    **stage**: Указывает, что это этап тестирования.
    **script**: Выполняет команду helm lint для проверки Helm чарта.
`deploy`:
    **stage**: Указывает, что это этап развертывания.
    **script**: Выполняет команду curl для загрузки упакованного Helm чарта в GitLab Package Repository.
    **rules**: Указывает, что этот этап выполняется только на ветке main.

Закоммитьте и отправьте файл .gitlab-ci.yml в репозиторий:
```bash
git add .gitlab-ci.yml
git commit -m "Add GitLab CI/CD pipeline configuration"
git push origin main
```
## Проверка результатов
### Шаги для проверки выполнения вашего GitLab CI/CD пайплайна

После настройки вашего GitLab CI/CD пайплайна важно убедиться, что он выполняется корректно. Вот шаги для проверки выполнения вашего GitLab CI/CD пайплайна:

#### 1. Закоммитьте и отправьте ваши изменения

Убедитесь, что вы закоммитили и отправили файл `.gitlab-ci.yml` и любые другие необходимые файлы в ваш GitLab репозиторий:

```bash
git add .gitlab-ci.yml
git commit -m "Add GitLab CI/CD config"
git push origin main
```

#### 2. Перейдите в ваш проект на GitLab

1. Откройте ваш веб-браузер и перейдите на ваш GitLab.
2. Перейдите в ваш проект, выбрав его из списка проектов или используя строку поиска.

#### 3. Доступ к странице CI/CD пайплайнов

1. В левом боковом меню нажмите на **CI/CD**, чтобы развернуть меню.
2. Нажмите на **Pipelines** (Пайплайны), чтобы просмотреть список пайплайнов для вашего проекта.

#### 4. Просмотр статуса пайплайна

На странице пайплайнов вы увидите список запущенных пайплайнов. Каждая запись пайплайна будет показывать следующую информацию:
- **Pipeline ID**: Уникальный идентификатор пайплайна.
- **Status**: Текущий статус пайплайна (например, выполняется, успешно, неудачно).
- **Stages**: Этапы, определенные в вашем файле `.gitlab-ci.yml` (например, build, test, deploy).
- **Duration**: Время, затраченное на выполнение пайплайна.
- **Trigger**: Событие, которое запустило пайплайн (например, push, merge request).

#### 5. Просмотр деталей пайплайна

1. Нажмите на ID пайплайна или на иконку статуса, чтобы просмотреть подробную информацию о выполнении пайплайна.
2. Вы увидите визуальное представление этапов и заданий пайплайна. Каждое задание будет иметь свой статус (например, выполняется, успешно, неудачно).

#### 6. Просмотр логов заданий

1. Нажмите на конкретное задание, чтобы просмотреть его детальные логи.
2. Логи задания покажут вывод команд, выполненных во время задания. Это полезно для отладки и понимания того, что произошло во время выполнения задания.

#### 7. Проверка артефактов 
1. Если ваши задания создают артефакты (например, артефакты сборки, отчеты о тестах), вы можете скачать их со страницы деталей задания.
2. Найдите раздел **Artifacts** (Артефакты) на странице деталей задания и нажмите на ссылки, чтобы скачать артефакты.

#### 8. Мониторинг выполнения пайплайна

1. Если ваш пайплайн все еще выполняется, вы можете мониторить его прогресс в реальном времени.
2. Статус каждого этапа и задания будет обновляться по мере их выполнения.

### 9. Проверка загрузки helm чарта в репозиторий проекта
1. В левом боковом меню нажмите на **Deploy**, чтобы развернуть меню.
2. Нажмите на **Package Registry**, чтобы просмотреть список артифактов в хранилище вашего проекта.

# Заключение
Теперь у вас есть новый репозиторий в GitLab с настроенным CI/CD пайплайном. Вы можете продолжать добавлять файлы и конфигурации в ваш проект, а также использовать GitLab CI/CD для автоматизации сборки, тестирования и развертывания вашего приложения.