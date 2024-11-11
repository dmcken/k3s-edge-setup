

Copied from:
https://medium.com/swlh/serving-bind-dns-in-kubernetes-8639fce37448

Deployment:
```bash
kubectl create -f bind-deployment.yml
kubectl get pods -o wide
kubectl logs <pod_name>
```

Service:
```bash
kubectl create -f bind-service.yml
kubectl get services
```

Test:
```bash
dig
```

