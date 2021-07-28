## Node.js による Web Service を K8s 上に立てる
### 前提条件
- Docker Hub に自分のアカウントが存在する
- Docker Images を Deploy できる K8s の環境
### 1. Node.js と npm をインストール
```
apt update
apt install nodejs npm
```
### 2. Project Directry を作成する
```
mkdir webservice
cd webservice
```
### 3. package.json の作成
```
{
  "private": true,
  "repository": "",
  "name": "docker_web_app",
  "version": "1.0.0",
  "description": "Node.js on Docker",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.16.1"
  }
}
```
### 4. server.js の確認
```
'use strict';

const express = require('express');

// Constants
const PORT = 8080;
const HOST = '0.0.0.0';

// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hello World');
});

app.listen(PORT, HOST);
console.log(`Running on http://${HOST}:${PORT}`);
```
### 5. Dockerfile の作成
```
FROM node:12
WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .
EXPOSE 8080
CMD [ "node", "server.js" ]
```
### 6. .dockerignore の作成
```
node_modules
npm-debug.log
deployment.yaml
service.yaml
```
### 7. Docker Image の作成
- &lt;your docker repo&gt; は自分の環境に書き換えること
```
docker build -t <your docker repos>:v1 .
```
### 8 動作確認
- &lt;your docker repo&gt; は自分の環境に書き換えること
```
 docker run -p 80:8080 -d ninja12r/webservice:v2
 curl http://localhost:80
```
<pre>
Hello World
</pre>
### 9. Docker Hub にアップロード
```
docker login
docker push <your docker repos>:v1
```
### 10. deployment.yaml の作成
- &lt;your docker repo&gt; は自分の環境に書き換えること
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: <your docker repo>:v1
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 8080
```
### 11. service.yaml の作成
```
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```
### 12. K8s に Deploy する
```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```
### 13. 動作確認
```
kubectl get svc,po
```
<pre>
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP      10.96.0.1      &lt;none&gt;        443/TCP        6d
web-app-service    LoadBalancer   10.103.7.173   &lt;pending&gt;     80:30968/TCP   143m

NAME                           READY   STATUS    RESTARTS   AGE
pod/web-app-7fcff8564d-77zmz   1/1     Running   0          15h
pod/web-app-7fcff8564d-95g9l   1/1     Running   0          15h
pod/web-app-7fcff8564d-x2l9c   1/1     Running   0          15h
</pre>
CLUSTER-IP は自分の環境に読み替える
```
curl http://<CLUSTER-IP>:80
```
<pre>
Hello World
</pre>
