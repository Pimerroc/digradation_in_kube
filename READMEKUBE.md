# Запуск кубера
## Внутри minikube 
```bash
kubectl apply -f k8s/redis.yaml
```
##  Для внешнего неймспейса
```bash
kubectl apply -n digradation -f k8s/redis.yaml
```
## Создание секрета, после формирования файла .env
```bash
kubectl create secret generic di-env --from-env-file=.env
```
## Создание rabbitmq.yaml
```bash
kubectl apply -f k8s/rabbitmq.yaml
```
### Внешний 
```bash
kubectl apply -n digradation -f k8s/rabbitmq.yaml
```
## Создание s3 (garage)
```bash
kubectl apply -f k8s/garage.yaml
kubectl apply -f k8s/garage-bootstrap.yaml
kubectl wait jobs/garage-bootstrap --for=condition=complete --timeout=60s
kubectl apply -f k8s/garage-config.yaml
```
