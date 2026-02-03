# Kubernetes (k8s)

## Простой ответ
- Платформа, которая сама раскладывает контейнеры по нодам, следит за их здоровьем и обновляет без даунтайма.
- Вы описываете желаемое состояние (манифесты), control plane приводит его к факту.

## Ответ
Оркестрация контейнеров: Pods, Deployments, Services, ConfigMap/Secret, Ingress. Даёт автоскейлинг, самовосстановление, балансировку и обновления без простоя.

## Архитектура
- Control Plane: `kube-apiserver`, `controller-manager`, `scheduler`, `etcd` (хранит состояние).
- Node: `kubelet` (жизнь подов), `kube-proxy` (сетевые правила), container runtime.
- Namespace — логическая изоляция; RBAC — управление доступом.

## Базовые объекты
- Pod — один или несколько контейнеров (минимальная единица).
- Deployment — управление ReplicaSet/Pods, rolling update/rollback (`kubectl rollout undo`).
- Service — стабильный DNS/ClusterIP/LoadBalancer/NodePort.
- Ingress — L7 роутинг/SSL termination.
- ConfigMap/Secret — конфигурация/секреты (env/volume).
- Volume/PVC/PV/StorageClass — постоянное хранилище.
- Probes — `liveness`/`readiness`/`startup` для здоровья.
- HPA — горизонтальный автоскейлинг по метрикам.

## Минимальный пример Deployment+Service+Ingress
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec:
  replicas: 3
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
        - name: app
          image: nginx:1.25
          ports: [{containerPort: 80}]
          readinessProbe: {httpGet: {path: /, port: 80}}
---
apiVersion: v1
kind: Service
metadata: {name: web}
spec:
  selector: {app: web}
  ports: [{port: 80, targetPort: 80}]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: web}
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: {service: {name: web, port: {number: 80}}}
```

## Полезные команды
- `kubectl get pods -A`, `kubectl describe pod <name>`.
- `kubectl logs -f pod/<name> -c <container>`.
- `kubectl rollout history|status|undo deployment/<name>`.
- `kubectl port-forward svc/web 8080:80`.

## Примеры

1. Deployment на 3 реплики с rolling update и `readinessProbe`.
2. Service + Ingress для публикации приложения.

## Доп. теория

1. Kubernetes работает в модели «desired state»: контроллеры стремятся к описанному состоянию.
2. Расширяемость через CRD/Operators позволяет описывать собственные сущности.
