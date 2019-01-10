# WIPWIP




Kubernetesはほとんどの場合、ステートレスと想定されている。そのためKubernetes上でステートフルなDBを動作させるには大きなオーバーヘッドを伴う。データ層はコンテナから遠ざけるべき






> The important lesson is that you don’t have to run everything in Kubernetes once you have Kubernetes.

- https://techbeacon.com/one-year-using-kubernetes-production-lessons-learned



## TL;DR

- テスト環境やCI環境のように使い捨てる環境ではコンテナ内のデータベースは優れたツールだが、実稼働環境ではデータベースとコンテナが混在するべきではない。


ステートフルサービス、特にデータベースは、信頼できるメディア（ほとんどの場合はディスク（ssdまたは回転錆））で状態を維持することに依存しています。また、ほとんどの場合、データベースのパフォーマンスはシステムパフォーマンスに影響を与える大きな問題です。これは部分的には設計によるものであり、また部分的にはリレーショナルデータベースが水平方向に拡張できないことによるものです。

簡単に言うと、データベースで最も一般的なボトルネックはディスクのパフォーマンスです。優れたデータベースパフォーマンスを得るためには、ディスクパフォ​​ーマンスが必要です。これは、予測可能な待ち時間と優れたスループットを意味します。これは通常、プールの外ではなく、QoS付きの専用ボリュームを意味します。

他の問題は、大量のメモリを使用しているデータベースと関係があります。また、新しいインスタンスをすばやくスピンアップおよびスピンダウンすることは、次のような理由でうまくいきません。データセット全体をコピーする必要があり、時間がかかります。

これらすべてをまとめると、リレーショナルデータベースはコンテナ内で実行するのに非常に悪い候補になります - それはただ意味がありません。大量のデータが上下に回転している場合、開始するとすべてに時間がかかり、多くの時間がかかります。


```
$ kubectl create namespace my-test
$ kubectl config set-context $(kubectl config current-context) --namespace=my-test
$ kubectl config view | grep namespace
```


### volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  storageClassName: my-mysql
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /share/mysql
```

ポッドのボリュームにPersistenVolumeは紐付けず、PersistenVolumeClaims経由で紐付けを行う。これらは `.spec.storageClassName` で紐付けが行われる。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: my-mysql
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```
$ kubectl get pv,pvc
```

#### おまけ：persistentVolumeをマウントしたpodをデプロイするシーン

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: mydata
      mountPath: /usr/share/nginx/html
  volumes:
  - name: mydata
    persistentVolumeClaim:
      claimName: mysql-pv-claim
```

```
$ kubectl exec my-nginx-pod -ti bash
root@my-nginx-pod:/# df -h
```

### secret

SecretはKubernetesコンポーネントが呼び出せる、パスワードや鍵などの情報を扱うオブジェクト。 `.data` 以下はKey-Valueの形で記述し、ValueはBase64エンコードしておく必要がある。

`root_password` はユーザー名rootのパスワード。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dbsecret
type: Opaque
data:
  user: dXNlcg== # user
  password: cGFzc3dvcmQ= # password
  root_password: cm9vdF9wYXNzd29yZA== # root_password
```

kubectlからも中身は見れない。

```
$ kubectl describe secret dbsecret
...
Data
====
password:       8 bytes
root_password:  8 bytes
user:           4 bytes
```

### deployment


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-mysql-deployment
spec:
  selector:
    matchLabels:
      app: my-mysql-sample
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: my-mysql-sample
    spec:
      containers:
      - name: mysql
        image: mysql
        env:
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: password
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: root_password
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: mydata
          mountPath: /var/lib/mysql
      volumes:
      - name: mydata
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

mysqlのサービスを設定

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-mysql-svc
spec:
  ports:
  - port: 3306
  selector:
    app: my-mysql-sample
  clusterIP: None
```

「-p」オプションを使ってパスワードを指定する場合は、「-p」とパスワードの間に空白をいれずに指定します。

```
$ kubectl run -it --rm --image=mysql:8.0.3 --restart=Never mysql-client -- mysql -h my-mysql-svc -u user -ppassword
```

```
$ minikube ssh
$ ls /share/mysql
'#innodb_temp'	 binlog.000002	 ca.pem		   ib_buffer_pool   ibdata1   mysql.ibd		   public_key.pem    sys
 auto.cnf	 binlog.index	 client-cert.pem   ib_logfile0	    ibtmp1    performance_schema   server-cert.pem   undo_001
 binlog.000001	 ca-key.pem	 client-key.pem    ib_logfile1	    mysql     private_key.pem	   server-key.pem    undo_002
```


### アプリケーションのデプロイ

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app-pods
  ports:
  - port: 3000
  clusterIP: None
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app-pods
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: my-app-pods
    spec:
      containers:
      - name: nginx
        image: nginx
        env:
        - name: DB_HOST
          value: my-mysql-svc
        - name: DB_PORT
          value: '3306'
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: user
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dbsecret
              key: password
        - name: DB_DATABASE
          value: sample-table
        ports:
        - name: nginx
          containerPort: 3000
```
