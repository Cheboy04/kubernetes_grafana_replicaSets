# Grafana en Kubernetes con Deployment y ReplicaSet

Despliegue de **Grafana** en un clúster local de Kubernetes (Minikube) usando un **Deployment** con 3 réplicas, gestionadas por un **ReplicaSet**, y expuesto con un **Service NodePort**.

## Archivos

| Archivo | Qué define |
|---|---|
| `deployment.yml` | Deployment `grafana`: 3 réplicas, estrategia `RollingUpdate`, límites de CPU/RAM y `readinessProbe` sobre `/api/health` |
| `service.yml` | Service `grafana` tipo `NodePort` (puerto fijo `30300`) con `sessionAffinity: ClientIP` |

## Requisitos

- Docker
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

## Despliegue

```bash
minikube start
kubectl apply -f deployment.yml -f service.yml
kubectl get deployment,replicaset,pods -l app=grafana -o wide
```

Acceso: `http://$(minikube ip):30300` (por ejemplo `http://192.168.49.2:30300`), o con `minikube service grafana`.

Usuario `admin`, contraseña `admin123`.

> La contraseña está en texto plano solo porque es una demo local. En un entorno real iría en un `Secret`.

## Jerarquía de objetos

```
Deployment grafana
 └─ ReplicaSet grafana-<hash>     (mantiene 3 pods vivos)
     ├─ Pod grafana-<hash>-xxxxx
     ├─ Pod grafana-<hash>-yyyyy
     └─ Pod grafana-<hash>-zzzzz
Service grafana (NodePort 30300) ──► todos los pods con label app=grafana
```

## Pruebas realizadas

```bash
# Autorreparación (ReplicaSet): el pod borrado se recrea solo
kubectl delete pod <nombre-de-un-pod>

# Escalamiento
kubectl scale deployment grafana --replicas=5
kubectl scale deployment grafana --replicas=3

# Rolling update: crea un ReplicaSet nuevo y migra los pods de a uno
kubectl set image deployment/grafana grafana=grafana/grafana:12.2.0
kubectl rollout status deployment/grafana

# Historial y rollback: reactiva el ReplicaSet anterior
kubectl rollout history deployment/grafana
kubectl rollout undo deployment/grafana
```

## Limpieza

```bash
kubectl delete -f service.yml -f deployment.yml
```

## Limitación conocida

Cada réplica guarda sus datos (usuarios, dashboards, sesiones) en su **propia** base SQLite interna:
- Un dashboard creado en un pod no aparece en los otros, y se pierde si ese pod muere.
- Por eso el Service usa `sessionAffinity: ClientIP`: así cada navegador queda fijo en un pod y la sesión no se pierde.

En producción se conectaría Grafana a una base de datos compartida (PostgreSQL/MySQL) mediante `GF_DATABASE_*`, o se usaría almacenamiento persistente.
