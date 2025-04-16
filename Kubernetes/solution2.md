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

[root@clusterinfo vikas (Active)]# kubectl get pods --all-namespaces -o=jsonpath='{range .items[*]}{.metadata.namespace}|{.metadata.name}|{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}{end}' | grep whereabout
kube-system|whereabouts-4fc45|
kube-system|whereabouts-5r7ps|
kube-system|whereabouts-5tr79|
kube-system|whereabouts-6gg5b|
kube-system|whereabouts-6j9t8|
kube-system|whereabouts-8pvx6|
kube-system|whereabouts-8zd8w|
kube-system|whereabouts-9xdxv|
kube-system|whereabouts-c2pqz|
kube-system|whereabouts-f4628|
kube-system|whereabouts-h7dzr|
kube-system|whereabouts-hr9x5|
kube-system|whereabouts-htssh|
kube-system|whereabouts-hvfm2|
kube-system|whereabouts-jgxtt|
kube-system|whereabouts-jkmcc|
kube-system|whereabouts-krnfs|
kube-system|whereabouts-l28td|
kube-system|whereabouts-lw9nl|
kube-system|whereabouts-m4ltt|
kube-system|whereabouts-mv27x|
kube-system|whereabouts-qvn8g|
kube-system|whereabouts-rlj88|
kube-system|whereabouts-rv2fp|
kube-system|whereabouts-s6dw9|
kube-system|whereabouts-shrnp|
kube-system|whereabouts-vwqzk|
kube-system|whereabouts-x6hnl|
kube-system|whereabouts-xtnts|
kube-system|whereabouts-zh2mz|
[root@opl-n-kat-ksdm-masterbm-0 cbis-admin (Active)]#




```
