# Kubernetes 问题排查与记录

## kubectl 排查服务问题

### 部署服务失败
- 如果 k8s 上部署服务失败了，我们可以使用如下命令排查
```shell
kubectl describe ${RESOURCE} ${NAME}
```

拉到最后看 `Events` 部分，会显示 k8s 在部署这个服务过程的关键日志。

一般来说，通过 `kubectl describe pod ${POD_NAME}` 已经能定位绝大部分部署失败的问题了。

当然，具体问题还是需要具体分析。

### 部署的服务异常

如果服务部署成功了，且状态为 `running`，那么需要进入 Pod 内部容器去查看自己的服务日志

- 查看 Pod 内部某个 container 打印的日志：
```shell
kubectl log ${POD_NAME} -c ${CONTAAINER_NAME}
```

- 进入 Pod 内部某个 container：
```shell
kubectl exec -it [options] ${POD_NAME} -c ${CONTAINER_NAME} [args]
```

这个命令的作用是通过 kubectl 执行了 `docker exec xxx` 进入到容器实例内部。之后就是用户检查自己服务的日志来定位问题。


## k8s 集群部署问题

### 初始化报错

- 错误日志：
```shell
[root@k8s-master ~]# kubeadm init --kubernetes-version=1.20.10  --apiserver-advertise-address=192.168.65.160 --image-repository registry.aliyuncs.com/google_containers --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16
[init] Using Kubernetes version: v1.20.10
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR Port-10259]: Port 10259 is in use
        [ERROR Port-10257]: Port 10257 is in use
        [ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: /etc/kubernetes/manifests/kube-apiserver.yaml already exists
        [ERROR FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml]: /etc/kubernetes/manifests/kube-controller-manager.yaml already exists
        [ERROR FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: /etc/kubernetes/manifests/kube-scheduler.yaml already exists
        [ERROR FileAvailable--etc-kubernetes-manifests-etcd.yaml]: /etc/kubernetes/manifests/etcd.yaml already exists
        [ERROR Port-10250]: Port 10250 is in use
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

- 报错原因

由于上次初始化失败导致的部分资源被占用，需要重置 kubeadm。

- 解决办法
```shell
[root@k8s-master ~]# kubeadm reset  // 重启 kubeadm
[reset] Reading configuration from the cluster...
[reset] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0105 10:07:53.337894   30265 reset.go:101] [reset] Unable to fetch the kubeadm-config ConfigMap from cluster: failed to get config map: Get "https://39.108.37.153:6443/api/v1/namespaces/kube-system/configmaps/kubeadm-config?timeout=10s": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
[reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] Are you sure you want to proceed? [y/N]: y  // 输入 y
[preflight] Running pre-flight checks
W0105 10:07:57.015266   30265 removeetcdmember.go:80] [reset] No kubeadm config, using etcd pod spec to get data directory
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
[reset] Deleting contents of stateful directories: [/var/lib/etcd /var/lib/kubelet /var/lib/dockershim /var/run/kubernetes /var/lib/cni]

The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually by using the "iptables" command.

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.
```

### 配置问题

- 配置 kubernetes 出现以下报错：
```shell
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

- 报错原因：
  
kubernetes master 没有与本机绑定，集群初始化的时候没有设置 

- 解决办法：
```shell
export KUBECONFIG=/etc/kubernetes/admin.conf
```