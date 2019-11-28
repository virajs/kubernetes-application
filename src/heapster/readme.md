#### kubernetes yaml 文件

需要修改yaml中的image值，把k8s.gcr.io 全部修改为 registry.cn-hangzhou.aliyuncs.com/google_containers

#### kubernetes command

1. 用来显示当前的各种用户进程限制

```bash
> ulimit -a

core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 63459
max locked memory       (kbytes, -l) 8589934592
max memory size         (kbytes, -m) unlimited
open files                      (-n) 655360
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 4096
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

2. 更新test-service服务
```bash
> kubectl apply -f test-service.yml
```

3. 获取在kube-system空间上运行的pod和service
```bash
> kubectl get pod,svc -n kube-system
```

4. 获取所有命名空间上的pods
```bash
> kubectl get pods --all-namespaces
```

5. 根据metadata列表服务按名称排序
```bash
> kubectl get pods --sort-by=.metadata.name
```

6. 列出所有namespace中的所有 service
```bash
>  kubectl get services  --all-namespaces 
 
NAMESPACE     NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP                                       PORT(S)                  AGE
default       heapster               ClusterIP   10.98.124.49     <none>                                            80/TCP                   27h
default       kubernetes             ClusterIP   10.96.0.1        <none>                                            443/TCP                  111d
default       monitoring-grafana     ClusterIP   10.103.85.89     <none>                                            80/TCP                   22h
kube-system   heapster               ClusterIP   10.103.46.91     <none>                                            80/TCP                   29h
kube-system   kube-dns               ClusterIP   10.96.0.10       <none>                                            53/UDP,53/TCP,9153/TCP   111d
kube-system   kubelet                ClusterIP   None             <none>                                            10250/TCP                106d
kube-system   kubernetes-dashboard   NodePort    10.102.72.242    192.168.132.190,192.168.132.193,192.132.132.194   443:32631/TCP            105d
kube-system   metrics-server         ClusterIP   10.98.249.8      <none>                                            443/TCP                  106d
kube-system   monitoring-grafana     ClusterIP   10.102.205.153   <none>                                            80/TCP                   29h
kube-system   monitoring-influxdb    ClusterIP   10.97.131.84     <none>                                            8086/TCP                 29h                       
```

7. 使用详细输出来描述pods heapster
```bash
> kubectl describe pods heapster-646cd5db76-95wpz
```

8. 获取api的各个版本
```bash
> kubectl api-versions
```

9. 获取所有命名空间上部署的pods
```bash
> kubectl get deployment --all-namespaces
```

10. 获取kube-system命名空间上部署的pods
```bash
>  kubectl get deployment -n kube-system
```

11. 获取所有节点
```bash
> kubectl get nodes
```

12. 获取所有命名空间命名空间上部署的services
```bash
> kubectl get services --all-namespaces
```

13. 获取kube-system命名空间上部署的services
```bash
> kubectl get services -n kube-system
```

14. 获取kubeadm的版本
```bash
>  kubeadm version --output json
```

15. 编辑名为 docker-registry 的 service
```bash
> kubectl edit svc/docker-registry   
```

16. 与运行中的 Pod 交互
```bash
> kubectl logs my-pod                                 # dump 输出 pod 的日志（stdout）
> kubectl logs my-pod -c my-container                 # dump 输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
> kubectl logs -f my-pod                              # 流式输出 pod 的日志（stdout）
> kubectl logs -f my-pod -c my-container              # 流式输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
> kubectl run -i --tty busybox --image=busybox -- sh  # 交互式 shell 的方式运行 pod
> kubectl attach my-pod -i                            # 连接到运行中的容器
> kubectl port-forward my-pod 5000:6000               # 转发 pod 中的 6000 端口到本地的 5000 端口
> kubectl exec my-pod -- ls /                         # 在已存在的容器中执行命令（只有一个容器的情况下）
> kubectl exec my-pod -c my-container -- ls /         # 在已存在的容器中执行命令（pod 中有多个容器的情况下）
> kubectl top pod POD_NAME --containers               # 显示指定 pod 和容器的指标度量
```

17. 在编辑器中编辑任何 API 资源
```bash
> kubectl edit svc/docker-registry                      # 编辑名为 docker-registry 的 service
> KUBE_EDITOR="nano" kubectl edit svc/docker-registry   # 使用其它编辑器
```

18. 更新资源
```bash
> kubectl rolling-update frontend-v1 -f frontend-v2.json           # 滚动更新 pod frontend-v1
> kubectl rolling-update frontend-v1 frontend-v2 --image=image:v2  # 更新资源名称并更新镜像
> kubectl rolling-update frontend --image=image:v2                 # 更新 frontend pod 中的镜像
> kubectl rolling-update frontend-v1 frontend-v2 --rollback        # 退出已存在的进行中的滚动更新
> cat pod.json | kubectl replace -f -                              # 基于 stdin 输入的 JSON 替换 pod

# 强制替换，删除后重新创建资源。会导致服务中断。
> kubectl replace --force -f ./pod.json

# 为 nginx RC 创建服务，启用本地 80 端口连接到容器上的 8000 端口
> kubectl expose rc nginx --port=80 --target-port=8000

# 更新单容器 pod 的镜像版本（tag）到 v4
> kubectl get pod mypod -o yaml | sed 's/\(image: myimage\):.*$/\1:v4/' | kubectl replace -f -

> kubectl label pods my-pod new-label=awesome                      # 添加标签
> kubectl annotate pods my-pod icon-url=http://goo.gl/XXBTWq       # 添加注解
> kubectl autoscale deployment foo --min=2 --max=10                # 自动扩展 deployment “foo”
```

19. 显示和查找资源
```bash
> kubectl get services                          # 列出所有 namespace 中的所有 service
> kubectl get pods --all-namespaces             # 列出所有 namespace 中的所有 pod
> kubectl get pods -o wide                      # 列出所有 pod 并显示详细信息
> kubectl get deployment my-dep                 # 列出指定 deployment
> kubectl get pods --include-uninitialized      # 列出该 namespace 中的所有 pod 包括未初始化的

# 使用详细输出来描述命令
> kubectl describe nodes my-node
> kubectl describe pods my-pod

> kubectl get services --sort-by=.metadata.name # List Services Sorted by Name

# 根据重启次数排序列出 pod
> kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# 获取所有具有 app=cassandra 的 pod 中的 version 标签
> kubectl get pods --selector=app=cassandra rc -o \
  jsonpath='{.items[*].metadata.labels.version}'

# 获取所有节点的 ExternalIP
> kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# 列出属于某个 PC 的 Pod 的名字
# “jq”命令用于转换复杂的 jsonpath，参考 https://stedolan.github.io/jq/
> sel=${$(kubectl get rc my-rc --output=json | jq -j '.spec.selector | to_entries | .[] | "\(.key)=\(.value),"')%?}
> echo $(kubectl get pods --selector=$sel --output=jsonpath={.items..metadata.name})

# 查看哪些节点已就绪
> JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
 && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"

# 列出当前 Pod 中使用的 Secret
> kubectl get pods -o json | jq '.items[].spec.containers[].env[]?.valueFrom.secretKeyRef.name' | grep -v null | sort | uniq
```