# Подключение к digradation

Кластер: `stud-cluster`. Неймспейс и контекст: `digradation`.
Kubeconfig: `digradation.kubeconfig` из отдельного архива `digradation.zip`.
Он содержит все данные для подключения; cloudlogin, SSH и доступ в Cloud.ru не нужны.

## Подключение

Нужен установленный `kubectl`. Распакуйте архив, откройте терминал в этой папке.

macOS / Linux, bash или zsh:

```bash
chmod 600 digradation.kubeconfig
export KUBECONFIG="$PWD/digradation.kubeconfig"
kubectl config current-context
kubectl get pods
kubectl get resourcequota student-budget
```

Ожидаемый контекст — `digradation`. Команды без `-n` работают в этом неймспейсе.
Альтернатива без изменения переменной окружения:

```bash
kubectl --kubeconfig ./digradation.kubeconfig -n digradation get pods
```

Windows PowerShell:

```powershell
$env:KUBECONFIG = (Resolve-Path .\digradation.kubeconfig).Path
kubectl config current-context
kubectl get pods
```

API Kubernetes: `https://37.18.117.109:6443`.
Доступны управление ресурсами своего неймспейса, exec, logs и port-forward.
Квота: requests и limits суммарно по **2.6 CPU и 4 GB RAM**.
Администратор управляет квотой и общей инфраструктурой.

## Приложение и HTTPS

```bash
kubectl apply -f hello.yaml
kubectl rollout status deployment/hello --timeout=180s
curl https://app.digradation.mlops.goctf.online/
```

Для других приложений используйте `<app>.digradation.mlops.goctf.online`.
HTTPRoute ссылается на `envoy-gateway-system/itmo-shared`,
слушатель `http-digradation`, и на ListenerSet
`envoy-gateway-system/itmo-https`, слушатель `https-digradation`.
HTTPS-сертификат и его продление обслуживаются централизованно.
Общий публичный адрес — `37.18.117.114`, порты 80 и 443.

## PostgreSQL

```bash
kubectl apply -f postgres.yaml
kubectl wait cluster.postgresql.cnpg.io/lab-db --for=condition=Ready --timeout=10m
kubectl get pods -l cnpg.io/cluster=lab-db
```

Внутри неймспейса: `lab-db-rw:5432`, база и пользователь — `app`.
Для подключения с компьютера запустите в первом терминале:

```bash
kubectl port-forward service/lab-db-rw 15432:5432
```

Во втором терминале из той же папки, с установленным `psql`:

```bash
export KUBECONFIG="$PWD/digradation.kubeconfig"
export PGPASSWORD="$(kubectl get secret lab-db-app -o jsonpath='{.data.password}' | base64 --decode)"
kubectl get secret lab-db-ca -o jsonpath='{.data.ca\.crt}' | base64 --decode > lab-db-ca.crt
psql "host=lab-db-rw hostaddr=127.0.0.1 port=15432 dbname=app user=app sslmode=verify-full sslrootcert=$PWD/lab-db-ca.crt"
unset PGPASSWORD
```

`host=lab-db-rw` проверяется по сертификату PostgreSQL, а `hostaddr` направляет
соединение в локальный port-forward. Доступ к БД работает, пока первый терминал открыт.
Пример выделяет диск 5Gi; удаление `postgres.yaml` удаляет БД и её PVC.

Общая инструкция по Gateway API, cert-manager, подключению приложения к БД,
диагностике и очистке находится в `STUDENT-GUIDE.md`.

Kubeconfig даёт доступ к Secrets своего неймспейса. Храните его приватно и
не добавляйте в Git. Для отзыва доступа администратор удаляет Secret
`digradation/student-kubeconfig-token`.
