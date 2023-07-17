# 第七章 ConfigMap 和 Secret：配置应⽤程序

## 1.配置容器化应用程序

三种配置方式：

- 向容器传递命令行参数。
- 为每个容器设置环境变量。
- 通过卷将配置文件挂载至容器中。

容器传递配置文件的问题：修改配置需重新构建镜像，配置文件完全公开。解决：配置使用 ConfigMap 或 Secret 卷挂载。

## 2.向容器传递命令行参数

Dockerfile 中 `ENTRYPOINT` 为命令，`CMD` 为其参数。但参数能被 `docker run <image> <arg_values>` 中的参数覆盖。

```yaml
ENTRYPOINT ["/bin/fortuneloop.sh"] # 在脚本中通过 $1 获取 CMD 第一个参数，Go 中 os.Args[1] 类似
CMD ["10", "11"]
```

二者等同于 Pod 中的 `command` 和 `args`，但 pod 可通过 image 的 command 和 args 子标签进行覆盖，注意参数必须是字符串：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
    - image: luksa/fortune:args
      args: ["2"]
      name: html-generator
      volumeMounts:
        - name: html
          mountPath: /var/htdocs
    - image: nginx:alpine
      name: web-server
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
      ports:
        - containerPort: 80
          protocol: TCP
  volumes:
    - name: html
      emptyDir: {}
```

## 3.为容器设置环境变量

只能在各容器级别注入环境变量，而非 Pod 级别。配置容器部分 `spec.containers.env` 指定即可：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env
spec:
  containers:
    - image: luksa/fortune:env
      env:
        - name: INTERVAL
          value: "30" # 可引用其他环境变量
      name: html-generator
      volumeMounts:
        - name: html
          mountPath: /var/htdocs
    - image: nginx:alpine
      name: web-server
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
      ports:
        - containerPort: 80
          protocol: TCP
  volumes:
    - name: html
      emptyDir: {}
```

缺点：硬编码环境变量可能在多个环境下值不同无法复用，需将配置项解耦。

## 4.ConfigMap 卷

存储非敏感信息的文本配置文件。

```shell
# 创建 cm 的四种方式：可从 kv 字面量、配置文件、有命名配置文件、目录下所有配置文件
> kubectl create configmap fortune-config --from-literal=sleep-interval=25 # 从 kv 字面量创建 cm
> kubectl create configmap fortune-config --from-file=nginx-conf=my-nginx-config.conf # 指定 k 的文件创建 cm
```

通过 yaml 创建 configMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fortune-config
data:
  sleep-interval: "25"
```

两种方式将 cm 中的值传递给 Pod 中的容器：

### 设置环境变量或命令行参数

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
    - image: luksa/fortune:env
      name: html-generator
      env:
        - name: INTERVAL # 取 CM fortune-config 中的 sleep-interval，作为 html-generator 容器环境变量 INTERVAL 的值
          valueFrom:
            configMapKeyRef:
              name: fortune-config-cm
              key: sleep-interval
      envFrom: # 批量导入 cm 的所有 kv 作为环境变量，并加上前缀
        - prefix: CONF_
          configMapRef:
            name: fortune-config-cm
# ...
```

可使用 `kubectl get cm fortune-config -o yaml` 查看 CM 的 data 配置项。

### 配置 ConfigMap 卷

当配置项过长需放入配置文件时，可将配置文件暴露为 cm 并用卷引用，从而在各容器内部挂载读取。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume
spec:
  containers:
    - name: html-generator
      image: yinzige/fortuneloop:env
      env:
        - name: INTERVAL
          valueFrom:
            configMapKeyRef:
              key: sleep-interval # raw key file name
              name: fortune-config # cm name
      volumeMounts:
        - mountPath: /var/htdocs
          name: html
    - name: web-server
      image: nginx:alpine
      volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: html
          readOnly: true
        - mountPath: /etc/nginx/conf.d/gzip_in.conf
          name: config
          subPath: gzip.conf # 使用 subPath 只挂载部分卷 gzip.conf 到指定目录下指定文件 gzip_in.conf
          readOnly: true
      ports:
        - containerPort: 80
          name: http
          protocol: TCP
  volumes:
    - name: html
      emptyDir: {}
    - name: config
      configMap:
        name: fortune-config # cm name
        defaultMode: 0666 # 设置卷文件读写权限
        items: # 使用 items 限制从 cm 暴露给卷的文件
          - key: my-nginx-config.conf
            path: gzip.conf # 把 key 文件的值 copy 一份到新文件中
```

添加 `items` 来暴露指定的文件到卷中，`subPath` 用来挂载部分卷，而不隐藏容器目录原有的初始文件。

```shell
> kubectl exec fortune-configmap-volume -c web-server -it -- ls -lA /etc/nginx/conf.d
total 8
-rw-r--r--    1 root     root          1093 Apr 14 14:46 default.conf # subPath
-rw-rw-rw-    1 root     root           242 Apr 27 16:49 gzip_in.conf
```

### ConfigMap 场景

使用 `kubectl edit cm fortune-config` 修后，容器中对应挂载的卷文件会延迟将修改同步。问题：若 pod 应用不支持配置文件的热更新，那同步了的修改并不会再旧 pod 生效，反而新起的 pod 会生效，造成新旧配置共存的问题。

场景：cm 的特性是不变性，若 pod 应用本身支持热更新，则可修改 cm 动态更新，但注意有 k8s 的监听延迟。

## 5.Secret

存储敏感的配置数据，大小限制 1MB，其配置条目会以 Base64 编码二进制后存储：

```shell
> kubectl create secret generic fortune-auth --from-file=fortune-auth/ # password.txt
```

在 pod 中加载：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-with-serect
spec:
  containers:
    - name: fortune-auth-main
      image: yinzige/fortuneloop
      volumeMounts:
        - mountPath: /tmp/no_password.txt
          subPath: password.txt
          name: auth
  volumes:
    - name: auth
      secret:
        secretName: fortune-auth
```

读取正常：

```shell
> kubectl exec fortune-with-serect -it -- ls -lh /tmp
total 4.0K
-rw-r--r-- 1 root root 10 Apr 27 17:51 no_password.txt
> kubectl exec fortune-with-serect -it -- mount | grep password
tmpfs on /tmp/no_password.txt type tmpfs (ro,relatime) # secret 仅存储在内存中
```

secret 可用于从镜像仓库中拉取 private 镜像，需配置专用的 secret 使用：

```yaml
apiVersion: v1
kind: Pod
spec:
  imagePullSecrets:
    - name: dockerhub-secret
  containers: # ...
```
