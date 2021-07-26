## Node.js による Web Service を K8s 上に立てる
### 前提条件
- Docker Hub に自分のアカウントが存在し使い方を知っている
- Docker Images を Deploy できる K8s の環境
### 1. Node.js と Express をインストール
```
apt update
apt install nodejs
npm install -g express-generator
```
### 2. Project を作成する
```
epress webservice
cd webservice
```
### 3. index.js の作成
```
const express = require('express')
const app = express()
const port = process.env.PORT || 3000

app.get('/', (req, res) => {
  res.json({
    message: 'Hello Sample Web Service',
  });
})

app.listen(port, () => console.log(`Web Server listening on port ${port}!`))
```
### 4. package.json の編集
以の内容を追加する
```
{
  ...
  "dependencies": {
    "express": "^4.16.3"
  }
  ...
}
```
### 5. Dockerfile の作成
```
FROM node:8.12.0-alpine
ENV NODE_ENV=development
ARG project_dir=/app/
WORKDIR /app/
ADD index.js $project_dir
ADD package.json $project_dir
ADD package-lock.json $project_dir
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
```
### 6. Docker Image の作成
- &lt;your docker repo&gt; は自分の環境に書き換えること

```
docker build --no-cache -t <your docker repo>:v1 .
```
### 7. Docker Image を Docker Hub にアップロード
- &lt;your docker repo&gt; は自分の環境に書き換えること
```
docker login
docker push <your docker repo>:v1
```
### 8. deployment.yaml の作成
- &lt;your docker repo&gt; は自分の環境に書き換えること
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
  labels:
    app: node-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
      - name: node-app
        image: <your docker repo>:v1
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 3000
```
### 9. service.yaml の作成
```
apiVersion: v1
kind: Service
metadata:
  name: node-app-service
spec:
  type: LoadBalancer
  selector:
    app: node-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
```
### 10. K8s に Deploy する
```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```
### 11. 動作確認
```
kubectl get svc
```
<pre>
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP      10.96.0.1      &lt;none&gt;        443/TCP        6d
node-app-service   LoadBalancer   10.103.7.173   &lt;pending&gt;     80:30968/TCP   143m
</pre>
CLUSTER-IP は自分の環境に読み替える
```
curl http://10.103.7.173:80
```
<pre>
{"message":"Hello Sample Web Service"}
</pre>
