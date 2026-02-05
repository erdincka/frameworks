# Porting

## values.yaml
Add PCAI labels
```yaml
podLabels: # ADD PCAI Labels
  # -- Pod labels for KEDA operator
  keda:
    hpe-ezua/app: keda
    hpe-ezua/type: vendor
  # -- Pod labels for KEDA Metrics Adapter
  metricsAdapter: 
    hpe-ezua/app: keda
    hpe-ezua/type: vendor  
  # -- Pod labels for KEDA Admission webhooks
  webhooks: 
    hpe-ezua/app: keda
    hpe-ezua/type: vendor  
```

## VirtualService
VirtualService is not necessary for this framework. 

# Notes
To download the original Chart
```bash
helm repo add kedacore https://kedacore.github.io/charts  
helm pull kedacore/keda
```
