# kubectl – Readiness & Liveness probes

## Access endpoint /health

```bash
curl --fail --show-error --max-time 5 http://127.0.0.1:8080/health
```

![Résultat de la commande curl sur /health](images/image1.png)

## Readiness

```bash
kubectl get pods -l app=quote-app
```

![kubectl get pods](images/image2.png)

```bash
kubectl get pods -l app=quote-app
```

![kubectl get pods](images/image3.png)

```bash
kubectl describe pod POD_NAME
```

![kubectl describe pod (1/2)](images/image4.png)

![kubectl describe pod (2/2)](images/image5.png)

```bash
kubectl logs POD_NAME
```

![kubectl logs](images/image6.png)

```bash
kubectl get endpointslices -l kubernetes.io/service-name=quote-app -o yaml
```

![kubectl get endpointslices](images/image7.png)

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/quote-app --timeout=120s
kubectl get pods -l app=quote-app
```

![Résultat de apply, rollout status et get pods](images/image8.png)

## Liveness (Lane B)

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 3
  periodSeconds: 5
```

```bash
kubectl get pods -l app=quote-app
```

![kubectl get pods (1/2)](images/image9.png)

![kubectl get pods (2/2)](images/image10.png)

```bash
kubectl describe pod POD_NAME
```

![kubectl describe pod (1/2)](images/image11.png)

![kubectl describe pod (2/2)](images/image12.png)

```bash
kubectl get endpointslices -l kubernetes.io/service-name=quote-app -o yaml
```

![kubectl get endpointslices](images/image13.png)

```bash
kubectl get pods -l app=quote-app
```

![kubectl get pods](images/image14.png)

## Synthesis

- La **readiness probe** indique si un Pod est prêt à recevoir du trafic. Si elle échoue, le Pod est retiré du Service (`READY 0/1`), mais le conteneur n'est pas redémarré (`RESTARTS 0`).
- La **liveness probe** indique si le conteneur est encore en vie. Si elle échoue, Kubernetes le redémarre : `RESTARTS` augmente, puis le Pod passe en `CrashLoopBackOff`.
