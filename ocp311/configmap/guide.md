## Create container f5-hello-world
```
oc create -f f5-hello-world-deployment.yaml
oc create -f f5-hello-world-configmap.yaml
oc create -f f5-hello-world-service.yaml
```

## Delete container f5-demo-app-route
```
oc delete -f f5-hello-world-service.yaml
oc delete -f f5-hello-world-configmap.yaml
oc delete -f f5-hello-world-deployment.yaml
```
