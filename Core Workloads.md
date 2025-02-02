
## Core Abstractions

- Pods are the smallest abstraction for a grouping of containers making one application.
	- This group of containers share the same resources like file systems, kernel namespaces and IP address
- Pod is the atomic unit of scheduling in a Kubernetes cluster

## Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    args: [/bin/sh, -c,
            'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
    ports:
    - containerPort: 80
```

<aside> 💡

Once a pod is deployed (by itself) the spec of the containers cannot be modified. It must be re-created.

</aside>


### Multi-Containers
- A pod can hold multiple containers for a Pod kind
- Containers within the same pod share the Pod's IP address and can communicate with each other, not requiring a NAT table
- To add multiple containers in a pod, add all the required containers within the Pod's spec

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    ports:
    - ContainerPort: 8080
  - name: log-agent
    image: log-agent
```

[multi-container pattern - Kubernetes blog](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/)
> Note: these are not part of the CKA exam, but it's a good to know different sidecar patterns exists for different architectural use cases
### Sidecar Contianer Pattern
- Sidecar containers will enhance the "main" container. (logging sidecar container)
- We modularized and decoupled the main application and the logging scraper which share the same Filesystem for example. Meaning two different teams can work on the main app - and the logging application separately

![](https://kubernetes.io/images/blog/2015-06-00-The-Distributed-System-Toolkit-Patterns/sidecar-containers.png)

### Ambassador containers
- These kinds of containers proxy a local connection to the world (e.g. istio sidecar)
- The ambassador can split read and writes and send them to appropriate servers. Since containers in the same pods share the network namespace, the application opens a connection to localhost and finds the proxy without any additional service discovery 
![](https://kubernetes.io/images/blog/2015-06-00-The-Distributed-System-Toolkit-Patterns/ambassador-containers.png)


### Adapter containers
- Adapter containers standardizes and normalizes output. We transform heterogeneous monitoring data from different systems into a single unified representation by creating pods grouping the application containers with adapters performing the transformation

![](https://kubernetes.io/images/blog/2015-06-00-The-Distributed-System-Toolkit-Patterns/adapter-containers.png)

### Commands

```yaml
# kubectl commands for pods
kubectl get pods <pod name> -n <namespace> -o yaml 
kubectl describe pod <pod name> -n <namespace>
kubectl logs <pod name> -n <namespace>
kubectl delete pod <pod name> -n <namespace>

# create a pod fast using --dry-run flag
kubectl run <pod name> -n <namespace> --image=<the image for the container, ...> --dry-run=client -o yaml > <pod file>.yaml

kubectl apply -f <pod file name> 
```

## ReplicaSet

Provides a way to maintain X stable set of replica Pods running at any given time.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # modify replicas according to your case
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
  # metadata.labels MUST MATCH spec.selector, or the API will reject the object req. 
    spec:
      containers:
      - name: php-redis
        image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5

```

## DaemonSet

[DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

- DaemonSets allows us to guarantee that an instance of our workload will be deployed onto each of our nodes.

Useful for use cases such as:

- monitoring and log ingestion on a per-node basis
    
- networking solutions that needs to be deployed on each node in the cluster
    
- See sample YAML file
    
    ```yaml
    ## It is similar to a ReplicaSet except the kind is set
    # as DaemonSet
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: monitoring-daemon
      labels:
        app: nginx
    spec:
    # notice how we didn't add a replica annotation?
      selector:
        matchLabels:
          app: monitoring-agent
      template:
        metadata:
         labels:
           app: monitoring-agent
        spec:
          containers:
          - name: monitoring-agent
            image: monitoring-agent
    ```
    

## Static Pods

What if we have no kube-controller? Can a kubelet operate independently from the Control Plane components?

We can additionally make use of static pods to deploy api components on our control plane nodes.

The Kubelet can actually operate and schedule workloads on its own by reading from the dedicated directory designated to store information about pods

<aside> 💡

/etc/kubernetes/manifests

</aside>

Kubelet periodically checks this directory for manifests, and also ensures the pods created within it stay alive. (if app crashes, kubelet re-creates it)

- However, if the manifest is removed from the directoy, it is deleted.
- These are called static pods (and only pods can be created)

This “path / directory” is designated by the Kubelet configuration that we configure and can be anywhere on the host:

We can configure this in 2 ways:

1. When setting up the kubelet.service, we can configure in ExecStart the pod-manifest-path
    
    ![Screenshot 2024-09-21 at 1.48.52 AM.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2028682c-8c9a-469b-aca6-52abed699b3a/fc584fb0-b8b9-48b6-b56d-e8cc1c2df951/Screenshot_2024-09-21_at_1.48.52_AM.png)
    

```yaml
# This is usually under this directory

/etc/systemd/system/kubelet.service
```

1. specify a config.yaml file and within that config file, add a staticPodPath (Most common for kubeadm deployed clusters)
    
    ![Screenshot 2024-09-21 at 1.50.02 AM.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2028682c-8c9a-469b-aca6-52abed699b3a/aeb8c5ce-a8ba-49aa-a7f8-f6a85350a849/Screenshot_2024-09-21_at_1.50.02_AM.png)
    

```yaml
# bash command to check the kubelet
$ systemctl status kubelet

# /var/lib/kubelet/config.yaml <--- config for kubelet
# This for kubeadm deployed kubelet agent

apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
containerRuntimeEndpoint: ""
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMaximumGCAge: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
logging:
  flushFrequency: 0
  options:
    json:
      infoBufferSize: "0"
    text:
      infoBufferSize: "0"
  verbosity: 0
memorySwap: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
resolvConf: /run/systemd/resolve/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests <---
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
```

<aside> 💡

Static Pods can only be deleted from the manifest directory of the node that is hosting this pod. It cannot be deleted by kubectl

</aside>

![Screenshot 2024-09-21 at 1.55.33 AM.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2028682c-8c9a-469b-aca6-52abed699b3a/71705695-4a31-44e7-a8af-e5dc768bf9af/Screenshot_2024-09-21_at_1.55.33_AM.png)

## Optional Specs
### Running Commands in the Pod’s Containers

Similar to adding EntryPoints and Commands for Dockerfiles when creating Containers, we can add the command and args annotations in the container spec to perform additional action.

- Note the commands and args you provide in the Pod’s manifest overrides the default commands and args for a Container image
- ENTRYPOINT ←→ command
- CMD ←→ args

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
 containers:
 - name: ubuntu-sleeper
   image: ubuntu-sleeper
   command: ["sleep2.0"]
   args: ["10"]
   
--
# We can also use environment variables to define arguments
...
env:
- name: MESSAGE
  value: "hello world"
command: ["/bin/echo"]
args: ["$(MESSAGE)"]

--
# In some cases, certain commands must run in a shell
command: ["/bin/sh"]
args: ["-c", "while true; do echo hello; sleep 10;done"] 
```

[Define a Command and Arguments for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)

### Environment Variables

As from previous, we can set an environment variable with using the env property in the Pod’s manifest.

We can set environment variables in the following methods:

- Plain key-value pair
- ConfigMap
- Secrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
 containers:
 - name: simple-webapp-color
   image: simple-webapp-color
   ports:
   - containerPort: 8080
   env:
   - name: APP_COLOR
     value: pink
```

### Init Containers
- We can use init containers to run containers first and have it complete before the main application container starts
- We can configure multiple initContainers. Note that these containers are run **one at a time in sequential order**

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```


## HealthChecks via Probes
- Kubernetes offers Liveness, Readiness and Startup Probes to check on the health status of a Pod

Liveness probe
- determines when to restart a containers (e.g. could catch a deadlock when an application is running but unable to progress)
	- If the probe fails repeatedly, then the Kubelet agent will attempt to restart the container
- Liveness probes **do not wait for readiness probes to succeed**. To wait before executing a liveness probe, define the `initialDelaySeconds` spec or startup probe

Readiness probe
- Determines if the container is ready to start accepting traffic
	- Useful cases include waiting for an application to perform time-consuming tasks such as network connection establishment, file loading, cache warming
- If this probe returns a failed state, Kubernetes removes the pod from all matching service endpoints
- Runs on the container during its whole lifecycle

Startup probe
- Checks to verify if the application within the container is started. Does NOT run periodically
- Can be used to adopt liveness checks on slow starting containers, thus avoiding them being killed by the Kubelet agent before they are up
Note: liveness and readiness checks are disabled until the startup probe succeeds


## ConfigMaps

Resources created using the ConfigMap API stores data as key-value pairs, which can be consumed in pods or provide configuration for components like the controller

While ConfigMaps are similar to how Secrets are created, it provides us a way to manipulate data that are not considered as sensitive information

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
 name: app-config
data:
 APP_COLOR: blue
 APP_MODE: prod
```

Adding a ConfigMap into a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
 containers:
 - name: simple-webapp-color
   image: simple-webapp-color
   ports:
   - containerPort: 8080
   envFrom:  <--- define the configmap we will use
   - configMapRef:
       name: app-config
```

We can also inject a ConfigMap into a Pod using alternative methods:

1. If we want to only get one variable from the configmap

```yaml
env:
	- name: APP-COLOR
		valueFrom:
			configMapKeyRef:
				name: app-config
				key: APP_COLOR
```

1. Inject the configmap as a file in a volume for the pod to consume

```yaml
volume:
- name: app-config-volume
	configMap:
		name: app-config
```

### Commands

```bash
# create configmap using kubectl create
$ kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod
$ kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties
# create from a directory containing CMs
$ kubectl create configmap game-config --from-file=my/configmaps/

# useful commands
k get cm <name> -n <namespace>
k describe configmaps <name> -n <ns>

```

## Secrets

- Secrets are similar to ConfigMaps, except they are used to store sensitive information in an encrypted / hashed format
- Secrets encode data in base64 format, but are not natively encrypted. Secrets should be managed / handled using another way that is more secure (such as aws secretsmanager or vault)

### Commands

```yaml
$ kubectl create secret generic app-secret --from-literal=DB_Host=mysql --from-literal=DB_User=root --from-literal=DB_Password=paswrd
$ kubectl create secret generic app-secret --from-file=app_secret.properties
```

We can encode and decode a secret
```bash
kubectl get secrets/db-user-pass --template={{.<key>.value}} | base64 -d
```

# Custom Resource Definitions and Operators

## CRDs

[CRDs Docs link](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

Custom resources are an extension of the Kubernetes API. Think of it as a customization feature of a Kubernetes installation. 
- Cluster admins must manage the CRs independently of the cluster. But, it is possible to access it via kubectl once the definition of the resource is created. (resources are but an endpoint of the K8s api storing a collection of API objects which forms a `kind`)

![[Screenshot 2025-02-02 at 3.26.35 PM.png]]

Custom Resource Definition
```YAML
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
	name: my.api.com
spec:
	scope: Namespaced <--- tells us if scoped by namespace
	group: dragonquestmob.com/v1
	names:
		kind: RPGMob
		singular: RPGmob
		plural: RPGmobs
		shortnames:
		- mob
	versions:
		- name: v1
		  served: true  <- choose which version served to API
		  storage: true
	schema:
		openAPIV3Schema: <-- tells what all possible fields are supported
			type: object
			properties:
				spec:
				  type: object
				  properties:
				    hp:
				      type: integer
				    name:
					  type: string
				[...] 
```


Custom Resource
```YAML
apiVersion: dragonquestmob.com/v1
kind: RPGMob
metadata:
	name: slime-mob
spec:
	hp: 2
	name: slime
	region: Fortuna-area
```


## Operator
- Controller runs in a loop and watches the cluster and listens to events related to the designated custom resource(s)
- Kubernetes has a template for controllers labeled as [sample controller](https://github.com/kubernetes/sample-controller)

An example of a popular operator is the ETCd operator which has CRDS for etcd clusters, backups and restores, allowing you to deploy, backup and restore ETCd for Kubernetes.

[operator-hub](https://operatorhub.io/)
