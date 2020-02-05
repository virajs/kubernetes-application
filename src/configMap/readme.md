#### ConfigMap详解

应用部署的一个最佳实践是将应用所需的配置信息与程序进行分离，这样可以使得应用程序被更好地复用，通过不同的配置也能实现更灵活的功能。
将应用打包为容器镜像后，可以通过环境变量或者外挂文件的方式在创建容器时进行配置注入，但在大规模容器集群的环境中，对多个容器进行不同的配置将变得非常复杂。

ConfigMap供容器使用的主要用法：

* 生成为容器内的环境变量；
* 设置容器启动命令的启动参数（需设置为环境变量）；
* 以Volume的形式挂载为容器内部的文件或目录。

#### 创建ConfigMap资源对象

创建一个 `test.yaml` 的ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test
data:
  apploglevel: info
  appdatadir: /var/data
```

运行:

```bash
> kubectl create -f test.yaml
```

查看创建好的ConfigMap:
```bash
# 查看ConfigMap列表
kubectl get configmap

# 查看ConfigMap test
kubectl describe configmap test

# 以yaml的形式输出test
kubectl get configmap test -o yaml
```

#### 通过kubectl命令行方式创建ConfigMap

如果我们不使用yaml文件，也可以直接通过`kubectl create configmap` 命令创建ConfigMap，可以使用参数 `--from-file` 或 `--from-literal` 指定内容，并且可以在一行命令中指定多个参数。


1. 通过 `--from-file` 参数从文件中进行创建，可以指定key的名称，也可以在一个命令行中创建包含多个key的ConfigMap，语法为：

```bash
> kubectl create configmap NAME --from-file=[key=]source --from-file=[key=]source
```

2. 通过 `--from-file` 参数从目录中进行创建，该目录下的每个配置文件名称都被设置为key，文件的内容被设置为value，语法为：

```bash
> kubectl create configmap NAME  --from-file=config-files-dir
```

3. 使用`--from-literal` 从文本中进行创建，直接将指定的key-value对创建为ConfigMap的内容，语法为：

```bash
> kubectl create configmap NAME --from-literal=key1=value1 --from-literal=key2=value2
```

#### 使用ConfigMap

容器应用对ConfigMap的使用有以下两种方法：

* 通过环境变量获取ConfigMap
* 通过Volume挂载的方式将ConfigMap中的内容挂载为容器内部的文件或目录

#### 在Pod中使用ConfigMap

1. 创建一个 `test.yaml`,通过环境变量方式使用ConfigMap.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test
data:
  apploglevel: info
  appdatadir: /var/data
```

在Pod的 `cm-test-pod` 的定义中，将ConfigMap `test` 中的内容以环境变量（APPLOGLEVEL和APPDATADIR）设置为容器内部的环境变量，容器的启动命令将显示这两个环境变量的值（"env | grep APP"）

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: cm-test-pod
spec:
  containers:
  - name: cm-test
    image: busybox
    command: ["/bin/sh", "-c", "env | grep APP"]
    env:
    - name: APPLOGLEVEL
      valueFrom:
        configMapKeyRef:
          name: test
          key: apploglevel
    - name: APPDATADIR
      valueFrom:
        configMapKeyRef:
          name: test
          key: appdatadir
    restartPolicy: Never
```
使用 `kubectl create -f` 命令创建该Pod，由于是测试Pod，所以该Pod在执行完启动命令后将会退出，并且不会被系统自动重启（restartPolicy: Never）：

```bash
> kubectl create -f cm-test-pod.yaml
```
如果要查看已经停止的Pod可以通过:
```bash
> kubectl get pods --show-all
```

如果需要查看该Pod的日志，可以看到启动命令“env | grep APP”的执行结果如下：
```bash
> kubectl logs cm-test-pod APPDATADIR=/var/data  APPLOGLEVEL=info
```
从Kubernetes v1.6开始，引入了一个新的字段 envFrom ，实现在Pod环境内将ConfigMap（也可用于Secret资源对象）中所定义的`key=value`自动生成为环境变量：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - configMapRef:
          name: test
  restartPolicy: Never
```

通过这个定义，在容器内部将生成如下环境变量：

```bash
apploglevel=info
appdatadir=/var/data
```
需要说明的是，环境变量的名称受POSIX命名规范（[a-zA-Z_][a-zA-Z0-9_]*）约束，不能以数字开头。

如果包含非法字符，则系统将跳过该条环境变量的创建，并记录一个Event来描述环境变量无法生成，但不会阻止Pod的启动

#### 通过volumeMount使用ConfigMap

当使用 `--from-file` 创建 ConfigMap 时， 文件名将作为键名保存在 ConfigMap 的 data 段，文件的内容变成键值。

从 ConfigMap 里的数据生成一个卷

我们通过一个名为 special-config 的 ConfigMap 的配置来熟悉下.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.level: very
  special.type: charm
```

在 Pod 的配置文件里的 volumes 段添加 ConfigMap 的名字。

这会将 ConfigMap 数据添加到 `volumeMounts.mountPath` 指定的目录里面（在这个例子里是 /etc/config）。command 段引用了 ConfigMap 里的 special.level.

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: api-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config
  restartPolicy: Never
```
Pod 运行起来后, 运行：

```bash
> ls /etc/config/

special.level
special.type
```

#### 添加 ConfigMap 数据到卷里指定路径

使用 path 变量定义 ConfigMap 数据的文件路径。在我们这个例子里，special.level 将会被挂载在 config-volume 的文件 `/etc/config/keys` 下.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh","-c","cat /etc/config/keys" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.level
          path: keys
  restartPolicy: Never
```
Pod 运行起来后，运行：

```bash
> cat /etc/config/keys

very
```

#### 使用ConfigMap的限制条件

使用ConfigMap的限制条件如下：

* ConfigMap必须在Pod之前创建（除非您把 ConfigMap 标志成”optional”）。如果您引用了一个不存在的 ConfigMap， 那这个Pod是无法启动的。就像引用了不存在的 Key 会导致 Pod 无法启动一样。

* ConfigMap受Namespace限制，只有处于相同的Namespace中的Pod可以引用它；

* ConfigMap中的配额管理还未能实现；

* kubelet值支持可以被API Server管理的Pod使用ConfigMap。kubelet在当前Node上通过 `--manifest-url`或 `--config` 自动创建的静态Pod将无法引用ConfigMap；

* 在Pod对ConfigMap进行挂载（volumeMount）操作是，容器内部只能挂载为目录，无法挂载为文件。

* 在挂载到容器内部后，目录中将包含ConfigMap定义的每个item，如果该目录下原理还有其他文件，则容器内的该目录会被挂载的ConfigMap覆盖。