# kubernetes-practice

-------------------------------------------------Kubernetes-------------------------------------------

----------------------------------------Node Selector ---------------------------------

This is used when we have to depoy the pod or replicas of pod in one particular node.

* kubectl label node nodename key=value -- to label a node

apiVersion: v1
kind: Pod
metadata:
  name: node-selector
spec:
  containers:
  - name: my-nginx
    image: nginx
    ports:
    - containerPort: 80


--------------------------------------- taint -------------------------------------------------

* kubectl taint node nodename key=value:NoSchedule

The command is used to taint a node. When a node is tainted no new pods can be deployed on the particular node

* kubectl taint node nodename key=value:NoSchedule-

---------------------------------------- tolerance ---------------------------------------------

Tolerance is used when we need to run a pod on tainted node, when taint and tolerance matches pod get deployed on that particular node.

apiVersion: v1
kind: Pod
metadata:
  name: node-selector
spec:
  containers:
  - name: my-nginx
    image: nginx
    ports:
    - containerPort: 80
    tolerations:
      - key:
        value:
        effect: NoSchedule
        operator: Equal


------------------------------------------- Daemon set ---------------------------------------------------------

It is a type of workload controller that ensures that copy of pod is running on each node. when a new node is added daemon controller will automatically 
schedule pod on that node, if a node is removed corresponding pod is also terminated.

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemon-set
  labels:
    name: test
spec:
  selector:
    matchLabels:
      name: test
  template:
    metadata:
      labels:
        name: test
    spec:
      containers:
      - name: my-nginx
        image: nginx

--------------------------------------------------- Requests and limits ----------------------------------------

* Request is asection in k8s where a pod can be created with maximum of given memory and cpu.

* limits limit is something where a pod can ask for maximum memory and cpu and cannot ask beyond that limit.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: testing
  labels:
    name: test
spec:
  selector:
    matchLabels:
      name: test
  template:
    metadata:
      labels:
        name: test
    spec:
      containers:
      - name: my-nginx
        image: nginx
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi

----------------------------------------------- liveness probe   --------------------------------------------------

* liveness probe is used to check if the container is till running correctly. if the probe fails kubernetes will automatically reatsrt conatiner to recover application.
  In below exmaple im using nginx image, so liveness probe sends http request to endpoint to determine pod is still running and healthy.


apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: / # The endpoint to hit for the probe 
          port: 80 
        initialDelaySeconds: 15 
        periodSeconds: 10 

-----------------------------------------------  Readness probe --------------------------------------------------

* The readness probe is used to determine if conatiner is raedy to handle incoming traffic, if it fails the conatiners will be removed from the list of endpoints
  that receive traffic.

apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: / 
          port: 80 
        initialDelaySeconds: 5 
        periodSeconds: 5

-------------------------------------------------  startup probe -------------------------------------------
* This is used to know when a conatiner application has started.

apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
      startupProbe:
        httpGet:
          path: / # The endpoint to hit for the probe 
          port: 80 
        initialDelaySeconds: 10 
        periodSeconds: 5 

----------------------------------------------------  secrets ---------------------------------------
* in k8s secrests is used to manage sensitive information such as API keys, passwords, certificates that shold key secrue.

1. secrets.yml (withoutencoding)

apiVersino: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque'
stringData:
  username: root
  password: test

2. secrets. (with encoding)

apiVersino: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque'
data:
  password: encoded value

3. deployment.yml


apiVersion: apps/v1
kind: Deployment
metadata:
  name: testing
  labels:
    name: test
spec:
  selector:
    matchLabels:
      name: test
  template:
    metadata:
      labels:
        name: test
    spec:
      containers:
      - name: my-nginx
        image: nginx
        env:
          - name: MYSQL_ROOT_USER
            valueFrom:
            secretKeyRef:
              name: my-secret
              value: username
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
            secretKeyRef:
              name: my-secret
              value: password

------------------------------------------------------Horizontal pod scaler ----------------------------------------------------
*  when load increases on conatiner beyond 50 kubernetes automatically scales maxreplicas mentioned in maifest files

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: 
spec:
  maxReplicas: 5
  minReplicas: 2
  scaleTragetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-dep
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 50
 
---------------------------------------------------------  stateful set and headless service -----------------------------------

* each pod in statefulset is assigned with unique host name, it remains same even if pod is restarted or rescheduled.
  when a statefulset is created, kubernetes automaically creates headless service. each pod is assigned a uinque DNS entry.
 $(pod-name).$(service-name).$(name-space).svc.cluster.local
1. statefulset 

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset
  labels:
    run: test
spec:
  serviceName: my-headless
  replicas: 3
  selector:
    matchLabels:
      run: test
  template:
    metadata:
      labels:
        run: test
    spec:
      containers:
      - name: my-sql
        image: mysql:latest
        ports:
        - containerPort: 3306
2. headless service

apiVersion: v1
kind: Service
metadata:
  name: my-headless
spec:
  clusterIP: None
  selector:
    name: my-statefulset
  ports:
    - port: 3306
      targetPort: 3306

-------------------------------------------------------------  persistent volume claim  ------------------------------------------------

1. storage class 

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer

2.  persistent volume claim

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi
3.  deployment

#04-mysql-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate 
  template: 
    metadata: 
      labels: 
        app: mysql
    spec: 
      containers:
        - name: mysql
          image: mysql:5.6
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: dbpassword11
          ports:
            - containerPort: 3306
              name: mysql    
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql    
      volumes: 
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: claim-name

--------------------------------------- Ingress controller (nginx ingress controller) --------------------------------------
Ingress:

 Ingress is an object that provides external access to services within cluster. it allows you to define rules for routing traffic based on hostname, path

IngressController:

 This is responsible for implementing rules defined in ingress resource.

Ingress resource:
 
 It is the place to define rules for routing external traffic to service within cluster. routes traffic based on hostname, path, routing rules.

1. application deployment 1

apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - name: httpd
        image: httpd
        ports:
        - containerPort: 80
        env:
        - name: TITLE
          value: "APACHE APP2"
---
apiVersion: v1
kind: Service
metadata:
  name: httpd
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: httpd

2. application deployment 2

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        env:
        - name: TITLE
          value: "NGINX APP1"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: nginx

3. ingress-resource 

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k8s-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /nginx(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
      - path: /httpd(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: httpd
            port:
              number: 80
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80










