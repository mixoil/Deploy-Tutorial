# Инструкция по организации CI-CD приложения в Kubernetes

## Создание манифеста CI-CD (.gitlab-ci.yml)

Для начала, следует написать файл, описывающий последовательность интеграции, публикации и развертывания приложения. 
Информация о правилах написания файла, о том, как его выполняет раннер доступна в официальной документации: https://docs.gitlab.com/ee/ci/
Пример файла:
```
  stages:
    - build
    #TODO - test
    - deploy

  variables:
    TAG_LATEST: $CI_REGISTRY_IMAGE:latest
    TAG_COMMIT: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

  image: docker:latest

  build:
    stage: build
    before_script:
      - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    script:
      - docker build --pull -t $TAG_COMMIT -t $TAG_LATEST .
      - docker push $TAG_COMMIT
      - docker push $TAG_LATEST
    after_script:
      - docker logout $CI_REGISTRY
    when: manual
    only:
      - main

  deploy:
    stage: deploy
    before_script:
      - echo "$SSH_KEY" > ssh_private_key
      - chmod 600 ssh_private_key
    script:
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "rm /app/docker-compose.dev.yml"
      - scp -i ssh_private_key docker-compose.dev.yml $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP:/app
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "docker login -u "$CI_REGISTRY_FRONT_USER" -p 
"$CI_REGISTRY_FRONT_PASSWORD" $CI_REGISTRY"
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "cd /app/ && docker-compose -f docker-compose.dev.yml pull frontend"
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "docker logout $CI_REGISTRY"
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" 
$CI_REGISTRY"
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "cd /app/ && docker-compose -f docker-compose.dev.yml pull api && docker-
compose -f docker-compose.dev.yml up -d"
    after_script:
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "docker logout $CI_REGISTRY"
    when: manual
    only:
      - main
    environment:
      name: dev
```
В данном случае, интеграция и публикация реализованы в одном стейдже – build. Далее происходит деплой приложения с помощью docker-compose.dev.yml который разворачивает
докер контейнеры на указанном сервере (его ip подставляется из переменной $DEPLOY_HOST_IP) . Все переменные типа $DEPLOY_HOST_IP спрятаны в хранилище секретов gitlab 
репозитория. Вы можете основывать свою конфигурацию CD на этом стейдже, но если вы хотите развернуть приложение в Kubernetes, ваша конфигурация стейджа будет выглядеть 
примерно следующим образом:
```
stages:
  - deploy

deploy-to-kubernetes:
  stage: deploy
  image: mydeployimage:dev
  scripts:
    - kubectl apply -f kuberapp.yml
```
Данный стейдж должен выполняться раннером на образе специальном образе (в примере mydeployimage:dev), на котором должна быть зашита конфигурация утилиты kubectl, в 
которой будут зашиты ip и ключи доступа к мастер-ноде кластера Kubernetes, на который будет происходить деплой (данный образ просить у администратора кластера). Скрипт 
выполнит единственную команду – применение манифеста, развёртывающего приложение на кластере. Перед выполнением скрипта, в кластер должны быть загружены данные для 
авторизации в том docker registry, в котором будут храниться приватные образы для вашего приложения (образы обязательно должны быть приватными, использование Container 
Registry GitLab'a разрешается, но можно использовать и DockerHub). Ниже об этом будет написано подробнее.

## Создание манифеста развёртывания приложения на кластере

Прежде всего рекомендую ознакомиться с туториалом по Kubernetes в целом: https://www.youtube.com/playlist?list=PLg5SS_4L6LYvN1RqaVesof8KAf-02fJSi. Данный манифест 
описывает, в каком виде и с какой конфигурацией приложение будет развёрнуто в кластере. Для развёртывания будем использовать объект “deployment” (кроме него, есть ещё
другие типы объектов, подробнее: https://medium.com/stakater/k8s-deployments-vs-statefulsets-vs-daemonsets-60582f0c62d4). Подробный туториал по созданию манифеста для 
приложения на ASP.NET Core: https://habr.com/ru/post/709342/.
Пример манифеста приложения, состоящего из бэкэнда на ASP.NET Blazor и базы данных Postgres:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trans-cal-core
  namespace: cal-core
  labels:
    app: cal-core
    env: Production
    owner: OlegGubanov
spec:
  replicas: 1
  selector:
    matchLabels:
      project: tcc
  template:
    metadata:
      labels:
        project: tcc
    spec:
      containers:
        - name: cal-core-web
          image: mercenary95/cal-core:latest
          ports:
            - containerPort: 80
          env:
          - name: ASPNETCORE_ENVIRONMENT
            value: Production
      imagePullSecrets:
        - name: m95
---
apiVersion: v1
kind: Service
metadata:
  name: trans-cal-core-service
  namespace: cal-core
  labels:
    env: Production
    owner: OlegGubanov
spec:
  # replicas: 2
  selector:
    project: tcc
  ports:
    - name: app-listener
      protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tcc-postgres
  namespace: cal-core
spec:
  selector:
    matchLabels:
      app: tcc-postgres # has to match .spec.template.metadata.labels
  serviceName: "tcc-postgres"
  replicas: 1 # by default is 1
  minReadySeconds: 5 # by default is 0
  template:
    metadata:
      labels:
        app: tcc-postgres # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: tcc-postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: tcc-postgres
          mountPath: /var/lib/postgresql/data
          # subPath: data
        env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: password
        - name: POSTGRES_DB
          value: TransContCoreDB
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
      initContainers:
      - name: tcc-rights-to-data
        image: busybox
        command: ["sh","-c","mkdir -p /var/lib/postgresql/data/pgdata && chown -R 999:999 /var/lib/postgresql/data/pgdata"]
        securityContext:
          runAsUser: 0
          privileged: true
        volumeMounts:
        - name: tcc-postgres
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: tcc-postgres
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "rook-ceph-block"
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: tcc-postgres-service
  namespace: cal-core
  labels:
    app: tcc-postgres-service
spec:
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
  selector:
    app: tcc-postgres
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cal-core-ingress
  namespace: cal-core
spec:
  rules:
  - host: "tcc.stk8s.66bit.ru"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: trans-cal-core-service
            port:
              number: 80
```

Манифест определяет 6 объектов:  деплоймент для бэка, стейтфулсет для базы, и по сервису и ингресу для бэка и базы. Объекты должны разделяться с помощью ---. 
Подробности о типах объектов, о том, что означает каждая директива, есть в приведённом выше туториале по kubernetes на YouTube.
Также необходимо создать namespace и занести credentials для доступа к аккаунту вашего registry на кластер (в этот же namespace):

```
check:
  stage: deploy
  image: hub.66bit.ru/urfu2022/ci/deploy:1.26.2
  script:
    - kubectl create namespace cal-core
    - kubectl create secret generic m95 --from-file=.dockerconfigjson=$DOCKER_M95_CREDS --type=kubernetes.io/dockerconfigjson --namespace=cal-core
  when: manual
```
Этот stage нужно будет выполнить один раз, затем его можно удалить из CI файла.
Затем можно будет использовать кред в imagePullSecrets и ваш приватный образ будет доступен.

```
    spec:
      containers:
        - name: cal-core-web
          image: mercenary95/cal-core:latest
          ports:
            - containerPort: 80
          env:
          - name: ASPNETCORE_ENVIRONMENT
            value: Production
      imagePullSecrets:
        - name: m95
```

Подробнее: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

## Про применение манифеста

Выше уже упоминалось как применить манифест (то есть, по сути задеплоить приложение в kuber), это делается командой `kubectl apply -f kuberapp.yml`. Есть случаи, когда 
только применения манифеста не достаочно для полноценного CI-CD. Пример приведён ниже (приведённый выше манифест тоже является примером такого случая).

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: astrum-back
  namespace: astrum
  labels:
    app: astrum-back
    env: Production
    owner: MikhailBulatov
spec:
  replicas: 1
  selector:
    matchLabels:
      project: astrum-back
  template:
    metadata:
      labels:
        project: astrum-back
    spec:
      containers:
        - name: api
          image: *some-image*:latest
          ports:
            - containerPort: 80
          env:
          - name: ASPNETCORE_ENVIRONMENT
            value: Production
      imagePullSecrets:
        - name: astrum-secret
      dnsConfig:
        options:
          - name: ndots
            value: "2"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: astrum-postgres
  namespace: astrum
spec:
  selector:
    matchLabels:
      app: astrum-postgres # has to match .spec.template.metadata.labels
  serviceName: "astrum-postgres"
  replicas: 1 # by default is 1
  minReadySeconds: 5 # by default is 0
  template:
    metadata:
      labels:
        app: astrum-postgres # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: astrum-postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: astrum-postgres
          mountPath: /var/lib/postgresql/data
          # subPath: data
        env:
        - name: POSTGRES_USER
          value: user
        - name: POSTGRES_PASSWORD
          value: password
        - name: POSTGRES_DB
          value: db
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
      initContainers:
      - name: astrum-rights-to-data
        image: busybox
        command: ["sh","-c","mkdir -p /var/lib/postgresql/data/pgdata && chown -R 999:999 /var/lib/postgresql/data/pgdata"]
        securityContext:
          runAsUser: 0
          privileged: true
        volumeMounts:
        - name: astrum-postgres
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: astrum-postgres
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "rook-ceph-block"
      resources:
        requests:
          storage: 4Gi
---
apiVersion: v1
kind: Service
metadata:
  name: astrum-back-service
  namespace: astrum
  labels:
    env: Production
    owner: MikhailBulatov
spec:
  selector:
    project: astrum-back
  ports:
    - name: app-listener
      protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: astrum-postgres-service
  namespace: astrum
  labels:
    app: astrum-postgres-service
spec:
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
  selector:
    app: astrum-postgres
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: astrum-back-ingress
  namespace: astrum
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: 50m
    nginx.org/client-max-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  rules:
  - host: "*some host*"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: astrum-front-service
            port:
              number: 80
      - pathType: Prefix
        path: /api
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /admin
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /_framework
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /_content
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /Astrum.Api.styles.css
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /_blazor
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /adminStatic
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /graphql
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /swagger
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80

```
После первого применения манифеста, будут созданы соответсвтующие объекты. Теперь, для того, чтобы обновить объекты, должен быть обновлён манифест. В данном случае, 
обычно, при деплое будет обновляться Deployment astrum-back. К примеру разработчики написали новые фичи, залили их registry в образ `*some-image*:latest` и теперь нужно
пересоздать Deployment и переподгрузить приложение из этого образа. Проблема в том, что при вызове команды apply произойдёт проверка, поменялась ли в манифесте 
декларация этого объекта, и если она не поменялась, объект обновлен не будет. Если бы вместо `*some-image*:latest` мы явно писали новый тег каждый раз, к примеру 
`*some-image*:fa83aflgia`, где fa83aflgia был бы короткий хэш последнего комита, тогда при проверке кубер бы переподгружал новый образ, но так как у нас всегда тег 
latest, переподгрузки не произойдёт. Мы видим два варианта решения этой проблемы (если вы нашли ещё решение, пожалуйста напишите нам): 
1. Каждый раз записывать образ в уникальный тег (к примеру, хэш комита), тогда кубер будет каждый раз детектить обновление манифеста и переподгружать образ. Мы 
настоятельно не рекомендуем этот вариант, потому что при каждом создании нового тэга, старый остаётся, и это может серьёзно загрязнить хранилище если его не чистить 
вручную, или не отчищать автоматически старый тег при создании нового (если вы пользуетесь DockerHub, то можете выбрать этот вариант, нам не жалко места на серверах 
докерхаба :), но в registry гитлаба такое делать крайне не рекомендуем).
2. Удалять объект перед командой apply. Оптимальный вариант, по нашему мнению, поэтому мы и используем его в своих проектах. Для этого надо разделить манифест на два 
манифеста: манифест, содержащий объекты, которым не требуется каждый раз подгружать новые образы, и манифест с объектами по типу Deployment`а astrum-back из примера 
выше, то есть требующими подгрузки образа.

### Манифест с персистентными объектами cube-persistent-deployment.yaml:

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: astrum-postgres
  namespace: astrum
spec:
  selector:
    matchLabels:
      app: astrum-postgres # has to match .spec.template.metadata.labels
  serviceName: "astrum-postgres"
  replicas: 1 # by default is 1
  minReadySeconds: 5 # by default is 0
  template:
    metadata:
      labels:
        app: astrum-postgres # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: astrum-postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: astrum-postgres
          mountPath: /var/lib/postgresql/data
          # subPath: data
        env:
        - name: POSTGRES_USER
          value: user
        - name: POSTGRES_PASSWORD
          value: password
        - name: POSTGRES_DB
          value: db
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
      initContainers:
      - name: astrum-rights-to-data
        image: busybox
        command: ["sh","-c","mkdir -p /var/lib/postgresql/data/pgdata && chown -R 999:999 /var/lib/postgresql/data/pgdata"]
        securityContext:
          runAsUser: 0
          privileged: true
        volumeMounts:
        - name: astrum-postgres
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: astrum-postgres
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "rook-ceph-block"
      resources:
        requests:
          storage: 4Gi
---
apiVersion: v1
kind: Service
metadata:
  name: astrum-back-service
  namespace: astrum
  labels:
    env: Production
    owner: MikhailBulatov
spec:
  selector:
    project: astrum-back
  ports:
    - name: app-listener
      protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: astrum-postgres-service
  namespace: astrum
  labels:
    app: astrum-postgres-service
spec:
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
  selector:
    app: astrum-postgres
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: astrum-back-ingress
  namespace: astrum
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: 50m
    nginx.org/client-max-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  rules:
  - host: "*some host*"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: astrum-front-service
            port:
              number: 80
      - pathType: Prefix
        path: /api
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /admin
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /_framework
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /_content
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /Astrum.Api.styles.css
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /_blazor
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /adminStatic
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /graphql
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
      - pathType: Prefix
        path: /swagger
        backend:
          service:
            name: astrum-back-service
            port:
              number: 80
```

### Манифест с объектами, каждый раз требующих подгрузки образа cube-deployment.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: astrum-back
  namespace: astrum
  labels:
    app: astrum-back
    env: Production
    owner: MikhailBulatov
spec:
  replicas: 1
  selector:
    matchLabels:
      project: astrum-back
  template:
    metadata:
      labels:
        project: astrum-back
    spec:
      containers:
        - name: api
          image: *some-image*:latest
          ports:
            - containerPort: 80
          env:
          - name: ASPNETCORE_ENVIRONMENT
            value: Production
      imagePullSecrets:
        - name: astrum-secret
      dnsConfig:
        options:
          - name: ndots
            value: "2"
```

Тогда стейдж деплоя будет выглядеть так:

```
k8s_deploy_back:
  stage: deploy
  image: *образ для доступа к куберу*
  script:
    - kubectl delete -f cube-deployment.yaml
    - kubectl apply -f cube-persistent-deployment.yaml
    - kubectl apply -f cube-deployment.yaml
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```
При первом запуске нужно убрать `- kubectl delete -f cube-deployment.yaml`, так как кубер выдаст ошибку о том, что удаляемые объекты не существуют, и после первого 
запуска все объекты создадуться и нужно будет вернуть команду удаления неперсистентного манифеста.

Можно и не выделять персистентные объекты в отдельный манифест у удалять сразу все объекты и сразу все пересоздавать, но в таком подходе есть несколько минусов:
1. Это влечёт накладные расходы
2. Некоторые объекты, такие как базы данных, нельзя удалять, так как это повлечёт потерю всех данных, и как минимум базы точно стоит выносить в манифест для
персистентных объектов

## Про роутинг в Ingress

Обратите внимание на декларацию объекта astrum-back-ingress из примера выше. Для каждого пути прописан свой path, который ведёт к своему сервису, в частности 9 
специфических путей, ведущих на бэкенд и корневой путь ведущий на фронт (то есть все остальные пути будут вести на фронт). (Немного не удобно писать отдельный path для
каждого пути, поэтому если вы нашли способ писать несколько путей в одном path, (с помощью регулярных выражений как в nginx, к примеру) то пожалуйста напишите нам.

В прошлых версиях инструкции в примере манифеста писался Ingress для Postgres базы данных:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tcc-postgres-ingress
  namespace: cal-core
spec:
  rules:
  - host: "tcc-postgres.stk8s.66bit.ru"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: tcc-postgres-service
            port:
              number: 5432
```

Это была ошибка :), так делать нет смысла, потому что Ingress транслирует HTTP трафик, а не TCP (протокол, который использует Postgres) -
https://stackoverflow.com/questions/72400370/connect-to-psql-through-k8s-ingress
Для подключения к базе данных нужно использовать port forwarding кубера, но для этого вам нужен полный доступ к kubectl, пока мы не можем его предоставить :), поэтому
пока вы не сможете подключаться к базе извне (если всё таки очень нужно подключиться, то можете написать нам).
Пример port forwarding: `kubectl port-forward -n default svc/dev-pg-service 5432:5432`
