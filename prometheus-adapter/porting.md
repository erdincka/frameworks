# Porting

## values.yaml
Add resource requests/limits and labels
```
resources: # Enable for PCAI
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 1024Mi
...
podLabels:
  hpe-ezua/app: prometheus-adapter
  hpe-ezua/type: vendor-service
```

## VirtualService
Virtualservice is not necessary for this framework. 


# Notes
- https://github.com/kubernetes-sigs/prometheus-adapter
