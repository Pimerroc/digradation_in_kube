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
