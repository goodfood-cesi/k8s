# k8s

## Add Container Registry
```
kubectl create secret docker-registry ghcr-secret --docker-server=registry --docker-username=username --docker-password=PAT --docker-email=email
```

### Add Deployments
```
kubectl apply -f ./deployments
```

### Deploy Load Balancer
```
kubectl create -f traefik-loadbalancer.yml
```
