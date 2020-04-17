# Docker 이미지 생성
./mvnw com.google.cloud.tools:jib-maven-plugin:build -Dimage=gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v1

# Docker 이미지 실행
docker run -ti --rm -p 8080:8080 gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v1

# K8s 로그인
gcloud container clusters get-credentials baik-cluster-1 --zone asia-east1-a

# yml 빌더
https://static.brandonpotter.com/kubernetes/DeploymentBuilder.html

$GOOGLE_CLOUD_PROJECT 는 Google Colud Platform 의 각자의 프로젝트 ID 확인

deployment.yml
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: spring-boot-example
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: spring-boot-example
    spec:
      containers:
        - name: spring-boot-example
          image: 'gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v1'
          ports:
            - containerPort: 8080
```

service.yml
```
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-example
  labels:
    name: spring-boot-example
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: spring-boot-example
  type: LoadBalancer
```

# yml 적용 및 삭제
kubectl apply -f deployment.yml
kubectl apply -f service.yml

kubectl delete -f deployment.yml
kubectl delete -f service.yml

# Deployment 전략
* Recreate
* RollingUpdate
* Blue/Green
* Canary

## 버전 v1 -> v2 변경
./mvnw com.google.cloud.tools:jib-maven-plugin:build -Dimage=gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v2

## Recreate Deployment

$GOOGLE_CLOUD_PROJECT 는 Google Colud Platform 의 각자의 프로젝트 ID 확인

kube.yml
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: spring-boot-example
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: spring-boot-example
    spec:
      containers:
        - name: spring-boot-example
          image: 'gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v1'
          ports:
            - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-example
  labels:
    name: spring-boot-example
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: spring-boot-example
  type: LoadBalancer
```

### kube.yml 변경

$GOOGLE_CLOUD_PROJECT 는 Google Colud Platform 의 각자의 프로젝트 ID 확인

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: spring-boot-example
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: spring-boot-example
    spec:
      containers:
        - name: spring-boot-example
          image: 'gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v2'
          ports:
            - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-example
  labels:
    name: spring-boot-example
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: spring-boot-example
  type: LoadBalancer
```

### v2 적용
kubectl apply -f kube.yml

## RollingUpdate deployment

### kube.yml 변경

$GOOGLE_CLOUD_PROJECT 는 Google Colud Platform 의 각자의 프로젝트 ID 확인

kube.yml
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: spring-boot-example
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: spring-boot-example
    spec:
      containers:
        - name: spring-boot-example
          image: 'gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v1'
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /lazy
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-example
  labels:
    name: spring-boot-example
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: spring-boot-example
  type: LoadBalancer
```

### v2 적용
kubectl apply -f kube.yml

## Blue/Green Deployment

$GOOGLE_CLOUD_PROJECT 는 Google Colud Platform 의 각자의 프로젝트 ID 확인

kube-b-v1.yml
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: spring-boot-example-v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: spring-boot-example
        version: "v1"
    spec:
      containers:
        - name: spring-boot-example
          image: 'gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v1'
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /lazy
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            
---
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-example
  labels:
    name: spring-boot-example
    version: "v1"
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: spring-boot-example
    version: "v1"
  type: LoadBalancer
```

kube-g-v2.yml
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: spring-boot-example-v2
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: spring-boot-example
        version: "v2"
    spec:
      containers:
        - name: spring-boot-example
          image: 'gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v2'
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /lazy
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
---

apiVersion: v1
kind: Service
metadata:
  name: spring-boot-example-green
  labels:
    name: spring-boot-example-green
    version: "v2"
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: spring-boot-example
    version: "v2"
  type: LoadBalancer
```

kube-b-v2.yml
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: spring-boot-example-v2
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: spring-boot-example
        version: "v2"
    spec:
      containers:
        - name: spring-boot-example
          image: 'gcr.io/$GOOGLE_CLOUD_PROJECT/spring-boot-example:v2'
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /lazy
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-example
  labels:
    name: spring-boot-example
    version: "v2"
spec:
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: spring-boot-example
    version: "v2"
  type: LoadBalancer
```

### v2 적용
kubectl apply -f kube-g-v2.yml
kubectl apply -f kube-b-v2.yml
kubectl delete -f kube-g-v2.yml
