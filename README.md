# llabdocs-helm

Helm-чарт сервиса **LLabDocs** (Spring Boot приложение подписания документов) + сопутствующие
ресурсы:

- deployment/service/configmap/PVC самого приложения (образ `lichbalab/llabdocs-images`);
- Keycloak + PostgreSQL с TLS (опционально, `keycloak.enabled`);
- нативные Ingress-ресурсы (`ingressClassName: nginx` — чарт только декларирует, контроллер вне чарта);
- cert-manager `ClusterIssuer` + `Certificate keycloak-tls` для выпуска TLS-сертификатов.

**Ingress-nginx controller в чарт не входит** (упрощено): он ставится один раз на кластер,
а чарт создаёт только `Ingress`-ресурсы. TLS keycloak обеспечивает cert-manager
(`Certificate` → Secret `keycloak-tls`), а не ingress-nginx.

## Как устроен деплой

Деплой идёт через **ArgoCD v3**. Классический тип «Helm chart из Helm-репозитория» в ArgoCD v3
отсутствует, поэтому чарт **рендерится в нативные Kubernetes-манифесты** локально (`helm template`),
результат коммитится в приватный git-репозиторий, а ArgoCD **Application** применяет этот репозиторий
в целевой namespace.

Поток данных:

```
charts/llabdocs (этот репозиторий)
        │  helm template -f overrides.yaml
        ▼
llabdocs-deployment.git  (приватный репозиторий с отрендеренными манифестами)
        │  ArgoCD Application "app3" (тип Git, ветка keycloak-integration)
        ▼
namespace "backend"  (Deployment, Service, Ingress, PVC, Secret, Certificate, ClusterIssuer)
```

Рабочая инсталляция на момент написания: ArgoCD `v3.5.3`, локальный кластер
`docker-desktop` (context `kubectl` — `docker-desktop`), приложение `app3`
(deployment `llabdocs-deployment`, keycloak, postgres), хосты
`llabdocs.tech.mc` и `keycloak.llabdocs.mc`.

---

## Шаг 1. Установка ArgoCD

Официальный manifest `install.yaml` ставит в namespace `argocd` все компоненты:
`argocd-server`, `argocd-application-controller`, `argocd-repo-server`, `argocd-redis`,
а в v3 ещё `argocd-dex-server`, `argocd-notifications-controller`, `argocd-applicationset-controller`.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# проверить, что всё поднялось
kubectl -n argocd get pods
```

### Доступ к UI и первый вход

```bash
# проброс порта (service argocd-server слушает 443)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# первичный пароль (username: admin)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Открыть https://localhost:8080 → войти `admin` / полученный пароль → сменить пароль.

CLI (по желанию):

```bash
brew install argocd
argocd login https://localhost:8080 --username admin   # после порт-форварда
```

> **Известная проблема:** `argocd-applicationset-controller` крутится в `CrashLoopBackOff`
> (`no matches for kind "ApplicationSet"` — CRD не установлен). Он нужен только для
> ресурсов `ApplicationSet`, которых в этом проекте нет, поэтому его можно отключить:
> ```bash
> kubectl -n argocd patch deployment argocd-applicationset-controller -p '{"spec":{"replicas":0}}'
> ```

## Шаг 2. Установка cert-manager

Нужен в двух местах (контроллер ingress-nginx для этого не требуется):

- **TLS keycloak** — `Certificate keycloak-tls` (самоподписанный, issuer `ClusterIssuer llabdocs-local`) → секрет `keycloak-tls`, который монтируется в под keycloak (`/etc/x509/https`);
- **на прод-кластере** — TLS для нативных Ingress через аннотацию `cert-manager.io/cluster-issuer`.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.yaml
kubectl -n cert-manager get pods   # cert-manager / cainjector / webhook
```

## Шаг 3. Файл значений (values overlay)

Значения по умолчанию — `charts/llabdocs/values.yaml`. Для реального деплоя делается
оверлей (файл не в этом репозитории — он уходит вместе с рендером). Пример, соответствующий
рабочей инсталляции:

```yaml
# deploy-overrides.yaml
namespace: backend
image:
  repository: docker.io/lichbalab/llabdocs-images
  tag: build-2024.1-SNAPSHOT-68a7171   # конкретный билд из реестра
  pullPolicy: Always
ingress:
  hosts:
    - host: llabdocs.tech.mc
      paths:
        - path: /
          pathType: Prefix
keycloak:
  enabled: true
  adminUser: admin
  adminPassword: "<сменить>"
  hostname: keycloak.llabdocs.mc
  ingress:
    tls:
      - secretName: keycloak-tls
        hosts: [keycloak.llabdocs.mc]
certManager:
  clusterIssuer:
    enabled: true
    name: llabdocs-local          # selfsigned, см. values.yaml
```

### Оверлей для локальной разработки

На локальном кластере ingress-контроллера нет, поэтому ингрессы отключаем, а доступ идёт
напрямую через port-forward. TLS keycloak при этом продолжает работать (cert-manager):

```yaml
# dev-overrides.yaml
namespace: backend
ingress:
  enabled: false          # некому обработать — контроллера нет
keycloak:
  ingress:
    enabled: false
  certificate:
    enabled: true         # TLS keycloak остаётся (по умолчанию уже true)
```

```bash
kubectl -n backend port-forward svc/llabdocs-service 8080:8080   # http://localhost:8080/actuator/health
kubectl -n backend port-forward svc/keycloak 8443:8443           # https://localhost:8443
```

Так как в манифестах нет LoadBalancer-сервиса и нативных Ingress, в ArgoCD приложение
в dev-конфигурации становится **Healthy** (см. «Про статус приложения» в Шаге 7).

> **Секрет `do-registry`:** в `values.yaml` стоит `image.pullSecrets: do-registry`.
> Секрет нужен только если образ в реестре приватный. Сейчас образ публичный, поэтому
> деплой проходит и без создания секрета. Если реестр станет приватным:
> ```bash
> kubectl create secret generic do-registry \
>   --from-file=.dockerconfigjson=docker-config.json \
>   --type=kubernetes.io/dockerconfigjson \
>   --namespace backend
> ```

## Шаг 4. Рендер чарта и публикация манифестов в git

ArgoCD применяет нативный YAML, поэтому сначала рендерим чарт, затем коммитим результат
в приватный репозиторий `github.com/lichbalab/llabdocs-deployment`
(рабочая ветка — `keycloak-integration`).

```bash
# 1. отрендерить чарт в файлы (имя release = app3, как в рабочей инсталляции)
helm template app3 charts/llabdocs \
  -f deploy-overrides.yaml \
  --api-versions networking.k8s.io/v1 --api-versions cert-manager.io/v1 \
  --output-dir rendered/

# 2. скопировать результат в локальный клон llabdocs-deployment и запушить
cd ../llabdocs-deployment
cp -R ../llabdocs-helm/rendered/app3/* .
git add -A && git commit -m "deploy: render llabdocs chart v1.9-helm"
git push origin keycloak-integration
```

Зависимостей у чарта сейчас нет (контроллер вынесен из чарта). Если их вернут в
`Chart.yaml` — перед рендером понадобится `helm dependency update charts/llabdocs`
(`Chart.lock` и `charts/llabdocs/charts/*.tgz` в git не попадают — см. `.gitignore`).

## Шаг 5. Регистрация git-репозитория в ArgoCD

Репозиторий приватный — ArgoCD нужно дать доступ:

- **UI:** Settings → Repositories → Add Repository: тип **Git**, URL
  `https://github.com/lichbalab/llabdocs-deployment.git`, тип авторизации
  **Personal Access Token → GitHub** (токен `repo`), username.
- Для SSH используют deploy-key в секрете `argocd-git` (Settings → Repositories → SSH Private Key).

## Шаг 6. Создание Application в ArgoCD

Чарт деплоится как приложение **типа Git** (`argo cd::application`):

- **UI:** Applications → Create Application:
  - Name: `app3`
  - Source Type: **Git**
  - Repository: `https://github.com/lichbalab/llabdocs-deployment.git`
  - Revision Type: **Branch** → `keycloak-integration` (либо конкретный commit/release-tag)
  - Path: `.`
  - Namespace: `backend`
  - Sync Policy → Sync Options: **Create Namespace** (включить)
- **ArgoCD Resources** → Application → будет создан ресурс `app3` в namespace `argocd`.

Созданный ресурс можно посмотреть напрямую:

```bash
kubectl -n argocd get application app3 -o yaml     # spec.source / history / operationState
kubectl -n argocd get application app3 -o jsonpath='{.status.operationState.phase}'  # Running/...
```

## Шаг 7. Запуск и проверка

После создания приложение синхронизируется автоматически. Проверяем результат:

```bash
# всё, что создал чарт в namespace backend
kubectl -n backend get all

# конкретно
kubectl -n backend get pods
kubectl -n backend get svc llabdocs-service
kubectl -n backend get ing            # ingress-llabdocs, ingress-keycloak (если включены)
kubectl -n backend get certificates   # keycloak-tls (selfsigned)
kubectl -n backend get secrets        # letsencrypt-nginx-llabdocs, keycloak-tls, keycloak-secret
kubectl get clusterissuers            # llabdocs-local (selfsigned)
```

Логи приложений:

```bash
kubectl -n backend logs deployment/llabdocs-deployment
kubectl -n backend exec -it deployment/llabdocs-deployment -- tail -f /logs/application.log
```

Проверка приложения:

```bash
kubectl -n backend exec -it deployment/llabdocs-deployment -- curl -sk https://127.0.0.1:8080/actuator/health
```

> **Про статус приложения:** при рендере с `ingress.enabled: true` (прод-ориентированные
> значения) ArgoCD на локальном кластере будет бесконечно «синхронизироваться»: он ждёт
> здоровья `LoadBalancer`-сервиса и нативных Ingress, а на `docker-desktop` внешнего IP нет,
> и ингресс некому обработать. Если поды `1/1 Running` — деплой прошёл. В dev-конфигурации
> (ингресс выключен, см. Шаг 3) таких ресурсов в манифестах нет, и приложение получает
> статус **Healthy**.

## Шаг 8. Обновление приложения

Изменяем чарт/оверлей → повторяем Шаг 4 (рендер + push) → в ArgoCD
Applications → app3 → Sync (или `kubectl -n argocd get application app3` → статус обновится сам).
ArgoCD сам заметит новый коммит ветки только по ручному Sync либо по включённому webhook
(настроить в Settings → Repositories → Webhook).

Полезная команда для отладки рендера:

```bash
helm template llabdocs charts/llabdocs -f deploy-overrides.yaml --debug
```

---

## Производственный кластер (DigitalOcean)

Справочные команды, использовавшиеся при деплое на DO Kubernetes
(на локальном `docker-desktop` не имеют эффекта).

Ingress-nginx controller ставится на DO **один раз** на кластер (в чарт он не входит):

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace

kubectl -n ingress-nginx get svc   # контроллер поднялся и получил внешний IP
```

```bash
# параметры Load Balancer и сертификаты DO
doctl compute load-balancer list
doctl compute load-balancer get <lb-id>
doctl compute certificate list

# ingress-nginx v1.x: разрешить snippet-аннотации (для кастомных путей/редиректов)
kubectl describe configmap ingress-nginx-controller -n ingress-nginx
kubectl patch configmap ingress-nginx-controller -n ingress-nginx \
  --patch '{"data":{"allow-snippet-annotations":"true"}}'

# K8s dashboard (порт-форвард + токен администратора)
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
kubectl apply -f dashboard-adminuser.yaml
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d
```