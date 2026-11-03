# 配置管理：Secret 与 ConfigMap

Secret 存口令，ConfigMap 存明文配置，二者都进 etcd。

## Secret

`data` 只是 Base64。`echo -n 'operator'` 得到 `b3BlcmF0b3I=`，`S3cret-91` 得到 `UzNjcmV0LTkx`。漏掉 `-n` 会编进换行。

### 用环境变量注入

`stringData` 写原文，由 apiserver 编码。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-cred
type: Opaque
stringData:
  username: operator
  password: S3cret-91
---
apiVersion: v1
kind: Pod
metadata:
  name: cfg-demo
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      env:
        - name: APP_OPERATOR
          valueFrom:
            secretKeyRef:
              name: db-cred
              key: username
        - name: APP_PASSPHRASE
          valueFrom:
            secretKeyRef:
              name: db-cred
              key: password
```

```bash
kubectl apply -f db-cred.yaml
kubectl exec cfg-demo -- sh -c 'echo "$APP_OPERATOR $APP_PASSPHRASE"'
kubectl delete -f db-cred.yaml
```

### 用卷挂进容器

把同一 Secret 挂到目录后，每个键成为一个文件。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cfg-demo
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: cred
          mountPath: /etc/db-cred
          readOnly: true
  volumes:
    - name: cred
      secret:
        secretName: db-cred
```

```bash
kubectl apply -f db-cred-vol.yaml
kubectl exec cfg-demo -- cat /etc/db-cred/username
```

## ConfigMap

ConfigMap 放明文，不要放口令。`cache.properties` 含 `cache.host=10.6.4.30`、`cache.port=6380`、`cache.password=lab-cache-6380`。

```bash
kubectl create configmap cache-settings --from-file=cache.properties
kubectl describe cm cache-settings
```

### 以卷的形式挂载

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cfg-demo
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "cat /etc/cache/cache.properties"]
      volumeMounts:
        - name: settings
          mountPath: /etc/cache
  volumes:
    - name: settings
      configMap:
        name: cache-settings
```

```bash
kubectl apply -f cache-vol.yaml
kubectl logs cfg-demo
```

### 以变量的形式注入

环境变量名不能带点，所以键用大写。`cache-env.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cache-env
data:
  CACHE_HOST: "10.6.4.30"
  CACHE_PORT: "6380"
---
apiVersion: v1
kind: Pod
metadata:
  name: cfg-demo
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "echo $CACHE_HOST:$CACHE_PORT"]
      envFrom:
        - configMapRef:
            name: cache-env
```

```bash
kubectl apply -f cache-env.yaml
kubectl logs cfg-demo
```

## 常见问题

- 换行被编进 Base64 后，口令会对不上；卷挂到已有目录会盖住原文件。
