## YAML の書き方
### 1. Pod マニフェスト
- Pod とは
  - K8s が管理する最小構成単位
  - 1つ以上の Continer から構成
  - Cluster IP が Pod 単位で払い出される
  - Pod 内の Resources は同一の Node に Deploy される
  - Pod を定義したマニフェストから生成される Pod は１つ

- 必須フィールドは(*)で示す
```
apiVersion: v1          # (*) 利用する Kubernetes API のバージョン
kind: Pod               # (*) どの種類のオブジェクトを作成するのかを指定
metadata:               # (*) オブジェクトを一意に特定するための情報
  name: sample-pod      # (*) リソース名（任意）他に UID, namespece が指定可能
                        # kucectl get po の NAME で表示される名前
spec:                   # (*) オブジェクトの望ましい状態（Desired State）
# これより以下の内容は kind 毎に異なる。ここではコンテナを指定している
  containers:           # コンテナを指定
  - name: nginx         # コンテナの名前
    image: nginx:1.13   # コンテナイメージの指定
    env:                # コンテナで利用される環境変数
    - name: BACKEND_HOST
      value: localhost:8080
    ports:              # EXPOSEするポートの指定
    - containerPort: 80
```
- kucectl get po の実行結果
<pre>
NAME              READY   STATUS    RESTARTS   AGE
sample-pod        1/1     Running   0          32m
</pre>
- 複数のコンテナを定義する（ここでは alpine）
```
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.13
    env:
    - name: BACKEND_HOST
      value: localhost:8080
    ports:
    - containerPort: 80
  - name: alpine          # ２つ目のコンテナを指定
    image: alpine         # alpine を指定 
    command: ["/bin/sh"]  # 実行コマンドを指定
    args: ["-c", "while true; do date; sleep 10;done"]  # 引数
```
- kucectl get po の実行結果。READY が 2/2 となり 2 Pods 稼働していることが分かる
<pre>
NAME              READY   STATUS    RESTARTS   AGE
sample-pod        2/2     Running   0          32m
</pre>
### 2. ReplicaSet のマニフェスト
- ReplicaSet とは
  - 同一仕様の Pod を複数生成・管理
  - Pod 障害時指定した数になるよう自動回復
  - Cluster 内で管理する Pod の追跡に Label を利用

```
apiVersion: apps/v1
kind: ReplicaSet          # ReplicaSet を定義
metadata:
  name: sample-rs         # kubectl get rs で表示される毎小
Spec:
  replicas: 3             # 複製する Pod の数
   selector:
    matchLabels:          # ReplicaSet管理対象label
      app: sample-pod     # 複製する対象の Pod 名
  template:               # template 以下は Pod マニフェストと同じ
    metadata:
      labels:             # Pod 自身の名称
        app: sample-pod
    spec:                 # Desired State
      containers:
         ・・・
```
- kubectl get rs の実行結果
<pre>
NAME        DESIRED   CURRENT   READY   AGE
sample-rs   3         3         3       25m
</pre>
ReplicaSet によって Lavel で指定された Pod を複製する数を指定する。つまり ReplicaSet は Pod の数を管理する。
- kubectl get po の実行結果。3 Pods 複製されていることが確認できる。
<pre>
sample-pod        2/2     Running   0          32m
sample-rs-4h66n   2/2     Running   0          2m34s
sample-rs-ckpw9   2/2     Running   0          2m34s
sample-rs-n9swz   2/2     Running   0          2m34s
</pre>
- それぞれの Pod が何処に配置されたのかを確認する。まず初めに Worker Nodes の名称を確認しておく。
<pre>
NAME       STATUS   ROLES                  AGE   VERSION
master01   Ready    control-plane,master   22h   v1.21.2
master02   Ready    control-plane,master   22h   v1.21.3
master03   Ready    control-plane,master   22h   v1.21.3
worker01   Ready    <none>                       22h   v1.21.2
worker02   Ready    <none>                       22h   v1.21.3
worker03   Ready    <none>                       22h   v1.21.2</pre>
Woeker Nodes の名前は worker01, worker02, worker03 だということが分かる。
次に調査対象の Pod 名を取得
<pre>
root@master01:/home/ubuntu/workspace# kubectl get po
NAME              READY   STATUS    RESTARTS   AGE
sample-pod        2/2     Running   0          40m
sample-rs-4h66n   2/2     Running   0          10m
sample-rs-ckpw9   2/2     Running   0          10m
sample-rs-n9swz   2/2     Running   0          10m
</pre>
それぞれ sample-rs-XXXX だと分かる。実際に確認してみる。
<pre>
root@master01:/home/ubuntu/workspace# kubectl describe po sample-rs-4h66n | grep worker
Node:         worker03/192.168.0.113
  Normal  Scheduled  12m   default-scheduler  Successfully assigned default/sample-rs-4h66n to worker03
</pre>
Pod sample-rs-4h66n は woeker03 に配置されている。
<pre>
root@master01:/home/ubuntu/workspace# kubectl describe po sample-rs-ckpw9 | grep worker
Node:         worker02/192.168.0.112
  Normal  Scheduled  13m   default-scheduler  Successfully assigned default/sample-rs-ckpw9 to worker02
</pre>
Pod sample-rs-ckpw9 は woeker02 に配置されている。
<pre>
root@master01:/home/ubuntu/workspace# kubectl describe po sample-rs-n9swz | grep worker
Node:         worker01/192.168.0.111
  Normal  Scheduled  14m   default-scheduler  Successfully assigned default/sample-rs-n9swz to worker01
</pre>
Pod sample-rs-n9swz は woeker01 に配置されている。それぞれが満遍なく配置されている事が確認できる。
### 3. Deployment のマニフェスト
- Deployment とは
  - ReplicaSet を操作・管理する
  - ReplicaSet の世代管理を可能にする
    - リビジョンを使って Rollout が可能
    - Deployment の編集によって ReplicaSet が更新される 

```
apiVersion: apps/v1
kind: Deployment          # Deploymentに を指定
metadata:
  name: simple-ds         # kubectl get deploy で表示される NAME
spec:                     # 以下は ReplicaSet リファレンスと同じ
  replicas: 3
  selector:
    matchLabels:
      app: simple 
  template:
    ...
```
- kubectl get deploy の実行結果
```
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
simple-ds   3/3     3            3           3m49s
```
- Deploymenbt, ResultSet, Pods を一度に表示する
```
root@master01:/home/ubuntu/workspace# kubectl get deploy,rs,po
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/simple-ds   3/3     3            3           5m29s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/simple-ds-77fff76bdb   3         3         3       5m29s

NAME                             READY   STATUS    RESTARTS   AGE
pod/simple-ds-77fff76bdb-7gfgw   2/2     Running   0          5m29s
pod/simple-ds-77fff76bdb-dw257   2/2     Running   0          5m29s
pod/simple-ds-77fff76bdb-ptj5m   2/2     Running   0          5m29s
```
- Deployment . ResultSet > Pod という抱合関係になる
- 実運用で扱う構成単位
