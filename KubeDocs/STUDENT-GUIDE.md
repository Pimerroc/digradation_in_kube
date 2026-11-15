# Работа в stud-cluster: инструкция студенту

Актуально на 13 ноября 2026 года. Каждый студент работает в своём
`itmo-student-XX`, где `XX` — номер от `01` до `30`.
Для `digradation` используются те же приёмы; его подключение описано в отдельном README.

## 1. Подключение и права

Получите у преподавателя свой архив `itmo-student-XX.zip`, распакуйте его.
В нём находятся kubeconfig, `hello.yaml`, `postgres.yaml`, README с вашими
конкретными адресами и эта инструкция. Установите `kubectl`; для SQL с компьютера
также понадобится `psql`.

Пример для студента 01, macOS / Linux:

```bash
cd /путь/к/распакованному/архиву
chmod 600 itmo-student-01.kubeconfig
export KUBECONFIG="$PWD/itmo-student-01.kubeconfig"
kubectl config current-context
kubectl get pods
```

Для своего номера замените имя файла. В PowerShell:

```powershell
$env:KUBECONFIG = (Resolve-Path .\itmo-student-01.kubeconfig).Path
kubectl get pods
```

Cloud.ru, cloudlogin и SSH для этого подключения не нужны.
Если приложение просит kubeconfig, выберите свой файл и одноимённый контекст.
Права распространяются на свой namespace, включая Deployments, Services,
Secrets, PVC, HTTPRoute, TCPRoute, Certificate, Issuer и PostgreSQL Cluster.
Поды, Secrets и БД других студентов недоступны. Кластерные операторы уже установлены.

Следующие команды рассчитаны на bash/zsh. Определите namespace из kubeconfig:

```bash
export NS="$(kubectl config view --minify -o jsonpath='{.contexts[0].context.namespace}')"
export LISTENER="${NS#itmo-}"
printf 'namespace=%s listener=%s\n' "$NS" "$LISTENER"
kubectl get resourcequota student-budget
kubectl get limitrange container-defaults
```

`LISTENER` будет `student-01` для `itmo-student-01` и `digradation` для `digradation`.
Не меняйте namespace в чужой: это не добавит прав. Храните kubeconfig и пароли
приватно, не добавляйте их в Git.

## 2. Что доступно

| Сервис | Что создаёт студент | Где находится общая часть |
|---|---|---|
| Gateway API | Service и HTTPRoute; для публичного PostgreSQL студентов — TCPRoute | Gateway `envoy-gateway-system/itmo-shared`, ListenerSet `envoy-gateway-system/itmo-https` |
| cert-manager | Для обычного публичного HTTPS достаточно HTTPRoute; для лабораторного внутреннего TLS можно создать свой Issuer и Certificate | Публичные сертификаты Let's Encrypt и DNS-проверки обслуживаются преподавателем |
| CloudNativePG | `Cluster` в API `postgresql.cnpg.io/v1`, затем приложение и подключение к БД | Общий оператор в `cnpg-system` |

Квота каждого namespace: **requests и limits суммарно по 2.6 CPU и 4 GB RAM**
(4 000 000 000 байт). Это общий бюджет всех контейнеров namespace, включая БД.
Если ресурсы не указаны, LimitRange добавит requests `100m/128Mi` и limits
`500m/512Mi` на контейнер. Минимум — `10m CPU / 1Mi RAM`.
Квоту и LimitRange меняет администратор.

Используйте Services типа `ClusterIP`. Внешний доступ предоставляется через общий
Gateway; отдельные LoadBalancer и NodePort в своём namespace запрещены квотой.
Pod Security baseline запрещает привилегированные контейнеры и доступ к хосту.
Входящие соединения по умолчанию разрешены из своего namespace, Gateway и оператора
PostgreSQL. При изменении NetworkPolicy сохраняйте эти необходимые подключения.

## 3. Быстрый запуск приложения

Из своего персонального архива:

```bash
kubectl apply -f hello.yaml
kubectl rollout status deployment/hello --timeout=180s
curl "https://app.${NS}.mlops.goctf.online/"
```

Ожидается страница nginx. Файл создаёт Deployment `hello`, Service `hello:80`
и HTTPRoute `hello`. Приложение доступно по HTTP :80 и HTTPS :443 на общем
публичном IP `37.18.117.114`.

Для своих приложений используйте имена вида:

```text
api.itmo-student-01.mlops.goctf.online
frontend.itmo-student-01.mlops.goctf.online
app.digradation.mlops.goctf.online
```

Каждый `<app>` — один DNS-компонент, без дополнительных точек.
Например, сертификат `*.itmo-student-01.mlops.goctf.online` покрывает
`api.itmo-student-01.mlops.goctf.online`.
Путь `/student-01` к URL добавлять не требуется: разделение выполняется по имени хоста.

## 4. Gateway API для своего приложения

Сначала разверните приложение и Service в своём namespace. Service должен
выбирать работающие Pod-ы, а его `targetPort` — соответствовать порту приложения.
Затем создайте маршрут. Этот пример публикует существующий Service `hello` на порту 80:

```bash
cat > route.yaml <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: ${NS}
spec:
  parentRefs:
    - name: itmo-shared
      namespace: envoy-gateway-system
      sectionName: http-${LISTENER}
    - name: itmo-https
      kind: ListenerSet
      group: gateway.networking.k8s.io
      namespace: envoy-gateway-system
      sectionName: https-${LISTENER}
  hostnames:
    - my-app.${NS}.mlops.goctf.online
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: hello
          port: 80
EOF
kubectl apply -f route.yaml
curl "https://my-app.${NS}.mlops.goctf.online/"
```

В `backendRefs.port` указывается **порт Service**. Для другого приложения
замените `backendRefs.name`, порт и первый компонент hostname. В примере две
parentRefs: HTTP направлен в Gateway, HTTPS — в ListenerSet. Если нужен только
HTTPS, оставьте вторую parentRef.

Не создавайте свой Gateway и не меняйте общий. Используйте слушатели своего
namespace: присоединение к слушателю другого студента будет отклонено.
Один hostname можно разделить между сервисами по путям `/api` и `/` через
несколько правил HTTPRoute. По умолчанию префикс пути передаётся приложению;
если его нужно убрать, задайте фильтр `URLRewrite`.

Проверка маршрута:

```bash
kubectl get httproute
kubectl get httproute my-app -o yaml
kubectl get service hello
kubectl get endpointslice -l kubernetes.io/service-name=hello
kubectl get gateway itmo-shared -n envoy-gateway-system
kubectl get listenerset itmo-https -n envoy-gateway-system
```

В `.status.parents[].conditions` ожидаются `Accepted=True` и `ResolvedRefs=True`.
Общие Gateway/ListenerSet доступны для чтения по этим конкретным именам;
запрос списка всех ресурсов инфраструктурного namespace может быть запрещён.
Подробности полей: [HTTPRoute](https://gateway-api.sigs.k8s.io/reference/api-types/httproute/)
и [ListenerSet](https://gateway-api.sigs.k8s.io/reference/api-types/listenerset/).

## 5. cert-manager: публичный и внутренний TLS

Для публичных адресов `<app>.<ваш-namespace>.mlops.goctf.online` сертификат
Let's Encrypt уже подключён к HTTPS-слушателю. cert-manager продлевает его
автоматически. Студенту достаточно создать HTTPRoute; выпускать Certificate
для каждого приложения или добавлять аннотацию к HTTPRoute не требуется.
Приложение за Gateway может слушать обычный HTTP, например Service `hello:80`.

Общий TLS Secret и DNS-ключи находятся в инфраструктурных namespace и
студентам недоступны. `letsencrypt-production` — namespaced Issuer в
`envoy-gateway-system`, его нельзя использовать в Certificate своего namespace.
Собственный домен за пределами выделенного шаблона согласуйте с преподавателем.

Если лабораторная требует изучить ресурсы cert-manager, можно выпустить
**самоподписанный тестовый сертификат внутри своего namespace**:

```bash
cat > internal-tls.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: lab-selfsigned
  namespace: ${NS}
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: hello-internal
  namespace: ${NS}
spec:
  secretName: hello-internal-tls
  commonName: hello.${NS}.svc.cluster.local
  dnsNames:
    - hello
    - hello.${NS}.svc.cluster.local
  duration: 168h
  renewBefore: 24h
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
  issuerRef:
    name: lab-selfsigned
    kind: Issuer
EOF
kubectl apply -f internal-tls.yaml
kubectl wait certificate/hello-internal --for=condition=Ready --timeout=120s
kubectl get certificate,issuer
kubectl describe certificate hello-internal
kubectl get secret hello-internal-tls
```

Secret содержит `tls.crt` и `tls.key`. Его нужно отдельно смонтировать в приложение
и настроить TLS в самом приложении. Создание Certificate само по себе не меняет
Service или nginx. Самоподписанный сертификат предназначен для лабораторных
проверок с явно настроенным доверием клиента; публичный HTTPS Gateway использует
свой сертификат Let's Encrypt.
Справка: [Certificate](https://cert-manager.io/docs/usage/certificate/),
[SelfSigned Issuer](https://cert-manager.io/docs/configuration/selfsigned/).

## 6. PostgreSQL через CloudNativePG

Файл `postgres.yaml` из архива создаёт одноэкземплярную учебную БД:

```bash
kubectl apply -f postgres.yaml
kubectl wait cluster.postgresql.cnpg.io/lab-db --for=condition=Ready --timeout=10m
kubectl get clusters.postgresql.cnpg.io
kubectl get pods -l cnpg.io/cluster=lab-db
kubectl get pvc
kubectl get service lab-db-rw
```

Основные параметры этого Cluster:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: lab-db
spec:
  instances: 1
  imageName: ghcr.io/cloudnative-pg/postgresql:18.4-minimal-trixie
  bootstrap:
    initdb:
      database: app
      owner: app
  storage:
    size: 5Gi
    storageClass: cloudru-ssd
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  postgresql:
    parameters:
      shared_buffers: 128MB
      max_connections: '50'
    pg_hba:
      - hostnossl all all all reject
```

Применяйте готовый файл из своего архива, где уже указан ваш namespace.
`instances: 1` подходит для лабораторной; это одна PostgreSQL-инстанция без
реплики для отказоустойчивости. Оператор создаёт Pod, PVC, Services и Secrets.
База — `app`, пользователь — `app`, пароль находится в `lab-db-app`.
Внутри namespace подключайтесь к `lab-db-rw:5432`; это Service текущего primary.
Пример требует TLS и отвергает нешифрованные PostgreSQL-соединения.

### Подключение с компьютера через port-forward

Способ подходит всем namespace, включая `digradation`. В первом терминале:

```bash
kubectl port-forward service/lab-db-rw 15432:5432
```

Во втором терминале задайте тот же `KUBECONFIG`, затем:

```bash
export PGPASSWORD="$(kubectl get secret lab-db-app -o jsonpath='{.data.password}' | base64 --decode)"
kubectl get secret lab-db-ca -o jsonpath='{.data.ca\.crt}' | base64 --decode > lab-db-ca.crt
psql "host=lab-db-rw hostaddr=127.0.0.1 port=15432 dbname=app user=app sslmode=verify-full sslrootcert=$PWD/lab-db-ca.crt"
```

`host` используется при проверке имени в сертификате БД, `hostaddr` — для
TCP-подключения к локальному port-forward. В psql:

```sql
SELECT current_database(), current_user;
SELECT ssl, version FROM pg_stat_ssl WHERE pid = pg_backend_pid();
CREATE TABLE lab_notes (id integer PRIMARY KEY, note text);
INSERT INTO lab_notes VALUES (1, 'PostgreSQL работает');
SELECT * FROM lab_notes;
\q
```

После работы выполните `unset PGPASSWORD`. Соединение доступно, пока открыт
терминал с port-forward; для остановки нажмите Ctrl+C.

### Публичное подключение студентов 01–30

В студенческом `postgres.yaml` также есть TCPRoute к `lab-db-rw:5432`.
Публичный IP — `37.18.117.114`; порт равен **15400 + номер студента**:
01 → 15401, 02 → 15402, …, 30 → 15430.
После применения файла подключитесь с паролем и CA, полученными выше:

```bash
export PGPASSWORD="$(kubectl get secret lab-db-app -o jsonpath='{.data.password}' | base64 --decode)"
NUMBER="${NS##*-}"
PGPORT_PUBLIC=$((15400 + 10#$NUMBER))
psql "host=lab-db-rw hostaddr=37.18.117.114 port=$PGPORT_PUBLIC dbname=app user=app sslmode=verify-full sslrootcert=$PWD/lab-db-ca.crt"
unset PGPASSWORD
```

Эта команда расчёта порта предназначена для `itmo-student-XX`.
Для `digradation` используйте port-forward из предыдущего раздела.
Для одного студента публичный порт один; чтобы вывести другую БД, измените
свой TCPRoute. Внутри namespace можно создавать несколько PostgreSQL Cluster,
учитывая общий бюджет CPU/RAM и расходы на диски.

### Подключение приложения к БД

Добавьте в контейнер приложения переменные и том с CA. Пароль берите через
`secretKeyRef`; не записывайте его в YAML открытым текстом:

```yaml
env:
  - name: PGHOST
    value: lab-db-rw
  - name: PGPORT
    value: '5432'
  - name: PGDATABASE
    value: app
  - name: PGUSER
    valueFrom:
      secretKeyRef:
        name: lab-db-app
        key: username
  - name: PGPASSWORD
    valueFrom:
      secretKeyRef:
        name: lab-db-app
        key: password
  - name: PGSSLMODE
    value: verify-full
  - name: PGSSLROOTCERT
    value: /etc/postgresql-ca/ca.crt
volumeMounts:
  - name: postgresql-ca
    mountPath: /etc/postgresql-ca
    readOnly: true
```

А на уровне `spec.template.spec` Deployment добавьте:

```yaml
volumes:
  - name: postgresql-ca
    secret:
      secretName: lab-db-ca
```

Клиентская библиотека должна поддерживать эти libpq-переменные (например,
psycopg). Для другого драйвера передайте те же host, database, user, password,
CA и режим проверки TLS через его параметры подключения.
Документация: [CloudNativePG: приложения](https://cloudnative-pg.io/docs/current/applications/).

## 7. Диагностика

| Симптом | Что проверить |
|---|---|
| `Forbidden` | Свой kubeconfig и текущий namespace; доступ к общим ресурсам выдаётся только на чтение по конкретному имени |
| `exceeded quota` | `kubectl describe resourcequota student-budget`; сумму requests/limits всех контейнеров и БД |
| `Pending` | `kubectl describe pod <имя>`, `kubectl get pvc`, `kubectl get events --sort-by=.lastTimestamp` |
| `ImagePullBackOff` | Имя образа и его доступность; внешние registry из облака иногда недоступны. nginx и PostgreSQL из примеров предзагружены на воркеры |
| HTTP 404 | Совпадают ли hostname URL и HTTPRoute, выбраны ли свой listener и правильный namespace |
| HTTP 503 | Есть ли Ready Pod-ы за Service, совпадают ли selector и порты, не закрыт ли вход NetworkPolicy |
| Ошибка TLS | Используйте своё полное DNS-имя и правильный CA; не обращайтесь к HTTPS по одному IP вместо hostname |
| БД не готова | `kubectl describe cluster.postgresql.cnpg.io lab-db`, логи Pod-а БД, PVC и свободная квота |

Для отделения проблем приложения от маршрута:

```bash
kubectl logs deployment/hello --tail=50
kubectl port-forward service/hello 8080:80
# В другом терминале:
curl http://127.0.0.1:8080/
```

Для проверки Gateway при проблемах локального DNS, с сохранением проверки TLS:

```bash
curl --resolve "app.${NS}.mlops.goctf.online:443:37.18.117.114" \
  "https://app.${NS}.mlops.goctf.online/"
```

## 8. Очистка лабораторной

Удаляйте только ресурсы своей лабораторной. Выполняйте команды для файлов,
которые вы создавали:

```bash
kubectl delete -f route.yaml --ignore-not-found
kubectl delete -f hello.yaml --ignore-not-found
kubectl delete -f internal-tls.yaml --ignore-not-found
kubectl delete secret hello-internal-tls --ignore-not-found
```

Удаление следующего файла **удаляет PostgreSQL и его PVC с данными**:

```bash
kubectl delete -f postgres.yaml
```

В этой конфигурации у PVC PostgreSQL включено удаление вместе с Cluster.
Перед удалением нужной БД сохраните дамп, например через `pg_dump`, используя
те же параметры TLS и подключения. Остановка приложения через
`kubectl scale deployment/hello --replicas=0` не удаляет данные БД.
