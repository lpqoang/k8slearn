## springboot项目yaml模板

```yaml
kind: Namespace
apiversion: v1
metadata:
  name: hello
---
kind: Service
apiVersion: v1
metadata:
  name: spring-boot-helloworld
  namespace: hello
  labels:
    app: spring-boot-helloworld
spec:
  type: NodePort
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
  selector:
    app: spring-boot-helloworld
---
kind: Deployment
apiVersion: app/v1
metadata:
  name: spring-boot-helloworld
  namespace: hello
  labels:
    app: spring-boot-helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-boot-helloworld
  template:
    metadata:
      name: spring-boot-helloworld
      labels:
        app: spring-boot-helloworld
    spec:
      containers:
      - name: spring-boot-helloworld
        image: '__IMAGE__'
        ports:
        - name: http
        containerPort: 80
        protocol: TCP
```

