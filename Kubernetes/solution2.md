Question-1:// if you want to restart the ds then how?


```bash
Kubectl rollout restart ds <> -n example
```

```bash
Your ClusterIP service network (service CIDR) is:
How to find it 

[root@masterbm ~]# grep service-cluster-ip-range /etc/kubernetes/manifests/kube-apiserver.yaml
    - --service-cluster-ip-range=10.96.0.0/12
[root@masterbm ~]#

```
