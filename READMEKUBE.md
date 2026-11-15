# Запуск кубера
## Внутри minikube 
```bash
kubectl apply -f k8s/redis.yaml
```
##  Для внешнего неймспейса
```bash
kubectl apply -n digradation -f k8s/redis.yaml
```
