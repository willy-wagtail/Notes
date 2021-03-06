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

