# 環境変数の扱い


##コンテナへの環境変数の設定方法

Dockerfile でのENV設定よりも、kubernetesのマニフェストが優先されます。


1. Dockerfile 内の ENV で設定

```
FROM node:latest
WORKDIR /usr/src/app

COPY . ./

RUN npm install 


COPY . .
EXPOSE 8080

ENV HELLO="Hello from Dockerfile"

CMD ["node", "index.js"]
```

2. Dockerで動かす場合は、-e or --env オプションで指定
```
docker run -d -e HELLO="Hello World" --rm --name node -p 8080:8080  nodehello 
```

3. kubernetes のマニフェストで指定

```
---
apiVersion : apps/v1
kind: Deployment
metadata:
  name: nodehelloenv
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodehelloenv
  template:
    metadata:
      labels:
        app: nodehelloenv
    spec:
       containers:
        - name: nodehelloenv
          image: akubicharmyf.azurecr.io/nodehello
          ports:
          - containerPort: 8080
          env:
          - name: HELLO
            value: "Hello from Manifest"
```
