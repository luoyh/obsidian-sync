
```bash

# 进入pod
> kubectl exec -ti <pod name> -n <namespace> -- bash

# 查看日志
> kubectl logs -f --tail 100 <pod name> -n <namespace> 

# 查看配置, like docker inspect
> kubectl describe configmap <configmap name> -n <namespace>
> kubectl describe pod <pod name> -n <namespace>
> kubectl describe deployment <deployment name> -n <namespace>

# cp
> kubectl cp <pod name>:/a/b.txt /b.txt -n <namespace>


```