
LAB create repository to deploy helm charts using helmfile

lab requirements:
kubectl, helm, helmchart, kubernetes cluster (local kind, microk8s, minikube or managed solution in the cloud gke, eks, aks etc.), gitlab server.
estimated time to complete 1h 30min 

1 create gitlab repository
2 create repository structure folders files etc. think of what abstraction layers could be in your setup (folders in release with values)

release-helmfile - релизный репозиторий
    ci - фолдер с gitlab ci yaml файлами
    charts - фолдер с локальными чартами для удобства и ускорения разработки чтобы не пересобирать чарт каждый раз как артефакт
    release - фолдер с вэльюс для релизов
        common-tier       - абстракция, слой для логической группировки релизов
            release-name1 - фолдеры должны иметь имена такие же как указаны в helmfile
            release-name2
                ....
            helmfile.yaml - helmfile  в котором описана устновка релизов
        another-tier      - абстракция, слой для логической группировки релизов
            release-name1 - фолдеры должны иметь имена такие же как указаны в helmfile
            release-name2
                ....
            helmfile.yaml - helmfile  в котором описана устновка релизов
        common.yaml - можно вынести общие настройки для helmfile укзать как и в каком порядке темлпейтить путь к вэльюс
        ....
    .gitignore
    .gitlab.yaml - основной файл описывающий запуск пайплайнов в корне репозитория
    environments.yaml - файл с описание окружений (путь к файлам с вэльюс, просто вэлью в явном виде и тд, фича флаги например ставить тот или иной релиз, здесь выполнятеся кастомизация каждого из окружений)
    helmfile.yaml - основной helmfile в котором указано расположение helmfile-ов расположенных в логических слоях
    README.md 

!important - create at least two layers of abstraction and deploy at least two charts one local and one from artifact registry

3 put existing helmchart or create new one in the charts folder
4 create helmfile for each of you logical tier folders and add a description of a release in each one for local chart and one more for the chart from artifact registry. you could resuse chart from lab `gitlab_ci_lab_helm.md`

#! give an example of release description one for local and for external

5 describe path to each of the logical tiers in helmfile located at the repository root (this is the main helmfile, content of all the helmfiles defined in ligical folders will be merged there)

#! give an example

6 define helmfile settings in common.yaml file (add remote registry alias here). next step is to define here description of all the values paths which helmfile will try to include while templating chart for release. path to values could be templated. define as much abstraction as you need all files which could not be found in release folders with values will be skipped

#! provide example and instructions

7 define several environments description in environments.yaml . values, feature flags and so on (ex kube-context, cluster name, something specific for the env)

#! give an example

8. create pipeline description in .gitlab.yaml to deploy helm charts to k8s cluster. all repeated parts that could be reused and stored in ci folder and included into the main .gitlab.ci file. also some required scripts for pipelines could be placed there.
 add variables i.e with secret values into repo ci/cd settings -> variables you could mask and safely use them in your pipelines ex. authorization to registry, k8s cluster, cloud project and so on.

#! for example before_script with auth, some logical grouping could also be made (dev, staging, prod and so on) beacuse sometimes if you have lots of envs it become difficult to find and manage pipeline in a very big file with thoussands of lines of code

possible pipline stages
scan repo for misconfigurations using trivy or any other scaner (could be run for main branch only on mr events) -> validate (lint your local helm charts - charts should be templated and linted without failures to continue) ->
plan (make a diff with current env state failure allowed for clean envs, diff should show all content as new) -> deploy helm charts to env.
jobs should be created for each env in each stage

one more action we could trigger deploy from another repo after helm charts where successfully builded and uploaded to artifact registry as .tgz artifacts 

9. add .gitignore file
10. add description of your repo to the readme.md file. its a good practice to describe the content and the purpose of the repo. also you could give some instruction how to work with it. (something specific developers should know, how to prepare local dev env etc)
