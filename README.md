# DevOps_Challenge
this is devops challenge for dockerizng ruby application on K8s:

 after follow this guide [dockerizing-a-ruby-on-rails-application](https://semaphoreci.com/community/tutorials/dockerizing-a-ruby-on-rails-application)

#### This is docker-compose.yaml after modification :
```
version: "2"

services:
  postgres:
    image: postgres:9.4.5
    environment:
      POSTGRES_USER: drkiq
      POSTGRES_PASSWORD: yourpassword
    ports:
      - '5432:5432'
    volumes:
      - drkiq-postgres:/var/lib/postgresql/data

  redis:
    image: redis:3.0.5
    ports:
      - '6379:6379'
    volumes:
      - drkiq-redis:/var/lib/redis/data

  drkiq:
    build: .
    image: sherifkhedr/drkiq:tag
    links:
      - postgres
      - redis
    volumes:
      - drkiq-sherif:/drkiqs
    ports:
      - '8000:8000'
  sidekiq:
    build: .
    command: bundle exec sidekiq -C config/sidekiq.yml
    image: sherifkhedr/sidekiq:tag
    links:
      - postgres
      - redis
    volumes:
      - drkiq-sherif:/drkiqs
```
##### Translate a Docker Compose File to Kubernetes Resources
```
$ mkdir export
$ kompose convert --file docker-compose.yml -o export/
```
and this is the result into export dir
```
 drkiq-claim0-persistentvolumeclaim.yaml
 drkiq-deployment.yaml
 drkiq-drkiq-persistentvolumeclaim.yaml
 drkiq-postgres-persistentvolumeclaim.yaml
 drkiq-redis-persistentvolumeclaim.yaml
 drkiq-service.yaml
 drkiq-sherif-persistentvolumeclaim.yaml
 postgres-deployment.yaml
 postgres-service.yaml
 redis-deployment.yaml
 redis-service.yaml
 sidekiq-claim0-persistentvolumeclaim.yaml
 sidekiq-deployment.yaml
 sidekiq-service.yaml
```

#### Configure a Pod to Use a ConfigMap

```
$ kubectl create configmap config-env --from-env-file=.drkiq.env
$ kubectl get configmap config-env -o yaml

apiVersion: v1
data:
  CACHE_URL: redis://redis:6379/0
  DATABASE_URL: postgresql://drkiq:yourpassword@postgres:5432/drkiq?encoding=utf8&pool=5&timeout=5000
  JOB_WORKER_URL: redis://redis:6379/0
  LISTEN_ON: 0.0.0.0:8000
  SECRET_TOKEN: asecuretokenwouldnormallygohere
  WORKER_PROCESSES: "1"
kind: ConfigMap
metadata:
  creationTimestamp: 2018-09-09T22:49:33Z
  name: config-env
  namespace: default
  resourceVersion: "27439"
  selfLink: /api/v1/namespaces/default/configmaps/config-env
  uid: 9eebb8db-b482-11e8-872c-080027c7146b
```
#### define configmap in pods as environmrnt variables in drkiq pod
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    kompose.cmd: kompose convert --file docker-compose.yml -o export/
    kompose.version: 1.1.0 (36652f6)
  creationTimestamp: null
  labels:
    io.kompose.service: drkiq
  name: drkiq
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        io.kompose.service: drkiq
    spec:
      containers:
      - image: sherifkhedr/drkiq:tag
        name: drkiq
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: config-env
        resources: {}
        volumeMounts:
        - mountPath: /drkiqs
          name: drkiq-sherif
      restartPolicy: Always
      volumes:
      - name: drkiq-sherif
        persistentVolumeClaim:
          claimName: drkiq-sherif
status: {}
```
#### define configmap in pods as environmrnt variables in sidekiq pod

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    kompose.cmd: kompose convert --file docker-compose.yml -o export/
    kompose.version: 1.1.0 (36652f6)
  creationTimestamp: null
  labels:
    io.kompose.service: sidekiq
  name: sidekiq
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        io.kompose.service: sidekiq
    spec:
      containers:
      - args:
        - bundle
        - exec
        - sidekiq
        - -C
        - config/sidekiq.yml
        image: sherifkhedr/sidekiq:tag
        name: sidekiq
        envFrom:
        - configMapRef:
            name: config-env
        resources: {}
        volumeMounts:
        - mountPath: /drkiqs
          name: drkiq-sherif
      restartPolicy: Always
      volumes:
      - name: drkiq-sherif
         persistentVolumeClaim:
          claimName: drkiq-sherif
status: {}
```
after that make deployement 
```
$ kubectl create -f export
```
