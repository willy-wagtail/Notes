# Kubernetes notes

1. [Inventory](#1-inventory)
2. [Connection settings and privilege escalation](#2-connection-settings-and-privilege-escalation)

## 1. Overview of kubernetes

- open source container orchestration tool
- born from google's experience running production workfloads at scale
- release faster and recover faster
- distributed system
- machines form a cluster, may be physical virtual on-prem or in cloud
- schedules containers on machines
- moves containers as machines added or removed
- can use different contaqiner runtimes
  - commonly used with docker
- declarative config
- deploy containers
- wire up networking
- scale and expose services
- auto recover from machine failures
- built in support for machine maintenance
- join clusters to form a federation
- auto deployment rollout and rollback
- seamless horizontal scaling
- secret management
- service discovery and load balancing
- linux and windows containers
- simple log collection
- stateful app suport
- persistent volume management
- CPU memory quotas
- batch job processing
- role-based access control
- surge in tools to support containers
- compare k8 with other tools like DCOS, AmazonECS, and socker swarm mode

DCOS - data center OS
- ppools compute into uniform task pool
- supports many different types of workloads
- attractive to orgs not only using containers
- includes package manager to deploy popular systems
- even run k8 on DCOS

Amazon ECS
- elastic container service
- pools of compute resources
- api calls to orchestrate containers
- EC2 compute managed by you of by AWS Fargate
- only available in AWS
- useful if deep into AWS ecosystem

Docker Swarm
- cluster of multiple docker hosts
- works natively with docker command
- docker provides ful support of k8 to easily switch
- used by dockers enterprise edition

### 1.1 deploying k8

Single node k8 cluster support
- Docker desktop supports kubernetes.
- Minikube supports linux and windows and mac
- Kubeadm 

- useful for CI
- create clusters that start quickly and are pristine for testing apps in k8
- kubernetes-in-docker made for this use case

Multi node k8 clusters
- for prod workloads
- horizontal scaling
- tolerate node failures
- ask key Qs
  - hjow much control v effort in maintaining
    - fully managed sols free you from maintanance but often lag from the latest k8 releases 
    - new versions of k8every 3 months
    - fully managed: amazon eks, AKS azure, GKE google cloud
    - full control: kubespray, kops, kubeadm
  - do you have expertise with a particular cloud provider
    - tighter integration with cloud provider's other services 
  - do you need enterprise support
    - openshift by redhat, pivotal container service (PKS), and Rancher
  - are you concerned about vendor lock-in?
    - focus on open source solutions like kubespray and Ranger that can deploy k8 clusters to a wide variety of platforms
  - On-prem, in the cloud, or both? (less concerning question)
    - k8 provides abstraction of cluster of resources and the underlying nodes can be running in diff plaptforms.
    - k8 itself is at core of open source hybrid clouds
    - cloud vendor k8 solutions often allow using on-prem compute (e.g. GKE, AKS, EKS)
  - Linux, windows or both containers? (less concerning question)
    - to support linux containers you need linux nodes in your cluster
    - to support windows, you need windows nodes
    - both can exist in same cluster to supoort both clusters

### 1.2 k8 architecture

k8 has its own dialect
- cluster - all machines  
- nodes - machines in the cluster
- nodes are workers or masters
- worker nodes include software to run containers managed by the k8 control plane
- master nodes run the control plane
- control plane is a set of APIs and software that k8 users interact with
- APIs and software are referred to as master components

control plans schedules containers onto nodes
scheduling decisions consider required CPU and other factors
scheduling referes to the decision process of placing containers onto nodes according to compute requirements

k8 pods
- group of containers
- containers in pod runs on same node
- pods are the smallest building block in k8
- mode abstractions on top of pods

k8 services
- networking rules for exposing pods to other pods in cluster or internet

k8 deployments
- manage deploying config changes to running pods incl rollout and rollback
- horizontal scaling


### 1.3 interacting with k8

method 1
- rest api - not common. only when u are using language that doesnt have a k8 client lib

method 2
- client libs - handles auth and manage rest api requests & responses.
- k8 maintains official client libs for go, python, java, .net and js. Community managed ones are availabel for other languages

method 3 - most common
- k8 cli called ``kubectl``
- high level commands that translate to rest api
- local and remote access to nodes
- easy to understand command pattern

``kubectl``
- ``create`` - pods, services, etc
- ``delete`` 
- ``get`` - get list of resources of given type, e.g. ``kubectl get pods``
- ``describe`` - prints detailed info about a resource e.g. ``kubectl describe pod server``
- ``logs`` - to print container logs

method 4 - web dashbaord

## 2. Deploying containerised apps

### 2.1 Pods

pods
- basic building block 
- one or more containers (one for now until later in course)
- pod containers share a container network 
- one IP address per pod

pod declaration
- image
- ports
- restart policy
- resource limits
- others available

manifest file 
- declares desired properties
- describes all sorts of resources in k8
- the spec contains resource-specific properties
- advantageous over running commands and passing options, as they can be checked into source control
- manifests are sent to k8 api server
  - ``kubectl create`` sends to k8 API server
  - API service then
    - selects node with sufficient resources
    - schedules Pod onto the node
    - Node pulls Pod's container iamge
    - starts Pod's container

Demo
- ``kubectl get pods`` to confirm its connected to API server cluster
- create manifest yaml file
  ```
  # 1.1-basic_pod.yaml
  apiVersion: v1
  kind: Pod
  metadata: 
    name: mypod
  spec: 
    containers:
    - name: mycontainer
      image: nginx:latest
  ```
- ``cat 1.1-basic_pod.yaml``
- ``kubectl create -f 1.1-basic_pod.yaml`` - ``f`` option tells it that you are using manifest.yml
- ``kubectl get pods``
- ``kubectl describe pod | more`` gives more info

Add a port to the pod.
  ```
  # 1.2-port_pod.yaml
  apiVersion: v1
  kind: Pod
  metadata: 
    name: mypod
  spec: 
    containers:
    - name: mycontainer
      image: nginx:latest
      port:
        - containerPort: 80
  ```
- ``kubectl delete pod mypod`` - can also do ``-f 1.1-basid_pod.yaml`` to delete all pods using that manifest file
- ``kubectl create -f 1.2-port_pod.yaml``
- ``kubectl describe pod mypod | more`` to see the updated port
- ``curl 172.17.0.3:80`` to test the new port
  - still not working because the port is only part of the container network, and we're not part of it

Add a label to the pod.
```
# 1.3-labeled_pod.yaml
apiVersion: v1
kind: Pod
metadata: 
  name: mypod
  labels:
    app: webserver
spec: 
  containers:
  - name: mycontainer
    image: nginx:latest
    port:
      - containerPort: 80
```

Add resources requests and limits 
- resource requests sets minimum resources required to schedule onto a node. it wont be scheduled until the resources become available.
- resource limit sets the max resources the node gives the pod
```
# 1.4-resources_pod.yaml
apiVersion: v1
kind: Pod
metadata: 
  name: mypod
  labels:
    app: webserver
spec: 
  containers:
  - name: mycontainer
    image: nginx:latest
    resources:
      requests:
        memory: "128Mi" # 128Mi = 128 mebibytes
        cpu: "500m"     # 500m = 500 milliCPUs (1/2 CPU)
      limits:
        memory: "128Mi"
        cpu: "500m"     
    port:
      - containerPort: 80
```
- ``kubectl create -f 1.4-resources_pod.yaml``
- ``kubectl describe pod mypod | more`` - you can see QoS class as ``Guaranteed``
  - benchmarking to set reasonable request requests and limits to best utilize the resources in the cluster
  - for the rest of this course, we wil use ``best effort`` pods with no resource limits and requests, but this is not what you should do in production.

### 2.2 Services

Service defines networking rules for accessing pods in cluster and from internet
- use labels to select a group of pods
- service has a fixed IP address
- distribute requests a cross pods in the group to balance the load

manifest.yaml
  ```
  # 2.1-web_service.yaml
  apiVersion: v1
  kind: Service
  metadata: 
    name: webserver
    labels:
      app: webserver
  spec: 
    ports:
    - port: 80
    selector:
      app: webserver
    type: NodePort
  ```

``cat 2.1-web_service.yaml``
- kind is set to Service
- label is the same as the pod since the service is related to the pod. Not required, but good to stay organised
- selector defines labels to match pods against
- service must declare port mappings
- optional type - defines how to expose the service
  - allocation of port per node, chosen from available node ports, unless specified 
  - (not recommended to declare a port yourself as it'll fail to create if there's port conflicts)

``kubectl create -f 2.1-web_service.yaml``
- ``kubectl get services``
  - cluster IP is internal/private IP
- ``kubectl describe service webserver``
  - ports, and endpoints
- ``kubectl describe nodes | grep -i address -A 1``
  - nodes are resources in cluster just like pods and services
  - we just need the internal IPS
``curl 192.168.64.2:32337``
  - gives the output!

- Allows us to expose pods via static IP address
- NodePort services allows us access from outside the cluster

### 2.3 Multi-container pods

demo app:
4 containers across 3 tiers
app tier is nodejs container
redis data iter storing counter
poller and counter in the support tier
all containers configured using env vars

- namespaces separate resources according to users , envs, or applications
- role based access control (RBAC) to secure access per namespace
- using namespaces is a best practice


``kubectl create -f 3.1-namespace.yaml`` to create a namespace
- specify a namespace using ``-f`` option, where the yaml file is like below:
  ```
  # 3.1-namespace.yml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: microservice
    labels: 
      app: counter
  ```


``kubectl create -f 3.2-multi_container.yaml -n microservice``
- specify the namespace its in
- manifest file for pod
  ```
  # 3.2-multi_container.yaml
  apiVersion: v1
  kind: Pod
  metadata: 
    name: app
  space:
    containers:
      - name: redis
        image: redis:latest
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 6379
      
      - name: server
        image: lrakai/microservices:server-v1
        ports:
          - containerPort: 8080
        env:
          - name: REDIS_URL
            value: redis://localhost:6379

      - name: counter
        image: lrakai/microservices:counter-v1
        env:
          - name: API_URL
            value: http://localhost:8080
  ```
- can give namespace in manifest using ``metadata: name: app``. But makes manifest less portable as it cant be altered via the command line.
- ``latest`` tag means itll pull every time. Can specify imagePullPolicy to IfNotPresent so it doesnt pull everytime. If specific tag is given, kubernetes uses IfNotPresent by default.

``kubectl get -n microservice pod app``

``kubectl describe -n microservice pod app``

``kubectl logs -n microservice app counter --tail 10``

``kubectl logs -n microservice apppoller -f``

### 2.4 Service discovery

Services supports multi pod design
provides static endpoint for each tier
handles pod IP changes
load balances

two service discovery mechanisms
- env vars
  - services address automatically injected in container
  - environment variables follow name conventions based on service name
- DNS
  - DNS record auto created in cluster's DNS
  - containers automatically configured to query cluster DNS


``kubectl create -f 4.1-namespace.yml`` create a new namespace called service-discovery



Start by creating data-tier
- run ``kubectl create -f 4.2-data_tier.yaml -n service-discovery``
- We can separate resources using ``---``.
- ClusterIP creates an internal IP for inside the cluster for internal access only
- resources are created in order listed in the file
```
# 4.2-data_tier.yaml
apiVersion: v1
kind: Service
metadata:
  name: data-tier
  labels: 
    app: microservices
spec:
  ports:
  - port: 6379
    protocol: TCP # default
    name: redis # optional when only 1 port
  selector:
    tier: data
  type: ClusterIP # default
---
apiVersion: v1
kind: Pod
metadata:
  name: data-tier
  labels:
    app: microservices
    tier: data
spec:
  containers:
    - name:redis
```

``kubectl get pod -n service-discovery``

``kubectl describe service -n service-discovery data-tier``


Create app tier
- ``kubectl create -f 4.3-app_tier.yaml -n service-discovery``
- the service host variable name is the service name capitalised and hyphens replaced with underscores followed by SERVICE_HOST
- the port variable name is similar with SERVICE_PORT. You can prepend the name optionally.
  ```
  # 4.3-app_tier.yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: app-tier
    labels: 
      app: microservices
  spec:
    ports:
    - port: 8080
    selector:
      tier: app
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: app-tier
    labels:
      app: microservices
      tier: app
  spec:
    containers:
      - name: server
        image: lrakai/microservices:server-v1
        ports:
          - containerPort: 8080
        env:
          - name: REDIS_URL
            value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
  ```

Create support tier
- ``kubectl create -f 4.4-support_tier.yaml -n service discovery``
- Kubernetes adds DNS records for every service, following the pattern of <service_name>.<service_namespace>
- if services use same namespace, can simply use service name.
  ```
  # 4.4-support_tier.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: support-tier
    labels: 
      app: microservices
      tier: support
  spec:
    containers:
      - name: counter
        image: lrakai/microservices:counter-v1
        env:
          - name: API_URL
            # DNS for service discovery
            # Naming pattern:
            #   IP address: <service_name>.<service_namespace>
            #   Port: needs to be extracted from SRV DNS record
            value: http://app-tier.service-discovery:8080

      - name: poller
        image: lrakai/microservices:poller-v1
        env:
          - name: API_URL
            value: http://app-tier:$(APP_TIER_SERVICE_PORT)
  ```

``kubectl get pods -n service-discovery``

``kubectl logs -n service-discovery support-tier poller -f``

Conclusion
- services for N-tier applications as interfaces
- ClisterIP service for internal-only access
- environment variables and DNS allow pods to discover services
- environment variables for services available at Pod creation time and in the same namespace
- DNS dynamically updated and across namespaces

### 2.5 Deployments

Deployments represent multiple replicas of a pod
- describes a desired state that kubernetes needs to achieve
- deployment controller master component converges actual state to the desired state

We'll replace individual pods with deployments that manage the pods for us

``kubectl create -f 5.1-namespace.yaml`` create new namespace called deployments
- a deployment is a template for creating pods
- a template is used to create replicas
- a replicas is a copy of a pod
- application scales by creating mode replicas

  ```
  apiVersion: apps/v1 # apps API group
  kind: Deployment
  metadata:
    name: data-tier
    labels:
      app: microservices
      tier: data
  spec:
    replicas: 1
    selector:
      matchLabels:
        tier: data
    template:
      metadata:
        labels: 
          app: microservices
          tier: data
      spec: # pod spec
        containers:
        - name: redis
          image: redis:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 6379
  ```

``diff -y 4.2-data_tier.yaml 5.2-data_tier.yaml`` to see diff between previous yaml and the deployment yaml
- apiVersion is now apps/v1. higher level abstractions are at their own API group rather than part of the core API
- metadata of the pod is now applied to the Deployment
- deployment spec contains deployment specific information 
  - it also includes the pod spec which is the same as before
  - replicas = how many pods (1 id default)
  - selectors mapping - use label selectors to grouppods that are in the deployment with that declared in the pod template
  - template metadata doesnt need a name as kubernetes generates unique names for each pod


``kubectl create -n deployments -f 5.2-data-tier.yaml -f 5.3-app-tier.yaml -f 5.4-support-tier.yaml``

``kubectl get -n deployments deployments``

``kubectl -n deployments get pods``
- deployment adds a hash after the name

``kubectl scale -n deployments deployments support-tier --replicas=5`` editing the replicas to 5

``kubectl -n deployments get pods``
- we see 2/2. replicas replicate pods, rather tha container inside the pods.

``kubectl delete -n deployments pods support-tier-997bc57fb-mtfrf support-tier-123fd9u1nf support-tier-192nfirubfg``
- watch as kubernetes deletes, then brings back up pods to match the replica number

``kubectl scale -n deployments deployment app-tier --replicas=5``

``kubectl -n deployments get pods``

``kubectl desribe -n deployments service app-tier``
- endpoints, has 5 IPs, trackable due to labels


Conclusion
- deployments to manage pods in each tier
- kubernetes ensures actual state matches desired state
- ``kubectl scale`` can scale the number of replicas
- services seamlessly support scaling
- scaling is best with stateless pods

### 2.6 Autoscaling

scale auto based on CPU or custom metrics
set target CPU along wtih min and max replicas
target CPU is expressed as % of pod's CPU request

kube will inc or dec replicas based on cpu usage
compares actual with target cpu usage

autoscaling depends on metrics being collected
metrics server is one solution for collecting metrics
several manifest files are used to deploy metrics server
- github.com/kubernetes-incubator/metrics-server

pods must have CPU requests
horizontalPodAutoscaler hpa kind is configured with target CPU and min max replicas


``kubectl apply -f metrics-server`` point to metrics server directory

``kubectl top pods -n deployments`` to see CPU and memory usage


``kind: HorizontalPodAutoscaler``
- ``targetCPUUtilizationPercentage``
- ``scaleTargetRef``

``kubectl create -f 6.2-autoscale.yaml -n deployments``

``watch -n 1 kubectl get -n deployments deployment``

``kubectl api-resources``

``kubectl describe -n deployments hpa``

``kubectl get -n deployments hpa``

``kubectl edit -n deployments``

### 2.7 Rolling updates and rollbacks

Rollouts update deployments
any change to a deployments template triggers a rollout
different rollout strategy

rolling updates by default
update in groups rather than all-at-once
both old and new versions running at the same time
app needs to handle that
alternative is recreate strategy which kills all old first 
scaling is not rollout
kubectl has commands check pause resume and rollback rollouts

maxSurge = how many replicas over the desired number is allowed for rollout
maxUnavailable = how many old can be deleted without waiting for new pods to be ready

``kubectl rollout -n deplotments status deployment app-tier`` to see rollouts in real time

``... rollout pause ....`` 
``... rollout resume ...``
``... rollout undo ...``
``... rollout history ...``

### 2.8 Probes

Probes aka healthchecks

readiness probe
- used to check when pod is ready to serve traffic and handle requests
- useful after startup to check external dependencies
- readiness probes sets the pods ready condition. services only send traffic to ready pods

liveness probe
- detect when pod enteres a broken state
- kube will restart the pod for you
- declared in the same way as readiness probes

probes can be declared in a pods containers
all container probes must pass for a pod to pass
probe actions can be a command that runs in the container, an HTTP get request , or open a tcp socket
by default probes check containers every 10 seconds

### 2.9 Init containers

Way to run tasks or prerequisites which are required to complete before the main container starts.
separate init logic from container image

pods declare any number. they will run in order they are declared. init containers use their own images.
easy way to block or delay starting an app until useful task is first done.
runss every time pod is created/restarted.

add init container in our app tier that will wait for redis before starting any application servers

useful for checking preconditions and preparing dependencies.

### 2.10 Volumes

share data between containers in pod

lifetime of container file system is limited to container lifetime which leads to unexpected consequences if container  restarts

how to handle nonephemeral data
deploy a persistent data tier
options for storing data


two high level storage options - volumes and persistent volumes

used by mounting dir in one or more containers

pods can use many

diff is how lifecycle is managed

volumes tied to a pod and pod lifecycle
share data between containers and tolerate restarts
use for non durable storage, deleted with the pod
default volume type is emptyDir
data is lost if pod is rescheduled on a different node

persistent volumes independent of pods lifetime
pods claim persistent vol to use throughtout their lifetimes
mounted by multiple pods on diff nodes 

persistent vol claim (PVC) 
 descibe a pods request for persistent volume
 how much, type, access mode
 access mode , read write one, read only many,  read-write many
 PVA stays pending if no PV can satisfy. 
connected to a pod through a volume of type PVC

backed by wide vaiety of volume types
use PV for durable types
supported storage types: GCE. Azire, Amazon EBS, NFS


### 2.11 Config maps and secrets

pod spec config is less portable
sensitive info such as api keys is security issue

config maps and secrests
separate config from pod specs
results in more portable and easier to manage manifests
Secrets are for sensitive data
specialised types of secrets 
here will refer to generic 

data stored as key value pairs
pods reference config map and secretes
references made by mounting volumes or setting env vars

create config name space
yaml has kind ConfigMap, metadata name = redisu, and data: config: whatever config



## 3 Lab

### 3.2 Code along

#### 3.2.1 Pods

``cat 1.1-basic_pod.yaml`` 
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
  spec:
    containers:
    - name: mycontainer
      image: nginx:latest
  ```

``kubectl create -f 1.1-basic_pod.yaml``
``kubectl get pods``
``kubectl describe pod mypod | more``
``kubectl delete pod mypod``

``cat 1.2-port_pod.yaml``
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
  spec:
    containers:
    - name: mycontainer
      image: nginx:latest
      ports:
        - containerPort: 80
  ```

``kubectl create -f 1.2-port_pod.yaml``
``kubectl describe pod mypod | more``
``curl 192.168.242.2:80``
- doesnt work as pod is part of container network
``kubectl delete pod mypod``

``cat 1.3-labeled_pod.yaml``
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
    labels:
      app: webserver
  spec:
    containers:
    - name: mycontainer
      image: nginx:latest
      ports:
        - containerPort: 80
  ```

``cat 1.4-resources_pod.yaml ``
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
    labels:
      app: webserver
  spec:
    containers:
    - name: mycontainer
      image: nginx:latest
      resources:
        requests:
          memory: "128Mi" # 128Mi = 128 mebibytes
          cpu: "500m"     # 500m = 500 milliCPUs (1/2 CPU)
        limits:
          memory: "128Mi"
          cpu: "500m"
      ports:
        - containerPort: 80
  ```
``kubectl create -f 1.4-resources_pod.yaml``
``kubectl describe pod mypod | more``

#### 3.2.2 Services

``cat 2.1-web_service.yaml``
  ```
  apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: webserver
    name: webserver
  spec:
    ports:
    - port: 80
    selector:
      app: webserver
    type: NodePort
  ```

``kubectl create -f 2.1-web_service.yaml``
``kubectl get services``
``kubectl describe service webserver``
``kubectl describe nodes | grep -i address -A 1``
``curl 10.0.0.100:30731`` (replace #### with the actual port digits) (can curl the IP of any of the nodes)

#### 3.2.3 Multicontainer pods

``cat 3.1-namespace.yaml``
  ``` 
  apiVersion: v1
  kind: Namespace
  metadata:
    name: microservice
    labels:
      app: counter
  ```
``kubectl create -f 3.1-namespace.yaml``

``cat 3.2-multi_container.yaml`` 
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: app
  spec:
    containers:
      - name: redis
        image: redis:latest
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 6379
      
      - name: server
        image: lrakai/microservices:server-v1
        ports:
          - containerPort: 8080
        env:
          - name: REDIS_URL
            value: redis://localhost:6379

      - name: counter
        image: lrakai/microservices:counter-v1
        env:
          - name: API_URL
            value: http://localhost:8080

      - name: poller
        image: lrakai/microservices:poller-v1
        env:
          - name: API_URL
            value: http://localhost:8080
  ```

``kubectl create -f 3.2-multi_container.yaml -n microservice``
``kubectl get -n microservice pod app``
``kubectl describe -n microservice pod app``
``kubectl logs -n microservice app counter --tail 10``
``kubectl logs -n microservice app poller -f``

#### 3.2.4 Service discovery

``cat 4.1-namespace.yaml``
  ``` 
  apiVersion: v1
  kind: Namespace
  metadata:
    name: service-discovery
    labels:
      app: counter
  ```
``kubectl create -f 4.1-namespace.yaml``

``cat 4.2-data_tier.yaml``
  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: data-tier
    labels:
      app: microservices
  spec:
    ports:
    - port: 6379
      protocol: TCP # default 
      name: redis # optional when only 1 port
    selector:
      tier: data 
    type: ClusterIP # default
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: data-tier
    labels:
      app: microservices
      tier: data
  spec:
    containers:
      - name: redis
        image: redis:latest
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 6379
  ```
``kubectl create -f 4.2-data_tier.yaml -n service-discovery``
``kubectl get pod -n service-discovery``
``kubectl describe service -n service-discovery data-tier``

``cat 4.3-app_tier.yaml ``
  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: app-tier
    labels:
      app: microservices
  spec:
    ports:
    - port: 8080
    selector:
      tier: app
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: app-tier
    labels:
      app: microservices
      tier: app
  spec:
    containers:
      - name: server
        image: lrakai/microservices:server-v1
        ports:
          - containerPort: 8080
        env:
          - name: REDIS_URL
            # Environment variable service discovery
            # Naming pattern:
            #   IP address: <all_caps_service_name>_SERVICE_HOST
            #   Port: <all_caps_service_name>_SERVICE_PORT
            #   Named Port: <all_caps_service_name>_SERVICE_PORT_<all_caps_port_name>
            value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
            # In multi-container example value was
            # value: redis://localhost:6379 
  ```

``kubectl create -f 4.3-app_tier.yaml -n service-discovery``

``cat 4.4-support_tier.yaml`` 
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: support-tier
    labels:
      app: microservices
      tier: support
  spec:
    containers:

      - name: counter
        image: lrakai/microservices:counter-v1
        env:
          - name: API_URL
            # DNS for service discovery
            # Naming pattern:
            #   IP address: <service_name>.<service_namespace>
            #   Port: needs to be extracted from SRV DNS record
            value: http://app-tier.service-discovery:8080

      - name: poller
        image: lrakai/microservices:poller-v1
        env:
          - name: API_URL
            # omit namespace to only search in the same namespace
            value: http://app-tier:$(APP_TIER_SERVICE_PORT)
  ```
``kubectl create -f 4.4-support_tier.yaml -n service-discovery``
``kubectl get pods -n service-discovery``
``kubectl logs -n service-discovery support-tier poller -f``

#### 3.2.5 Deployments

``cat 5.1-namespace.yaml ``
  ```
  apiVersion: v1
  kind: Namespace
  metadata:
    name: deployments
    labels:
      app: counter
  ```
``kubectl create -f 5.1-namespace.yaml``

``cat 5.2-data_tier.yaml``
  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: data-tier
    labels:
      app: microservices
  spec:
    ports:
    - port: 6379
      protocol: TCP # default 
      name: redis # optional when only 1 port
    selector:
      tier: data 
    type: ClusterIP # default
  ---
  apiVersion: apps/v1 # apps API group
  kind: Deployment
  metadata:
    name: data-tier
    labels:
      app: microservices
      tier: data
  spec:
    replicas: 1
    selector:
      matchLabels:
        tier: data
    template:
      metadata:
        labels:
          app: microservices
          tier: data
      spec: # Pod spec
        containers:
        - name: redis
          image: redis:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 6379
  ```

``cat 5.3-app_tier.yaml ``
  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: app-tier
    labels:
      app: microservices
  spec:
    ports:
    - port: 8080
    selector:
      tier: app
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: app-tier
    labels:
      app: microservices
      tier: app
  spec:
    replicas: 1
    selector:
      matchLabels:
        tier: app
    template:
      metadata:
        labels:
          app: microservices
          tier: app
      spec:
        containers:
        - name: server
          image: lrakai/microservices:server-v1
          ports:
            - containerPort: 8080
          env:
            - name: REDIS_URL
              # Environment variable service discovery
              # Naming pattern:
              #   IP address: <all_caps_service_name>_SERVICE_HOST
              #   Port: <all_caps_service_name>_SERVICE_PORT
              #   Named Port: <all_caps_service_name>_SERVICE_PORT_<all_caps_port_name>
              value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
              # In multi-container example value was
              # value: redis://localhost:6379 
  ```

``cat 5.4-support_tier.yaml``
  ```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: support-tier
    labels:
      app: microservices
      tier: support
  spec:
    replicas: 1
    selector:
      matchLabels:
        tier: support
    template:
      metadata:
        labels:
          app: microservices
          tier: support
      spec:
          containers:

          - name: counter
            image: lrakai/microservices:counter-v1
            env:
              - name: API_URL
                # DNS for service discovery
                # Naming pattern:
                #   IP address: <service_name>.<service_namespace>
                #   Port: needs to be extracted from SRV DNS record
                value: http://app-tier.deployments:8080

          - name: poller
            image: lrakai/microservices:poller-v1
            env:
              - name: API_URL
                # omit namespace to only search in the same namespace
                value: http://app-tier:$(APP_TIER_SERVICE_PORT)
  ```

``kubectl create -n deployments -f 5.2-data_tier.yaml -f 5.3-app_tier.yaml -f 5.4-support_tier.yaml``
``kubectl get -n deployments deployments``
``kubectl -n deployments get pods``
``kubectl scale -n deployments deployment support-tier --replicas=5``
``kubectl -n deployments get pods``
``kubectl delete -n deployments pods support-tier-... support-tier-... --wait=false`` (You can use tab completion to display the possible values to replace ... with)
``watch -n 1 kubectl -n deployments get pods``
``kubectl scale -n deployments deployment app-tier --replicas=5``
``kubectl -n deployments get pods``
``kubectl describe -n deployments service app-tier``

#### 3.2.6 Autoscaling

``kubectl create -f metrics-server/``
``watch kubectl top pods -n deployments``

``cat 6.1-app_tier_cpu_request.yaml ``
  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: app-tier
    labels:
      app: microservices
  spec:
    ports:
    - port: 8080
    selector:
      tier: app
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: app-tier
    labels:
      app: microservices
      tier: app
  spec:
    replicas: 5
    selector:
      matchLabels:
        tier: app
    template:
      metadata:
        labels:
          app: microservices
          tier: app
      spec:
        containers:
        - name: server
          image: lrakai/microservices:server-v1
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 20m # 20 milliCPU / 0.02 CPU
          env:
            - name: REDIS_URL
              # Environment variable service discovery
              # Naming pattern:
              #   IP address: <all_caps_service_name>_SERVICE_HOST
              #   Port: <all_caps_service_name>_SERVICE_PORT
              #   Named Port: <all_caps_service_name>_SERVICE_PORT_<all_caps_port_name>
              value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
              # In multi-container example value was
              # value: redis://localhost:6379 
  ```

``kubectl create -f 6.1-app_tier_cpu_request.yaml -n deployments`` - already exist and fails
``kubectl apply -f 6.1-app_tier_cpu_request.yaml -n deployments`` - updates existing
``kubectl get -n deployments deployments app-tier``

``cat 6.2-autoscale.yaml ``
  ```
  apiVersion: autoscaling/v1
  kind: HorizontalPodAutoscaler
  metadata:
    name: app-tier
    labels:
      app: microservices
      tier: app
  spec:
    maxReplicas: 5
    minReplicas: 1
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: app-tier
    targetCPUUtilizationPercentage: 70

  # Equivalent to
  # kubectl autoscale deployment app-tier --max=5 --min=1 --cpu-percent=70
  ```

``kubectl create -f 6.2-autoscale.yaml -n deployments`` 
``watch -n 1 kubectl get -n deployments deployments app-tier``
``kubectl api-resources``
``kubectl describe -n deployments hpa``
``kubectl get -n deployments hpa``
``kubectl edit -n deployments hpa`` - change min replica from 1 to 2
``watch -n 1 kubectl get -n deployments deployments app-tier``

### 3.2.7 Rolling Updates and Rollbacks

``kubectl delete -n deployments hpa app-tier``
``kubectl edit -n deployments deployment app-tier`` - change replicas from 2 to 10
``watch -n 1 kubectl get -n deployments deployments app-tier ``
``kubectl edit -n deployments deployment app-tier`` - check out the strategy rollingUpdate maxSurge and maxUnavailable. Then change name from server to cloudacademy to make a non functional change.
``kubectl rollout -n deployments status deployment app-tier``
``tmux`` control B then % to split terminal into 2
``kubectl edit -n deployments deployments app-tier`` (left terminal)
``kubectl rollout -n deployments status deployment app-tier`` (right terminal)
``kubectl rollout -n deployments pause deployment app-tier`` (left terminal)
``kubectl get deployments -n deployments app-tier`` (left terminal)
``kubectl rollout -n deployments resume deployment app-tier`` (left terminal)
``kubectl rollout -n deployments undo deployment app-tier`` rollback. 
``kubectl rollout history -n deployments deployment app-tier`` use history to see versions and we can rollback to specific version

``kubectl scale -n deployments deployment app-tier --replicas=1``
 
### 3.2.8 Probes

``cat 7.1-namespace.yaml``
  ```
  apiVersion: v1│
  kind: Namespace│
  metadata:│
    name: probes│
    labels:│
      app: counter
  ```

``cat 7.2-data_tier.yaml``

  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: data-tier
    labels:
      app: microservices
  spec:
    ports:
    - port: 6379
      protocol: TCP # default 
      name: redis # optional when only 1 port
    selector:
      tier: data 
    type: ClusterIP # default
  ---
  apiVersion: apps/v1 # apps API group
  kind: Deployment
  metadata:
    name: data-tier
    labels:
      app: microservices
      tier: data
  spec:
    replicas: 1
    selector:
      matchLabels:
        tier: data
    template:
      metadata:
        labels:
          app: microservices
          tier: data
      spec: # Pod spec
        containers:
        - name: redis
          image: redis:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 6379
              name: redis
          livenessProbe:
            tcpSocket:
              port: redis # named port
            initialDelaySeconds: 15
          readinessProbe:
            exec:
              command:
              - redis-cli
              - ping
            initialDelaySeconds: 5
  ```

``cat 7.3-app_tier.yaml``
  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: data-tier
    labels:
      app: microservices
  spec:
    ports:
    - port: 6379
      protocol: TCP # default 
      name: redis # optional when only 1 port
    selector:
      tier: data 
    type: ClusterIP # default
  ---
  apiVersion: apps/v1 # apps API group
  kind: Deployment
  metadata:
    name: data-tier
    labels:
      app: microservices
      tier: data
  spec:
    replicas: 1
    selector:
      matchLabels:
        tier: data
    template:
      metadata:
        labels:
          app: microservices
          tier: data
      spec: # Pod spec
        containers:
        - name: redis
          image: redis:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 6379
              name: redis
          livenessProbe:
            tcpSocket:
              port: redis # named port
            initialDelaySeconds: 15
          readinessProbe:
            exec:
              command:
              - redis-cli
              - ping
            initialDelaySeconds: 5
  ```

``kubectl create -f 7.1-namespace.yaml``
``kubectl create -f 7.2-data_tier.yaml -n probes``
``kubectl get deployments -n probes -w``
``kubectl create -f 7.3-app_tier.yaml -n probes``
``kubectl get -n probes deployments app-tier -w``
``kubectl get -n probes pods``
``kubectl logs -n probes app-tier-... | cut -d' ' -f5,8-11`` (You can use tab completion to display the possible values to replace ... with)

### 3.2.9 Init Containers

``cat 8.1-app_tier.yaml``
```
apiVersion: v1
kind: Service
metadata:
  name: app-tier
  labels:
    app: microservices
spec:
  ports:
  - port: 8080
  selector:
    tier: app
---
 tier: app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-tier
  labels:
    app: microservices
    tier: app
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: app
  template:
    metadata:
      labels:
        app: microservices
        tier: app
    spec:
      containers:
      - name: server
        image: lrakai/microservices:server-v1
        ports:
          - containerPort: 8080
            name: server
        env:
          - name: REDIS_URL
            # Environment variable service discovery
            # Naming pattern:
            #   IP address: <all_caps_service_name>_SERVICE_HOST
            #   Port: <all_caps_service_name>_SERVICE_PORT
            #   Named Port: <all_caps_service_name>_SERVICE_PORT_<all_caps_port_name>
            value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
            # In multi-container example value was
            # value: redis://localhost:6379 
          - name: DEBUG
            value: express:*
        livenessProbe:
          httpGet:
            path: /probe/liveness
            port: server
          initialDelaySeconds: 5
        readinessProbe:
          httpGet:
            path: /probe/readiness
            port: server
          initialDelaySeconds: 3
      initContainers:
        - name: await-redis
          image: lrakai/microservices:server-v1
          env:
          - name: REDIS_URL
            value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
          command:
            - npm
            - run-script
            - await-redis
```

``kubectl apply -f 8.1-app_tier.yaml -n probes``
``kubectl describe pod -n probes app-tier...`` (You can use tab completion to display the possible values to replace ... with)
``kubectl logs -n probes app-tier-... await-redis`` (You can use tab completion to display the possible values to replace ... with)
 

### 3.2.10 Volumes

``cat 9.1-namespace.yaml``
```
apiVersion: v1
kind: Namespace
metadata:
  name: volumes
  labels:
    app: counter
```

``kubectl -n deployments logs support-tier-... poller --tail 1`` (You can use tab completion to display the possible values to replace ... with)
``kubectl exec -n deployments data-tier-... -it /bin/bash`` (You can use tab completion to display the possible values to replace ... with)
``kill 1``
``kubectl -n deployments get pods``
``kubectl -n deployments logs support-tier-... poller --tail 1``(You can use tab completion to display the possible values to replace ... with)
Note: It takes around a couple of minutes for the effects of the restart to settle. The poller will stop updating and report the last value before restarting until it can reach the new data tier value. Try again after a minute if you don't see a relatively small value)
``kubectl create -f 9.1-namespace.yaml``
``aws ec2 describe-volumes --region=us-west-2 --filters="Name=tag:Type,Values=PV" --query="Volumes[0].VolumeId" --output=text``
``vol_id=$(aws ec2 describe-volumes --region=us-west-2 --filters="Name=tag:Type,Values=PV" --query="Volumes[0].VolumeId" --output=text)``
``sed -i "s/INSERT_VOLUME_ID/$vol_id/" 9.2-pv_data_tier.yaml``





``cat 9.2-pv_data_tier.yaml``
```
apiVersion: v1
kind: Service
metadata:
  name: data-tier
  labels:
    app: microservices
spec:
  ports:
  - port: 6379
    protocol: TCP # default 
    name: redis # optional when only 1 port
  selector:
    tier: data 
  type: ClusterIP # default
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-tier-volume
spec:
  capacity:
    storage: 1Gi # 1 gibibyte
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore: 
    volumeID: INSERT_VOLUME_ID # replace with actual ID
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-tier-volume-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 128Mi # 128 mebibytes 
---
apiVersion: apps/v1 # apps API group
kind: Deployment
metadata:
  name: data-tier
  labels:
    app: microservices
    tier: data
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: data
  template:
    metadata:
      labels:
        app: microservices
        tier: data
    spec: # Pod spec
      containers:
      - name: redis
        image: redis:latest
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 6379
            name: redis
        livenessProbe:
          tcpSocket:
            port: redis # named port
          initialDelaySeconds: 15
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
        volumeMounts:
          - mountPath: /data
            name: data-tier-volume
      volumes:
      - name: data-tier-volume
        persistentVolumeClaim:
          claimName: data-tier-volume-claim
```

``cat 9.3-app_tier.yaml``
```
apiVersion: v1
kind: Service
metadata:
  name: app-tier
  labels:
    app: microservices
spec:
  ports:
  - port: 8080
  selector:
    tier: app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-tier
  labels:
    app: microservices
    tier: app
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: app
  template:
    metadata:
      labels:
        app: microservices
        tier: app
    spec:
      containers:
      - name: server
        image: lrakai/microservices:server-v1
        ports:
          - containerPort: 8080
            name: server
        env:
          - name: REDIS_URL
            # Environment variable service discovery
            # Naming pattern:
            #   IP address: <all_caps_service_name>_SERVICE_HOST
            #   Port: <all_caps_service_name>_SERVICE_PORT
            #   Named Port: <all_caps_service_name>_SERVICE_PORT_<all_caps_port_name>
            value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
            # In multi-container example value was
            # value: redis://localhost:6379 
          - name: DEBUG
            value: express:*
        livenessProbe:
          httpGet:
            path: /probe/liveness
            port: server
          initialDelaySeconds: 5
        readinessProbe:
          httpGet:
            path: /probe/readiness
            port: server
          initialDelaySeconds: 3
      initContainers:
        - name: await-redis
          image: lrakai/microservices:server-v1
          env:
          - name: REDIS_URL
            value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
          command:
            - npm
            - run-script
            - await-redis
```

``cat 9.4-support_tier.yaml``
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: support-tier
  labels:
    app: microservices
    tier: support
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: support
  template:
    metadata:
      labels:
        app: microservices
        tier: support
    spec:
        containers:

        - name: counter
          image: lrakai/microservices:counter-v1
          env:
            - name: API_URL
              # DNS for service discovery
              # Naming pattern:
              #   IP address: <service_name>.<service_namespace>
              #   Port: needs to be extracted from SRV DNS record
              value: http://app-tier:8080

        - name: poller
          image: lrakai/microservices:poller-v1
          env:
            - name: API_URL
              # omit namespace to only search in the same namespace
              value: http://app-tier:$(APP_TIER_SERVICE_PORT)
```

``kubectl create -n volumes -f 9.2-pv_data_tier.yaml -f 9.3-app_tier.yaml -f 9.4-support_tier.yaml``
``kubectl describe -n volumes pvc``
``kubectl describe -n volumes pod data-tier-...`` (You can use tab completion to display the possible values to replace ... with)
``kubectl logs -n volumes support-tier-... poller --tail 1`` (You can use tab completion to display the possible values to replace ... with)
Note: It takes a few minutes for all of the readiness checks to pass and for the counter to start incrementing. If you don't see a counter value output then try again after a minute or two.
``kubectl delete -n volumes deployments data-tier``
``kubectl get -n volumes pods``

``kubectl create -f 9.2-pv_data_tier.yaml``
``kubectl logs -n volumes support-tier-... poller --tail 1`` (You can use tab completion to display the possible values to replace ... with)
 

### 3.2.11 ConfigMaps and Secrets

``cat 10.1-namespace.yaml``
```
apiVersion: v1
kind: Namespace
metadata:
  name: config
  labels:
    app: counterubuntu@ip-10-0-128-5:~/src$ 
ubuntu@ip-10-0-128-5:~/src$ cat 10.1-namespace.yaml  | more
apiVersion: v1
kind: Namespace
metadata:
  name: config
  labels:
    app: counter
```

``cat 10.2-data_tier_config.yaml ``
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  config: | # YAML for multi-line string
    # Redis config file
    tcp-keepalive 240
    maxmemory 1mb
```

``cat 10.3-data_tier.yaml``
```
apiVersion: v1
kind: Service
metadata:
  name: data-tier
  labels:
    app: microservices
spec:
  ports:
  - port: 6379
    protocol: TCP # default 
    name: redis # optional when only 1 port
  selector:
    tier: data 
  type: ClusterIP # default
---
apiVersion: apps/v1 # apps API group
kind: Deployment
metadata:
  name: data-tier
  labels:
    app: microservices
    tier: data
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: data
  template:
    metadata:
      labels:
        app: microservices
        tier: data
    spec: # Pod spec
      containers:
      - name: redis
        image: redis:latest
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 6379
            name: redis
        livenessProbe:
          tcpSocket:
            port: redis # named port
          initialDelaySeconds: 15
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
        command:
          - redis-server
          - /etc/redis/redis.conf
        volumeMounts:
          - mountPath: /etc/redis
            name: config
      volumes:
        - name: config
          configMap:
            name: redis-config
            items:
            - key: config
              path: redis.conf
```

``kubectl create -f 10.1-namespace.yaml``
``kubectl create -n config -f 10.2-data_tier_config.yaml -f 10.3-data_tier.yaml``
``kubectl exec -n config data-tier-... -it /bin/bash`` (You can use tab completion to display the possible values to replace ... with)
``cat /etc/redis/redis.conf``
``redis-cli CONFIG GET tcp-keepalive``
``exit``
``kubectl edit -n config configmaps redis-config``
``kubectl exec -n config data-tier-... redis-cli CONFIG GET tcp-keepalive`` (You can use tab completion to display the possible values to replace ... with)
``kubectl rollout -n config restart deployment data-tier``
``kubectl exec -n config data-tier-... redis-cli CONFIG GET tcp-keepalive`` (You can use tab completion to display the possible values to replace ... with)

``cat 10.4-app_tier_secret.yaml``
```
apiVersion: v1
kind: Secret
metadata:
  name: app-tier-secret
stringData: # unencoded data
  api-key: LRcAmM1904ywzK3esX
  decoded: hello
data: #for base-64 encoded data
  encoded: aGVsbG8= # hello in base-64

# api-key secret (only) is equivalent to
# kubectl create secret generic app-tier-secret --from-literal=api-key=LRcAmM1904ywzK3esX
```

``kubectl create -f 10.4-app_tier_secret.yaml -n config``
``kubectl describe -n config secret app-tier-secret``
``kubectl edit -n config secrets data-tier-secret``

``cat 10.5-app_tier.yaml``
```
apiVersion: v1
kind: Service
metadata:
  name: app-tier
  labels:
    app: microservices
spec:
  ports:
  - port: 8080
  selector:
    tier: app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-tier
  labels:
    app: microservices
    tier: app
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: app
  template:
    metadata:
      labels:
        app: microservices
        tier: app
    spec:
      containers:
      - name: server
        image: lrakai/microservices:server-v1
        ports:
          - containerPort: 8080
            name: server
        env:
          - name: REDIS_URL
            # Environment variable service discovery
            # Naming pattern:
            #   IP address: <all_caps_service_name>_SERVICE_HOST
            #   Port: <all_caps_service_name>_SERVICE_PORT
            #   Named Port: <all_caps_service_name>_SERVICE_PORT_<all_caps_port_name>
            value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
            # In multi-container example value was
            # value: redis://localhost:6379 
          - name: DEBUG
            value: express:*
          - name: API_KEY
            valueFrom:
              secretKeyRef:
                name: app-tier-secret
                key: api-key
        livenessProbe:
          httpGet:
            path: /probe/liveness
            port: server
          initialDelaySeconds: 5
        readinessProbe:
          httpGet:
            path: /probe/readiness
            port: server
          initialDelaySeconds: 3
      initContainers:
        - name: await-redis
          image: lrakai/microservices:server-v1
          env:
          - name: REDIS_URL
            value: redis://$(DATA_TIER_SERVICE_HOST):$(DATA_TIER_SERVICE_PORT_REDIS)
          command:
            - npm
            - run-script
            - await-redis
```

``kubectl create -f 10.5-app_tier.yaml -n config``
``kubectl exec -n config app-tier-... env`` (You can use tab completion to display the possible values to replace ... with)
