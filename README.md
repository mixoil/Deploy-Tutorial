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
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "docker login -u "$CI_REGISTRY_FRONT_USER" -p "$CI_REGISTRY_FRONT_PASSWORD" $CI_REGISTRY"
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "cd /app/ && docker-compose -f docker-compose.dev.yml pull frontend"
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "docker logout $CI_REGISTRY"
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY"
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "cd /app/ && docker-compose -f docker-compose.dev.yml pull api && docker-compose -f docker-compose.dev.yml up -d"
    after_script:
      - ssh -i ssh_private_key -o StrictHostKeyChecking=no $DEPLOY_HOST_LOGIN@$DEPLOY_HOST_IP "docker logout $CI_REGISTRY"
    when: manual
    only:
      - main
    environment:
      name: dev
```
В данном случае, интеграция и публикация реализованы в одном стейдже – build. Далее происходит деплой приложения с помощью docker-compose.dev.yml который разворачивает докер контейнеры на указанном сервере (его ip подставляется из переменной $DEPLOY_HOST_IP) . Все переменные типа $DEPLOY_HOST_IP спрятаны в хранилище секретов gitlab репозитория. Вы можете основывать свою конфигурацию CD на этом стейдже, но если вы хотите развернуть приложение в Kubernetes, ваша конфигурация стейджа будет выглядеть примерно следующим образом:
```
stages:
  - deploy

deploy-to-kubernetes:
  stage: deploy
  image: mydeployimage:dev
  scripts:
    - kubectl apply -f kuberapp.yml``
```
Данный стейдж должен выполняться раннером на образе специальном образе (в примере mydeployimage:dev), на котором должна быть зашита конфигурация утилиты kubectl, в которой будут зашиты ip и ключи доступа к мастер-ноде кластера Kubernetes, на который будет происходить деплой (данный образ просить у администратора кластера). Скрипт выполнит единственную команду – применение манифеста, развёртывающего приложение на кластере. Перед выполнением скрипта, в кластер должны быть загружены данные для авторизации в том docker registry, в котором будут храниться приватные образы для вашего приложения (образы обязательно должны быть приватными). Инструкция об этом выйдет чуть позже.

## Создание манифеста развёртывания приложения на кластере
Прежде всего рекомендую ознакомиться с туториалом по Kubernetes в целом: https://www.youtube.com/playlist?list=PLg5SS_4L6LYvN1RqaVesof8KAf-02fJSi. Данный манифест описывает, в каком виде и с какой конфигурацией приложение будет развёрнуто в кластере. Для развёртывания будем использовать объект “deployment” (кроме него, есть ещё другие типы объектов, подробнее: https://medium.com/stakater/k8s-deployments-vs-statefulsets-vs-daemonsets-60582f0c62d4). Подробный туториал по созданию манифеста для приложения на ASP.NET Core: https://habr.com/ru/post/709342/.
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
---
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

Манифест определяет 6 объектов:  деплоймент для бэка, стейтфулсет для базы, и по сервису и ингресу для бэка и базы. Объекты должны разделяться с помощью ---. Подробности о типах объектов, о том, что означает каждая директива, есть в приведённом выше туториале по kubernetes на YouTube.
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
