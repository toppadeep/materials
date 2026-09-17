# Kubernetes — практическое руководство

> Составлено на основе практики. Охватывает концепты, манифесты, команды и типовые сценарии.

---

## Содержание

1. [Зачем нужен Kubernetes](#1-зачем-нужен-kubernetes)
2. [Архитектура кластера](#2-архитектура-кластера)
3. [Основные объекты](#3-основные-объекты)
4. [Манифесты — шаблоны](#4-манифесты--шаблоны)
5. [ConfigMap и Secret](#5-configmap-и-secret)
6. [Rolling Update — деплой без даунтайма](#6-rolling-update--деплой-без-даунтайма)
7. [Ingress](#7-ingress)
8. [Полезные команды — шпаргалка](#8-полезные-команды--шпаргалка)
9. [Docker Compose vs Kubernetes](#9-docker-compose-vs-kubernetes)
10. [Типовые ошибки и их причины](#10-типовые-ошибки-и-их-причины)

---

## 1. Зачем нужен Kubernetes

**Docker Compose** — запускает контейнеры на одной машине.  
**Kubernetes** — запускает контейнеры на кластере машин и следит, чтобы всё работало.

### Проблемы, которые решает Kubernetes

| Проблема | Как решает |
|---|---|
| Сервер упал — всё умерло | Переносит поды на живые ноды |
| Нагрузка выросла — не справляемся | Автоскейлинг: добавляет копии подов |
| Обновление → даунтайм | Rolling update: заменяет поды по одному |
| 10 сервисов на 5 серверах | Единая система управления через `kubectl` |
| Контейнер упал | Self-healing: автоматически пересоздаёт под |

### Ключевая идея — декларативность

В Compose ты говоришь **«сделай»**.  
В Kubernetes ты говоришь **«должно быть так»** — и он сам добивается этого состояния.

```yaml
# Ты пишешь: "хочу 3 копии"
replicas: 3

# Kubernetes следит:
# упала 1 копия  → автоматически поднял новую
# нода умерла    → переехал на другую ноду
# ты написал 5   → добавил ещё 2
```

### Когда Kubernetes НЕ нужен

- Один сервер, небольшой проект → Docker Compose справится
- Нет команды, которая умеет его поддерживать → overhead огромный

---

## 2. Архитектура кластера

```
Cluster (весь Kubernetes)
├── Control Plane (мозг кластера)
│   ├── API Server        ← сюда идут все команды kubectl
│   ├── Scheduler         ← решает, на какой ноде запустить под
│   ├── etcd              ← база данных состояния кластера
│   └── Controller Manager← следит, что "желаемое" == "реальное"
│
└── Nodes (рабочие машины)
    ├── kubelet           ← агент на ноде, общается с Control Plane
    ├── kube-proxy        ← сетевые правила на ноде
    └── Container Runtime (containerd/Docker)
```

**Control Plane** — управляет. **Nodes** — исполняют.

На реальном кластере это разные машины. На локальном (Docker Desktop, minikube) — всё в одной VM.

---

## 3. Основные объекты

### Pod — минимальная единица

Один или несколько контейнеров, которые живут вместе: общий IP, общие volumes, общий lifecycle.

> Поды напрямую не создают — ими управляют Deployment/StatefulSet.

```
Pod: [контейнер app] + [контейнер sidecar-logger]
     └── один IP, общие volumes
```

---

### Deployment — управляет подами

Говоришь «хочу N копий этого пода» — Deployment следит за этим. Поддерживает rolling update и rollback.

```
Deployment "backend"
├── Pod backend-abc1  (нода 1)
├── Pod backend-abc2  (нода 2)
└── Pod backend-abc3  (нода 1)
```

---

### Service — стабильный адрес для подов

Поды умирают и пересоздаются, их IP меняются. Service — постоянная точка входа.

```
Запрос → Service "backend" → (балансирует) → Pod1, Pod2, Pod3
```

| Тип | Когда использовать |
|---|---|
| `ClusterIP` | Только внутри кластера (дефолт) |
| `NodePort` | Наружу через порт ноды (30000–32767) |
| `LoadBalancer` | Внешний балансировщик (облако: AWS, GCP, ...) |

---

### Ingress — HTTP-роутер

Один внешний IP/домен → роутит по path или hostname к разным сервисам.

```
example.com/api   →  Service "backend"
example.com/      →  Service "frontend"
```

Требует установленного **Ingress Controller** (например, ingress-nginx).

---

### ConfigMap — конфиги

Хранит конфигурацию отдельно от образа. Как `.env` файл, но в кластере.

---

### Secret — секреты

То же что ConfigMap, но для чувствительных данных (пароли, токены, ключи). Значения хранятся в base64.

---

### Namespace — логическое разделение

Изолирует ресурсы внутри одного кластера.

```bash
kubectl get pods -n production
kubectl get pods -n staging
```

---

## 4. Манифесты — шаблоны

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend          # Deployment управляет подами с этим лейблом
  template:
    metadata:
      labels:
        app: backend        # Pod получает этот лейбл
    spec:
      containers:
        - name: api
          image: my-api:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:         # берём из Secret
                  name: app-secrets
                  key: database-url
          resources:                  # лимиты ресурсов (хорошая практика)
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

---

### Service (ClusterIP — внутренний)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 8000          # порт сервиса внутри кластера
      targetPort: 8000    # порт контейнера
  type: ClusterIP
```

---

### Service (NodePort — наружу)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080     # диапазон 30000–32767
  type: NodePort
```

---

## 5. ConfigMap и Secret

### ConfigMap — для обычных конфигов

**Создать через yaml:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  API_BASE_URL: "http://backend:8000"
```

**Создать через команду:**

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info
```

**Использовать в Deployment:**

```yaml
# Вариант 1 — отдельные переменные
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV

# Вариант 2 — все ключи сразу
envFrom:
  - configMapRef:
      name: app-config
```

---

### Secret — для чувствительных данных

**Создать через команду (рекомендуется — не светить в git):**

```bash
kubectl create secret generic app-secrets \
  --from-literal=database-url="postgresql://user:pass@db:5432/mydb" \
  --from-literal=jwt-secret="supersecretkey"
```

**Создать через yaml (значения в base64):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  database-url: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0BkYjovbXlkYg==  # base64
  jwt-secret: c3VwZXJzZWNyZXRrZXk=
```

Закодировать в base64:
```bash
echo -n "mypassword" | base64
```

**Использовать в Deployment:**

```yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: database-url
```

**Проверить что есть:**

```bash
kubectl get secrets
kubectl get configmaps
kubectl describe secret app-secrets    # покажет ключи, но не значения
kubectl get secret app-secrets -o jsonpath='{.data.database-url}' | base64 -d
```

---

## 6. Rolling Update — деплой без даунтайма

Kubernetes по умолчанию делает rolling update: заменяет поды по одному, не убивая старые пока новые не готовы.

### Настройка стратегии в Deployment

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # сколько ДОПОЛНИТЕЛЬНЫХ подов можно создать во время обновления
      maxUnavailable: 0     # сколько подов можно убить до готовности новых (0 = zero downtime)
```

### Как задеплоить новую версию

```bash
# Обновить образ в deployment
kubectl set image deployment/backend api=my-api:2.0.0

# Или отредактировать yaml и применить
kubectl apply -f k8s/backend-deployment.yaml
```

### Следить за ходом деплоя

```bash
kubectl rollout status deployment/backend
# Waiting for deployment "backend" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "backend" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "backend" successfully rolled out
```

### Rollback — откат к предыдущей версии

```bash
# Посмотреть историю деплоев
kubectl rollout history deployment/backend

# Откатиться на предыдущую версию
kubectl rollout undo deployment/backend

# Откатиться на конкретную ревизию
kubectl rollout undo deployment/backend --to-revision=2
```

### Паузить и возобновлять деплой

```bash
kubectl rollout pause deployment/backend
kubectl rollout resume deployment/backend
```

---

## 7. Ingress

### Установить Ingress Controller (ingress-nginx)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml

# Ждать пока поднимется
kubectl get pods -n ingress-nginx -w
```

### Ingress манифест

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: example.com           # домен; для локала — localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 8000
```

### С TLS (HTTPS)

```yaml
spec:
  tls:
    - hosts:
        - example.com
      secretName: tls-secret     # Secret с сертификатом
  rules:
    - host: example.com
      ...
```

### Проверить

```bash
kubectl get ingress
kubectl describe ingress app-ingress
```

---

## 8. Полезные команды — шпаргалка

### Информация о ресурсах

```bash
# Посмотреть всё в текущем namespace
kubectl get all

# Поды
kubectl get pods
kubectl get pods -w                          # watch — обновляется в реальном времени
kubectl get pods -o wide                     # показывает ноду и IP
kubectl get pods -n kube-system              # поды системного namespace

# Deployments, Services, Ingress
kubectl get deployments
kubectl get services
kubectl get ingress
kubectl get configmaps
kubectl get secrets
kubectl get nodes                            # ноды кластера
```

### Детальная информация

```bash
kubectl describe pod <имя-пода>             # полная информация + события (Events)
kubectl describe deployment backend
kubectl describe service frontend
kubectl describe ingress app-ingress
```

> `describe` — первое что смотреть при проблемах, секция `Events` внизу

### Логи

```bash
kubectl logs <имя-пода>                     # логи пода
kubectl logs deployment/backend             # логи любого пода из деплоймента
kubectl logs -f deployment/backend          # follow — стримить логи в реальном времени
kubectl logs <имя-пода> --previous          # логи предыдущего (упавшего) пода
kubectl logs <имя-пода> -c <контейнер>      # конкретный контейнер в поде
```

### Выполнить команду в поде

```bash
kubectl exec -it <имя-пода> -- sh           # зайти внутрь (аналог docker exec)
kubectl exec -it <имя-пода> -- bash
kubectl exec deployment/backend -- env      # посмотреть переменные окружения
kubectl exec deployment/backend -- curl http://localhost:8000/health
```

### Применение и удаление манифестов

```bash
kubectl apply -f file.yaml                  # создать или обновить
kubectl apply -f k8s/                       # применить все yaml из папки
kubectl delete -f file.yaml                 # удалить ресурс
kubectl delete -f k8s/                      # удалить всё из папки

kubectl delete pod <имя-пода>               # удалить под (Deployment сразу пересоздаст)
kubectl delete deployment backend           # удалить deployment и все его поды
```

### Масштабирование

```bash
kubectl scale deployment/frontend --replicas=3
kubectl scale deployment/frontend --replicas=1
```

### Проброс портов (для отладки)

```bash
kubectl port-forward pod/<имя-пода> 8080:80
kubectl port-forward service/frontend 8080:80
kubectl port-forward deployment/backend 8000:8000
```

> Работает только пока команда запущена. Для дебага — удобнее NodePort.

### Деплой и rollback

```bash
kubectl set image deployment/backend api=my-api:2.0.0
kubectl rollout status deployment/backend
kubectl rollout history deployment/backend
kubectl rollout undo deployment/backend
kubectl rollout undo deployment/backend --to-revision=2
```

### Работа с namespace

```bash
kubectl get namespaces
kubectl create namespace staging
kubectl apply -f k8s/ -n staging            # применить в конкретный namespace
kubectl get pods -n staging
kubectl config set-context --current --namespace=production  # переключить дефолтный ns
```

### Контекст (если несколько кластеров)

```bash
kubectl config get-contexts                 # список кластеров
kubectl config use-context <имя>            # переключиться на другой кластер
kubectl config current-context             # текущий кластер
```

### Полезные флаги

```bash
-o wide           # расширенный вывод
-o yaml           # вывести объект в yaml (посмотреть что реально применено)
-o json           # вывести объект в json
-w                # watch — обновляться при изменениях
-n <namespace>    # указать namespace
--all-namespaces  # все namespaces сразу
--dry-run=client  # проверить yaml без применения
```

---

## 9. Docker Compose vs Kubernetes

| Compose | Kubernetes |
|---|---|
| `service` | `Deployment` + `Service` |
| `image` | `image` в spec контейнера |
| `ports` | `Service` (NodePort / LoadBalancer) |
| `environment` | `ConfigMap` / `Secret` |
| `volumes` | `PersistentVolume` / `PVC` |
| `networks` | встроено, работает через DNS |
| `replicas` | `replicas` в Deployment |
| `restart: unless-stopped` | Deployment по умолчанию пересоздаёт поды |
| `depends_on` | через readiness probe или init containers |
| `docker-compose up` | `kubectl apply -f k8s/` |
| `docker-compose down` | `kubectl delete -f k8s/` |
| `docker-compose logs` | `kubectl logs deployment/<name>` |

### Как сервисы находят друг друга

В Compose — по имени сервиса через `app-network`.  
В Kubernetes — то же самое, но через встроенный DNS кластера.

```
# В nginx.conf:
proxy_pass http://backend:8000;
# "backend" резолвится в IP Service через DNS кластера — автоматически.
```

---

## 10. Типовые ошибки и их причины

### ErrImageNeverPull

```
Reason: ErrImageNeverPull
Message: Container image "my-api:latest" is not present with pull policy of Never
```

**Причина:** `imagePullPolicy: Never` означает «только из локального store containerd». Образ есть в Docker, но не виден containerd.  
**Решение:** поменять на `imagePullPolicy: IfNotPresent`.

---

### ImagePullBackOff / ErrImagePull

```
Status: ImagePullBackOff
```

**Причина:** не может скачать образ из registry (неверное имя, нет доступа, не авторизован).  
**Диагностика:**
```bash
kubectl describe pod <имя-пода>   # смотреть Events внизу
```

---

### CrashLoopBackOff

```
Status: CrashLoopBackOff
RESTARTS: 5
```

**Причина:** контейнер падает сразу после запуска, Kubernetes пытается перезапустить с экспоненциальной задержкой.  
**Диагностика:**
```bash
kubectl logs <имя-пода> --previous   # логи предыдущего запуска
kubectl describe pod <имя-пода>
```

---

### Pending (под не запускается)

```
Status: Pending
```

**Причины:**
- Нет ноды с достаточными ресурсами
- Нет подходящей ноды по `nodeSelector`
- Не создан PVC (если указан volume)

**Диагностика:**
```bash
kubectl describe pod <имя-пода>   # секция Events покажет причину
```

---

### connection refused при обращении к сервису

**Диагностика:**
```bash
# Проверить что поды живы
kubectl get pods

# Проверить что selector в Service совпадает с labels пода
kubectl describe service backend
kubectl describe pod backend-xxx

# Зайти в под и попробовать изнутри
kubectl exec -it <frontend-pod> -- curl http://backend:8000
```

---

### Manifest list / multi-platform образ не подхватывается

**Симптом:** образ собран, но Kubernetes не видит его.  
**Причина:** BuildKit создал manifest list (multi-arch), containerd не умеет его резолвить локально.  
**Решение:**
```bash
docker build --provenance=false -t my-api:latest ./api
```

---

## Быстрый старт — чеклист деплоя

```bash
# 1. Собрать образы
docker build --provenance=false -t my-api:1.0.0 ./api
docker build --provenance=false -t my-web:1.0.0 ./web

# 2. Создать секреты (до apply манифестов)
kubectl create secret generic app-secrets \
  --from-literal=database-url="postgresql://..."

# 3. Применить манифесты
kubectl apply -f k8s/

# 4. Проверить статус
kubectl get pods -w

# 5. Посмотреть логи если что-то не так
kubectl logs deployment/backend
kubectl describe pod <имя-пода>

# 6. Проверить доступность
kubectl port-forward service/frontend 8080:80
curl http://localhost:8080

# 7. Задеплоить новую версию
docker build --provenance=false -t my-api:1.1.0 ./api
kubectl set image deployment/backend api=my-api:1.1.0
kubectl rollout status deployment/backend

# 8. Если что-то пошло не так — откат
kubectl rollout undo deployment/backend
```
