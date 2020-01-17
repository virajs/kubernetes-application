#### kubernetes yaml 文件

需要修改yaml中的image值，把k8s.gcr.io 全部修改为 registry.cn-hangzhou.aliyuncs.com/google_containers

#### 访问dashboard


访问dashboard有下面几种方式:

* Nodport方式访问dashboard，service类型改为NodePort

* loadbalacer方式，service类型改为loadbalacer

* Ingress方式访问dashboard

* API server方式访问 dashboard

* kubectl proxy方式访问dashboard

#### NodePort方式
```bash
>  vim kubernetes-dashboard.yaml 
......
---
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 32631 #增加nodePort: 32631
  selector:
    k8s-app: kubernetes-dashboard
```
重新应用yaml文件:

```bash
> kubectl apply -f kubernetes-dashboard.yaml
```

查看service，TYPE类型已经变为NodePort，端口为32631

```bash
> kubectl get service -n kube-system | grep dashboard
kubernetes-dashboard   NodePort    10.102.72.242    192.168.157.193,192.168.157.196,192.168.157.198   443:32631/TCP            116d
```

Dashboard 支持 Kubeconfig 和 Token 两种认证方式，我们这里选择Token认证方式登录：

创建dashboard-adminuser.yaml：
```bash
> vim dashboard-adminuser.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
name: admin-user
namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
name: admin-user
roleRef:
apiGroup: rbac.authorization.k8s.io
kind: ClusterRole
name: cluster-admin
subjects:
- kind: ServiceAccount
name: admin-user
namespace: kube-system
```

应用加载文件:

```bash
> kubectl create -f dashboard-adminuser.yaml
```

上面创建了一个叫admin-user的服务账号，并放在kube-system命名空间下，并将cluster-admin角色绑定到admin-user账户，这样admin-user账户就有了管理员的权限。默认情况下，kubeadm创建集群时已经创建了cluster-admin角色，我们直接绑定即可。

查看admin-user账户的token:
```bash
> kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-r9xm9
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 40357834-60c7-4172-a69c-b87bbbded009

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      ...
ca.crt:     1025 bytes
```
然后用token直接登录.

#### loadbalacer方式

```bash
> vim kubernetes-dashboard.yaml 
......
---
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
labels:
  k8s-app: kubernetes-dashboard
name: kubernetes-dashboard
namespace: kube-system
spec:
type: LoadBalancer
ports:
  - port: 443
    targetPort: 8443
selector:
  k8s-app: kubernetes-dashboard
```

重新应用yaml文件
```bash
> kubectl apply -f kubernetes-dashboard.yaml --force
```

注意: 由nodeport改为其他类型需要添加–forece才能执行成功。

查看service，TYPE类型已经变为LoadBalancer，并且分配了EXTERNAL-IP：
```bash
> kubectl get service kubernetes-dashboard -n kube-system 
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP                                      PORT(S)         AGE
kubernetes-dashboard   LoadBalancer   10.102.72.242   192.168.157.193,192.168.157.196,192.168.157.198   443:32631/TCP   10m

```

#### Ingress方式

Ingress作为一种API对象，用来管理从外部对集群内服务器的访问。Ingress可以提供负载均衡、SSL截止和虚拟主机服务等。

1. 创建Namespace及Service Account
```bash
> kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/install/common/ns-and-sa.yaml
```

2. 创建TLS证书及私钥，以下使用了示例的证书和私钥，建议自己生成
```bash
> kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/install/common/default-server-secret.yaml
```
3. 创建Config Map
```bash
> kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/install/common/nginx-config.yaml
```   
4. 创建RBAC
```bash
> kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/install/rbac/rbac.yaml
```   

5.部署Ingress Controller，下载image
```bash
> docker pull nginx/nginx-ingress:alpine
```

Ingress Controller有两种部署方式：

* Deployment：使用Deployment可以动态调整Ingress Controller的replica数量
* DaemonSet：使用DaemonSet可以使Ingress Controller运行在每台node或一组node之中

1. 使用Deployment部署
```bash
> cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      serviceAccountName: nginx-ingress
      containers:
      - image: nginx/nginx-ingress:alpine
        imagePullPolicy: Always
        name: nginx-ingress
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        args:
          - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
          - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
EOF
```
2. 使用DaemonSet部署
```bash
> cat <<EOF | kubectl create -f -
  apiVersion: extensions/v1beta1
  kind: DaemonSet
  metadata:
    name: nginx-ingress
    namespace: nginx-ingress
  spec:
    selector:
      matchLabels:
        app: nginx-ingress
    template:
      metadata:
        labels:
          app: nginx-ingress
      spec:
        serviceAccountName: nginx-ingress
        containers:
        - image: nginx/nginx-ingress:alpine
          imagePullPolicy: Always
          name: nginx-ingress
          ports:
          - name: http
            containerPort: 80
            hostPort: 80
          - name: https
            containerPort: 443
            hostPort: 443
          env:
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          args:
            - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
            - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
  EOF
```

确认Ingress Controller运行状态:
```bash
> kubectl get pods --namespace=nginx-ingress
```

如果部署方式是DaemonSet，则Ingress Controller的80和443端口将映射到Node的相同端口，访问Ingress Controller时，使用任意Node的IP加端口即可访问。

如果部署方式是Deployment，则需要创建基于NodePort的Service来访问（也可以使用LoadBalancer），方法如下：
```bash
> kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/master/install/service/nodeport.yaml
```
   
若要卸载Ingress Controller，直接删除整个命名空间即可:
```bash
> kubectl delete namespace nginx-ingress
```

* 应用示例:

本示例中NGINX Ingress Controller使用DaemonSet方式部署，好处是不用去找NodePort的端口号。示例的Ingress主机域名为dashboard.test.local，在DNS中配置该A记录对应任意worker节点地址即可访问。

创建dashboard的TLS证书，若有CA颁发的证书可跳过这步, 通过openssl生成了域名为dashboard.test.local的10年（3650天）自签名证书.

```bash
> openssl req -x509 -sha256 -nodes -days 3650 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=dashboard.test.local/O=dashboard.test.local"
```

基于以上创建的证书生成secret:
```bash
> kubectl create secret tls kubernetes-dashboard-certs --key tls.key --cert tls.crt -n kube-system
```

创建Ingress
```bash
> cat <<EOF | kubectl create -f -
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: ingress-dashboard
    namespace: kube-system
    annotations:
      nginx.org/ssl-services: "kubernetes-dashboard" 
  spec:
    rules:
    - host: dashboard.test.local
      http:
        paths:
        - backend:
            serviceName: kubernetes-dashboard
            servicePort: 443
    tls:
    - hosts:
      - dashboard.test.local
      secretName: kubernetes-dashboard-certs
  EOF

```

然后使用浏览器访问`https://dashboard.test.local`即可访问Kubernetes Dashboard了


#### API Server访问

通过API Server访问Dashboard的地址为：
`https://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

若k8s为集群高可用部署，则使用master vip地址和相应的端口

根据之前k8s部署教程，示例的访问地址为`https://k8s.test.local:8443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`。

但是直接访问该网址会返回Anonymous Forbidden的错误，是由于RBAC给未认证用户分配的默认身份没有访问权限的关系。可以通过k8s的admin.conf来生成证书，方法如下：

```bash
> grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
> grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
> openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```

将生成的证书导入浏览器，并重启浏览器再次访问Dashboard地址，登录时会显示选择证书，点击相应的证书便可进行登录.

Kubernetes使用token进行用户认证，为了正确访问Dashboard，需要创建相应的信息.

创建admin-user用户:
```bash
> cat <<EOF | kubectl create -f -
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: admin-user
    namespace: kube-system
  ---
  apiVersion: rbac.authorization.k8s.io/v1beta1
  kind: ClusterRoleBinding
  metadata:
    name: admin-user
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-admin
  subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kube-system
  EOF
```

再获取admin-user的token，token为图红框中那串base64字符串.
```bash
> kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}') | awk '$1=="token:"{print $2}'
```


#### kubectl proxy方式

```bash
> kubectl proxy 
Starting to serve on 127.0.0.1:8001
```

现在就可以通过以下链接来访问Dashborad UI:

`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

这种方式默认情况下，只能从本地访问（启动它的机器）。

我们也可以使用–address和–accept-hosts参数来允许外部访问：

```bash
> kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$'
Starting to serve on [::]:8001
```
`http://<master-ip>:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

可以成功访问到登录界面，但是填入token也无法登录，这是因为Dashboard只允许localhost和127.0.0.1使用HTTP连接进行访问，而其它地址只允许使用HTTPS。

因此，如果需要在非本机访问Dashboard的话，只能选择其他访问方式。