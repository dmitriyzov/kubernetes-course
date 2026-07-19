# Kubernetes

## Resources
* [Kubernetes Full Course](https://youtu.be/X48VuDVv0do)

## Table of contents

* [Main components](#main-components)
* [Layers of Abstraction](#layers-of-abstraction)
* [Configuration files](#configuration-files)
* [Namespaces](#namespaces)
* [Data persistence](#data-persistence)
* [ConfigMap and Secret](#configmap-and-secret)
* [Ingress and Egress](#ingress-and-egress)
* [minikube](#minikube)
* [kubectl](#kubectl)
* [Exercises](#exercises)
* [Helm](#helm)

## Main components

### **Node** - physical server
* each Node has multiple Pods on it
* There are 2 types of Nodes - Worker Nodes and Control Plane Nodes
* **Worker nodes**
    * 3 processes run on every worker node
        * container runtime - Docker
        * Kubelet - K8s process that interacts with both the container and the node
	    * Starts and runs the pods with a container inside.
	    * Watches the K8S API for Pods assigned to its node
	    * Tells the container runtime to start/stop containers
	    * Runs health checks
	    * Reports Pod and node status back to control plane
        * Kube Proxy - handles K8s Service networking on each worker node
	    * forwards the requests from Services to Pods
	    * Ensures performant communication with low overhead
* **Control Plane nodes** (formerly 'Master Nodes') - control the cluster state and the worker nodes
    * 4 processes run on every Control Plane node
        * API Server - cluster gateway. receives the initial requests to update the cluster. Gatekeeper for authentication
        * Scheduler - decides on which Node to put a new Pod (doesn’t actually run it, Kubelet does that on the worker node itself)
        * Controller manager - detect state changes (like when nodes die) and reschedules/recovers via scheduler
        * etcd - cluster brain, key/value store for changes. 
    * There are usually multiple Control Plane nodes. The API service is load balanced and etcd stores a distributed storage across all of the Control Plane nodes.  

### **Pod** - smallest unit of K8, abstraction over a container.
* Pods are ephemeral
* Creates a layer on top of a container so that the container is abstracted away.
* Runs 1 application container per pod.
* Each pod gets its own IP address.

### **Deployment** - blueprint for a Pod, abstraction of a Pod
* used for stateLESS applications
* you create Deployments rather than working with Pods directly
* you can scale the number of replicas you need using a replicaset
* deployments get a random hash appended to the name
* databases can’t be replicated using Deployments

### **StatefulSet** - component for stateFULL Applications
* should be used over Deployments for databases (MySql, MongoDB, ElasticSearch)
* takes care of replicating and scaling stateful Pods
* StatefulSets have fixed ordered names
* In addition, StatefulSet pods have fixed individual DNS names. When Pod restarts, IP address changes,
but name nad endpoint stay the same.
* deploying DBs using Stateful Sets is difficult, so DBs are often hosted outside of K8 clusters
    * replica Pods can't be created/deleted at the same time
    * replica Pods can't be randomly addressed
    * unlike Deployment Pod s, replica Pods are not identical - they have Pod Identity

### **Volumes** - attach a physical storage to a pod so that data lifecycle is not tied to a pod itself
* K8 doesn’t manage data persistence (unless you use Persistent Volumes)
* data can be on local machine
* can be remote, outside the K8 cluster
* when a Pod is restarted, data persists

### **Service** - permanent IP address that’s attached to a Pod.
* Serves as a permanent IP for a Pod and as a Load Balancer between pods.
* Pods communicate with each other using a service.
* Lifecycles of pods and services are not connected (if pod dies, service stays)
* External service - service that opens communication from external service.
* Internal service - for db, etc
* `protocol://ip-address:port`
* Service types
    * ClusterIP - default type 
    * Headless

### **Ingress** - pretty url, forwards the request to a service

### **ConfigMap** - external configuration of your application
* contains URLs of dbs, etc
* connected to a Pod so the Pod gets the data from it
* don’t use it for credentials in plain text

### **Secret** - used to store secret data (passwords, certificates)
* stored in base64, not plaintext
* should be preferred over ConfigMap for sensitive data
* the built-in security mechanism is not enabled by default
* not fully secure unless you use add'l measures like encryption at rest or tools like HashiCorp Vault

## Layers of Abstraction

* Deployment manages a ReplicaSet
* ReplicaSet manages a Pod
* Pod is an abstraction of a container
* Everything below a Deployment is managed by Kubernetes

## Configuration files

`kind` specifies the component kind ('Deployment', 'Service', etc). This determines what attributes are available in "spec".

Each configuration file has 3 main parts:

1. `metadata` - name of the component, etc.
2. `spec` - component specification. Replias, selector, template, ports, etc. Set of available attributes depends on the component "kind".
3. `status` - this is automatically generated and continuously updated by Kubernetes so that it can compare the desired state (config file) to the actual state (etcd). This is the basis of the self-healing feature that K8s provides.

### Spec

* Templates have their own "metadata" and "spec" sections that apply to Pods. They are like Blueprints for Pods.
* Labels & Selectors - used for establishing a connection. Label labels the entity itself, and selector is what lets it connect to other entities by their own labels. Pod gets a label through the template blueprint. This label is matched by the selector.

## Namespaces

Namespaces are used to group resources, usually in larger projects. Namespaces  are like clusters within a cluster.
There are 4 default namespaces:
* kube-system - not for your use, for internal components (system processes)
* kube-public - publicly-accessible data. Configmap contains cluster info without auth
* kube-node-lease - holds information about heartbeats of nodes
* default - this is the one you create resources in unless you create/specify another namespace. You can change the active namespace using `kubens`
* kubernetes-dashboard - non-default, specific to minikube.

There are a few use cases for namespaces:
1. Grouping resources logically
2. Grouping resources by teams (prevent affecting other teams)
3. Allowing resource sharing between environments (ie shared ELK stack) or versions of environments (Blue/Green)
4. Limiting resource access by namespace (each team only has access to their resources) and limit resources that a namespace consumes by using resource quotas

Characteristics of namespaces
1. You can't access most resources from another namespace (you can't use Project A's ConfigMap that references the database in Project B)
2. You CAN share a Service between namespaces (both Project A and Project B can use the mysql-service in the "database" namespace)
3. Some components cannot be created in a Namespace but are rather global (volume/persistent volume, node)

```bash
# Listing namespaced resources
kubectl api-resources --namespaced=true

# Listing non-namespaced resources
kubectl api-resources --namespaced=false
```

## Data persistence

### Persistent Volumes
* Persistent volumes are NOT namespaced, available to all clusters
* There are 2 categories - Local and Remote, each with it's own use case
  * Local - violates 2/3 requirements for data persistence - tied to one specific node, won't survive cluster crashes
  * Remote - 
* K8s Admin - sets up and maintains the cluster and configures storage
* K8s User - deploys applications in the custer and claims storage using a pvc (PersistentVolumeClaim)
* Claims - Pod requests the volume using the PersistentVolumeClaim. Claims must exist in the same namespace as the Pod
* Claim tries to find a Persistent Volume in the cluster that satisfies the claim
* Volume has the actual storage backend
* Volume can be mounted into the Pod or into the Container inside the Pod  
* PV Claims must exist in the same namespace as pods using them, unlike Persistent Volumes, which are not namespaced

### Storage Class
A Storage Class provisions Persistent Volumes dynamically, whenever Persistent Volume Claim claims it.
This way, provisioning Volumes in a cluster can be automated.  
The `provisioner` attribute tells K8s which provisioner to use (EBS, EFS, etc) for a specific Platform/Cloud provider to create a PV component.
There are internal (`kubernetes.io`) and external (Cloud) provisioners

### Storage abstraction layers
1. Pod claims storage via a Persistent Volume Claim  
2. Persistent Volume Claim requests storage from the Storage Class  
3. Storage Class creates a Persistent Volume that meets the needs of the Claim

## ConfigMap and Secret

* These are also volume types
* Not created via PV and PVC and is managed by Kubernetes itself
* You can mount either into your Pod or Container 

## Ingress and Egress

### Ingress

Instead of accessing the application using an external service (IP:PORT), you can use an Ingress, which will redirect the request to an internal service.
This way, you don't have to expose the IP and the port. Additionally, you don't have to specify the `nodePort` on the internal service, and you don't have to set `type:LoadBalancer` - the default (implied) `type:ClusterIP` is used instead.

However, creating the Ingress in yaml alone won't work - you also need an implementation for Ingress, aka Ingress Controller.

An Ingress Controller is installed on another Pod or set of Pods, that run on a node in your K8s cluster, and handles evaluation and processing of Ingress rules that are defined in the cluster. The Ingress Controller Pod becomes the entrypoint into the Cluster. There are different implementations of the Ingress Controller, including the K8s Nginx Ingress Controller.

- entrypoint to the cluster  
- evaluates all the rules  
- manages redirection

To install Ingress in a Minikube cluster, you can run `minikube addons enable ingress`.

This automatically starts the K8s Nginx implementation of Ingress Controller.

## minikube
* Creates a Virtual Box on your local machine
* A single Node runs both Control Plane and Worker processes
* minikube CLI is used mostly for starting up or deleting the cluster
* everything else will be done with `kubectl`

```bash
minikube config set driver hyperkit # or docker if using an Apple Silicon chip
minikube start
minikube status
minikube addons enable ingress
```

## kubectl
* CLI tool for Kubernetes clusters (minikube or Cloud)
* way to interact with a cluster

### CRUD commands

```bash
# create deployment
kubectl create deployment [name]

# edit deployment
kubectl edit deployment [name]

# delete deployment
kubectl delete deployment [name]
```

### Status of different K8s components

```bash
kubectl get nodes | pod | services | replicaset | deployment | all
```

### Debugging pods

```bash
# view logs
kubectl logs [pod name]

# get additional details
kubectl describe pod [pod name]

# get interactive terminal
kubectl exec -it [pod name] -- bin/bash
```

### Working with configuration files

```bash
# Create/update using a configuration file
kubectl apply -f nginx-deployment.yaml

# Delete using a configuration file
kubectl delete -f nginx-deployment.yaml
```

## Exercises

### Kubernetes Dashboard
```bash
minikube addons enable dashboard
minikube addons enable metrics-server
kubectl apply -f   ingress.yaml

# add this line to /etc/hosts
<minikube ip> dashboard.com # if using the hyperkit driver
127.0.0.1 dashboard.com # if using the docker driver
minikube tunnel # only needed if using the docker driver
```

Kubernetes dashboard will be available at http://dashboard.com

See https://github.com/kubernetes/minikube/issues/13510

### MongoDB & Mongo-Express
```bash
kubectl apply -f mongodb-secret.yaml
kubectl apply -f mongodb.yaml
kubectl apply -f mongo-express.yaml
kubectl apply -f mongo-express-ingress.yaml

# at this point, `kubectl get service` will list the External IP of the external service as "Pending".
# to assign it, run:
minikube service mongo-express-external-service

# username: admin
# password: pass
```

Alternatively, to access the mongo express dashboard using a hostname and a port:

```bash
# add this line to /etc/hosts
<minikube ip> mongo-express.com # if using the hyperkit driver
127.0.0.1 mongo-express.com # if using the docker driver
minikube tunnel # only needed if using the docker driver

# Note that you have to get the port the external service is running on:
kubectl get service

# you will see something like:
NAME                             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mongo-express-external-service   LoadBalancer   10.106.57.109    127.0.0.1     8081:30081/TCP   3d
```

Mongo-express will be available at:
* http://mongo-express.com:8081 (using an external service)
* http://mongo-express.com (using ingress)

See https://github.com/kubernetes/minikube/issues/13510


## Helm

### What is Helm?
#### 1. Package manager for Kubernetes 
packages YAML files as Helm Charts and distribute them in public and private registries

#### 2. Templating engine
You can define templates for microservices and store dynamic values in a separate file.

*Helm Chart directory structure:*
```bash
.
├── mychart/        -- Top level folder - name of the chart
│   ├── Chart.yaml  -- meta info about the chart
│   ├── values.yaml -- default values for the template files
│   ├── charts/     -- chart dependencies
│   └── templates/  -- template files, get filled with values from values.yaml
```

*Value injection using Helm*
```yaml
# values.yaml - default values
imageName: myapp
port: 8080
version: 1.0.0
```

```yaml
# my-values.yaml - override values
version: 2.0.0
```

```bash
helm install --values=my-values.yaml <chartname>
```

```yaml
# result
imageName: myapp
port: 8080
version: 2.0.0
```

#### 3. Release management
