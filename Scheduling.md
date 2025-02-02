# Scheduling

## Manual Scheduling

We can create so called “static pods” and explicitly note the node name of which will host the pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: nginx
 labels:
  name: nginx
spec:
 containers:
 - name: nginx
   image: nginx
   ports:
   - containerPort: 8080
   nodeName: ournode
```

Additionally, we can also create a “Binding” Kind object

```yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node02  <--- for which node to specify for
```

## Labels and Selectors

<aside> 💡

labels are case-sensitive!

</aside>

```yaml
# Query pods with a specific selector
$ kubectl get pods --selector app=App1
# gets us the count of pods using env=dev label 
# use no-headers flag to prevent the NAME - READY - etc. line to not be printed
$ kubectl get pods --selector env=dev --no-headers | wc -l

# Example of a ReplicaSet with selectors defined for the pod templates
# This will group the respective pods related to the RS together
 apiVersion: apps/v1
 kind: ReplicaSet
 metadata:
   name: simple-webapp
   labels:
     app: App1            # <---- They are given this label
     function: Front-end
 spec:
  replicas: 3
  selector:
    matchLabels:   # <----- We group the replica pods containing the label "app: App1"
     app: App1
  template:
    metadata:
      labels:
        app: App1     # <---- We match all pods which will add this label in template
        function: Front-end
    spec:
      containers:
      - name: simple-webapp
        image: simple-webapp  

# we can do the same with Services
apiVersion: v1
  kind: Service
  metadata:
   name: my-service
  spec:
   selector:
     app: App1
   ports:
   - protocol: TCP
     port: 80
     targetPort: 9376 
```

Note: we also use annotations to provide some info on the purpose or on the pod specifics

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: Front-end
  annotations:     # <--- Here we have given the buildversion annotation to specify the version of the app
     buildversion: 1.34
[...]
```

## Taints and Tolerations

Taints and Tolerations are used to direct pods onto specific nodes.

To put it simply:

- Taints - annotation on nodes, they push pods away from itself
- Tolerations - annotation on the pods, pull pods to a specific toleration (e.g. the scheduler is allowed to schedule a pod onto a specific node, but this is not necessarily a guarantee)

[https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

Multiple taints / tolerations can be added onto a node / pod.

The Kubernetes mechanism: Ignore matching taint - tolerations and watch for the effect of the mismatch.

- NoSchedule on nm, pod is not schedulable
- NoSchedule on nm and PreferNoSchedule on m, it will try not to schedule
- NoExecute on nm, pod is not schedule and is evicted if already on it.

<aside> 💡

nm = non matching

m = matching

</aside>

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2028682c-8c9a-469b-aca6-52abed699b3a/5491b814-a83c-428d-92e4-73d3126139b9/image.png)

Exists and will match a specific key to whatever the node’s Taint has the right key regardless of the value. the “equal” operator matches to an exact key-value taint

> If the “key” is empty +  “Exists” → everything is tolerated for the pod If “Effect” empty + “key” → all effect is matched

Effects:

**NoExecute**

Case A: Toleration does not match (currently on node)

- Pods with non matching toleration is evicted if on the node

Case B: toleration matches

- TolerationSeconds exists → pod evicted after that specified time by the node lifecycle controller
- TolerationSeconds DNE → pod stays bound forever

**NoSchedule**

- Pods on the node currently are not evicted, however new pods will not be scheduled on the node if toleration mismatches

**PreferNoSchedule**

- A soft requirement (kind of like a “Preferred qualification” in a job application) we might avoid placing XYZ pod on a node with mismatch taint but it is NOT guaranteed.

[+] [https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

TolerationSeconds

- is useful for cases where we want a pod to stay bound to a failing node for example.
- Say a node goes to not ready, then the node controller adds the [node.kubernetes.io/not-ready](http://node.kubernetes.io/not-ready) annotation.
- We can use a toleration annotation on a pod to say “hey stay there for 10 minutes to vibe”

```yaml
tolerations:
- key: " node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 600 

```

<aside> 💡

for the not-ready or unreachable annotation, K8s will auto add the toleration period to 5 min.

</aside>

## Node Selectors

the nodeSelector annotation is the simplest form of node selection contstraint. You can inject this into a Pod spec such that the Scheduler will only schedule that pod onto a node containing the labels we specify

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
# We make this pod schedule on nodes with the disktype=ssd label
  nodeSelector:
    disktype: ssd

# alternatively, we can force a pod to schedule on a specific node

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: foo-node # schedule pod to specific node
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent

```

[Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)

## Node Affinity/Anti-Affinity

Affinity and Anti-affinity are the concepts we use to constrain Pods to specific labels. (They give us more control than what Taints and Tolerations gives us [taint - toleration does not guarantee!] These also give us better control over the rules we place in the node world compared to a simple nodeSelector.

- Consider a case where a node does not have any taints. Then a pod with a specific toleration might end up being scheduled on it...

Affinity: you get to constrain pods on nodes you want it to be scheduled based on node labels

- We need to consider two states of the pod during its lifecycle

Types:

- requiredDuringSchedulingIgnoredDuringExecution
    - Like node selector. Pod cannot be scheduled unless it hits the right requirement
        - Scheduler will mandate it must be scheduled on the one matching the affinity rule
        - Useful for when placement of the pod is crucial
- preferredDuringSchedulingIgnoredDuringExecution:
    - It tries to look for the matching node and if a node with the right condition is not present, pod is scheduled regardless.

DuringScheduling - state where a pod does not exist and created for the first time

DuringExecution - state where pod is running and a change in the environment is made (e.g. relabeling the node with an affinity rule)

If both nodeSelector and affinity labels are specified, **both** must be met.

- Multiple parameters at the nodeSelectorTerms level operate on OR logic
- Multiple parameters at matchExpressions level operate on AND logic

<aside> 💡

requiredDuringSchedulingRequiredDuringExecution has not been released yet

</aside>

When configuring preferred → we can add a weight value (integer between 1 - 100). How the scheduler will evaluate which node to put this pod onto, it will tally up the weight values depending on what node will hit which criteria. The highest value determined will win the bid and get the pod to move in.

- We can also add a weight (think of it as bonus points in a game) for each expression hit

![Screenshot 2024-09-19 at 2.55.22 PM.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2028682c-8c9a-469b-aca6-52abed699b3a/8e68dd86-27c8-4313-85f6-b5effcafa1e6/Screenshot_2024-09-19_at_2.55.22_PM.png)

[+] [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)

```yaml
## Example YAML
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```

## Pod Affinity/Anti-Affinity

This allows us to constraint Pods against labels on other Pods.

Affinity attracts pods to each other and anti-affinity pushes pods away from each other.

<aside> 💡

requiredDuringSchedulingIgnoredDuringExecution: hard limit

preferredDuringSchedulingIgnoredDuringExecution: soft limit

</aside>

> Having a larger clusters with inter-pod affinity and anti-affinity can cause slow downs (I’m guessing this is due to scheduler having to check for all the conditions one by one. Like checking a long list of IF statements)

[+] [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)

```yaml
kubernetes.io/hostname=ip-192-168-1-158.ap-northeast-2.compute.internal
kubernetes.io/os=linux
node.kubernetes.io/instance-type=t3.medium
topology.ebs.csi.aws.com/zone=ap-northeast-2a
topology.kubernetes.io/region=ap-northeast-2 <---
topology.kubernetes.io/zone=ap-northeast-2a <---

```

Say we schedule a pod named solo that we want it to be scheduled with other solos. If the pod is the first solo, then the scheduler will verify if the solo’s other requirements are met and then be scheduled. (It does so by checking if a similar pod exists in the cluster first)

![Screenshot 2024-09-19 at 3.12.11 PM.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2028682c-8c9a-469b-aca6-52abed699b3a/7a93790a-7201-4316-88a5-f9ecdca97569/Screenshot_2024-09-19_at_3.12.11_PM.png)

![Screenshot 2024-09-19 at 3.12.32 PM.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2028682c-8c9a-469b-aca6-52abed699b3a/5b08c91b-b8e0-4701-a24a-e41315bcd00f/Screenshot_2024-09-19_at_3.12.32_PM.png)

```yaml
# the topologykey we define allows us to define a constraint per nodes 
# in specific topology we select (e.g. say we define at restricting for
# nodes in zone ap-northeast-2a)

apiVersion: v1
kind: Pod
metadata:
  name: tommy
  labels:
    catname: tommy
spec:
  nodeSelector:
    catbowl: tommys
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: tommyfood
                operator: Exists
                values:
                  - fromm
                  - catit
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 50
          preference:
            matchExpressions:
              - key: tommyprefers
                operator: Exists
                values:
                  - churu
              - key: tommyprefers
                operator: DoesNotExist
                values:
                  - jerky
        - weight: 20
          preference:
            matchExpressions:
              - key: tommyfood
                operator: Exists
                values:
                  - Bluebuffalo
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: catname
                operator: In
                values:
                  - tommy
          topologyKey: topology.kubernetes.io/region
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            topologyKey: topology.kubernetes.io/region
            labelSelector:
              matchExpressions:
                - key: catname
                  operator: In
                  values:
                    - haku
  containers:
    - name: catbowl
      image: tommyscontainer:latest

```

Despite tommy2 being in the third catbowl, tommy will go to the bowl (node) scoring the highest

Where: ip-192-168-1-158.ap-northeast-2.compute.internal --> A ip-192-168-43-248.ap-northeast-2.compute.internal --> B ip-192-168-70-175.ap-northeast-2.compute.internal --> D ip-192-168-83-140.ap-northeast-2.compute.internal ---> E

## Pod Priority/Preemption

Priority: It allows us to define what pods are more important than others. It becomes useful in situations of limited resources in a node that multiple pods are trying to claim for their workload. Of course, higher priority will be prioritized and lower priority pods will be evicted from the node.

We define priority with a PriorityClass object.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: Never --> check in Preemption section for details
description: "hello world!"

```

Setting globalDefault to true means pods without a priorityClassName will also have the priorities applied to them

[+] [https://archive.eksworkshop.com/intermediate/201_resource_management/pod-priority/](https://archive.eksworkshop.com/intermediate/201_resource_management/pod-priority/)

[+] [https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority)

Say we have multiple pods scheduled in the scheduler’s queue at the moment, Then the pods with higher priority will be scheduled first.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: haku
spec:
  containers:
  - name: hakuplaytime
    image: StevensSchedule:latest
  priorityClassName: high-priority
```

Preemption: This Kubernetes mechanism triggers when no node is satisfying the requirements of a pod. And tries to find a way, in which lower pods are removed to make space, and get the higher priority pod in.

We can set a preemptionPolicy in the PriorityClass object.

- Never: place pod in scheduling queue ahead of lower-priority pods but they cannot preempt other pods
    - g. it is a non-preempting pod → cannot evict other pods to have itself scheduled on a node
    - They are also subject to schedule back-off (but at a lower frequency) should it not be able to be placed on a node
    - They can be preempted by other higher priority pods
- PreemptLowerPriority: these can preempt other pods

[+][https://github.com/kubernetes/kubernetes/blob/97332c1edca5be0082414d8a030a408f91bed003/pkg/kubelet/preemption/preemption.go#L4](https://github.com/kubernetes/kubernetes/blob/97332c1edca5be0082414d8a030a408f91bed003/pkg/kubelet/preemption/preemption.go#L4)

It makes use of a critical pod admission handler which handles the logic in how to decide to evict the pods such that there is the least amount of impact.  ) e.g. the fewest pods evicted  > the fewest amount of total requests from pods

Ordering of priorities:

1. minimal impact for pods
2. minimal impact for burstable pods → QoS
3. minimal impact for best effort pods → Qos

These priorities are measured by the QoS that we can provide for the pods in its object manifest

[+] [https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)

[+] [https://www.youtube.com/watch?v=-QAyxUWI7Fs](https://www.youtube.com/watch?v=-QAyxUWI7Fs)

When Pod A preempts one or more pod on its Node B, a nominatedNodeName annotation is set to be Node B.

- This is what the scheduler uses to track what resources are reserved for A and gives the users information about preemptions on the cluster

Pod A might not be scheduled on the nominated node N. While N would be tried first and then other nodes.

Say Pod B is preempted but has a long gracefulTerminationSeconds added. In this time, say another node M is freed up and meets the requirements for B, then pod B will be scheduled onto M.

[+] [https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#user-exposed-information](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#user-exposed-information)

→ Best Practice: K8s recommends setting lower termination periods for lower priority Pods to zero or a small number.

→ Best Practice: create inter-Pod affinity between pods with equal priority or ones with higher priorities compared to other pods.

<aside> 💡

Note!!

→ While scheduler will do its best to not disrupt PDB as best effort, but if it has to happen as a last resort, the pods (even with a PDB) with a lower priority will be preempted to make way for higher priority pods

→ In cases where Inter-Pod affinity is in play, if a pending Pod A has affinity to a lower-priority Pod B, C, etc... then the scheduler will not try to remove those pods with inter-affinity. Instead, it will just look for another node. (e.g. it is possible for lower priority pods to be evicted for a higher priority one)

→ Cross node pre-emptions do not happen

</aside>

## Resource Requests and Limits

### Requests - Limits Range

Limit Range: We can set limits of how much quotas pods and other objects can request at a namespace level.

Example: set min-max for CPU, min-max for PVC storage req.

- If we have 2 or more LimitRange objects in a namespace then seeking which value will be applied is non-deterministic (next state is at random)
- Limit Ranges apply at the Pod admission state (when pods request are intercepted to check before it is applied to the API Server) e.g. if we already have pods running, it will not apply.
- Limit Ranges do not check the consistency of a request parameter assigned to a Pod.

default: containers in the namespace will default to the “default” value

min: containers cannot have limits or request smaller than the “min“ value

max: containers cannot have limits or requests smaller than the “max” value.

[+] [https://kubernetes.io/docs/concepts/policy/limit-range/](https://kubernetes.io/docs/concepts/policy/limit-range/)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tommyslimit
  namespace: tommycpu
spec:
  limits:
  - default: # this section defines default limits
      cpu: 500m
    defaultRequest: # this section defines default requests
      cpu: 500m
    max: # max and min define the limit range
      cpu: "1"
    min:
      cpu: 100m
    type: Container
---
apiVersion: v1
kind: Pod
metadata:
  name: tommy
  namespace: tommycpu
spec:
  containers:
  - name: alpine
    image: alpine:latest
    resources:
     requests:
      cpu: 700m
     limits:
      cpu: 700m

```

---

Sidenote:

It seems that pods even when the limit is at 700m can be scheduled despite the limit set. However, if only the request exists, it will not be scheduled on the namespace.

This is because the “default” values we assign is not checked (whether this is consistently respected). You should define these under the “max” and “min” sections.

[>] Testing with:

[https://github.com/narmidm/k8s-pod-cpu-stressor](https://github.com/narmidm/k8s-pod-cpu-stressor)

---

### Resource Quotas

Resource Quotas: Resources quotas allows us to define constraints for limits of total resource consumption per namespace.  This gives more power to the Kubernetes administrator to dictate which team (if constrained by their own namespace) gets to have how much X resource.

Resources in questions can range to things such as:

- Compute usage
- Storage usage
- Object counts
- Quota Scopes

[+] [https://kubernetes.io/docs/concepts/policy/resource-quotas/](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

[+] Static cpu management [https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/)

### Pod Priority

Priority: It allows us to define what pods are more important than others. It becomes useful in situations of limited resources in a node that multiple pods are trying to claim for their workload. Of course, higher priority will be prioritized and lower priority pods will be evicted from the node.

```yaml

# We define priority with a PriorityClass object
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: Never # --> check in Preemption 
# section for details
description: "hello world!"
```

<aside> 💡

Setting globalDefault to true means pods without a priorityClassName will also have the priorities applied to them

[+] [https://archive.eksworkshop.com/intermediate/201_resource_management/pod-priority/](https://archive.eksworkshop.com/intermediate/201_resource_management/pod-priority/)

</aside>

[+]

[https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority)

Say we have multiple pods scheduled in the scheduler’s queue at the moment, Then the pods with higher priority will be scheduled first:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: haku
spec:
  containers:
  - name: hakuplaytime
    image: StevensSchedule:latest
  priorityClassName: high-priority
```

### Preemption

This Kubernetes mechanism triggers when no node is satisfying the requirements of a pod. And tries to find a way, in which lower pods are removed to make space, and get the higher priority pod in.

We can set a preemptionPolicy in the PriorityClass object.

- Never: place pod in scheduling queue ahead of lower-priority pods but they cannot preempt other pods
    - e.g. it is a non-preempting pod → cannot evict other pods to have itself scheduled on a node
    - They are also subject to schedule back-off (but at a lower frequency) should it not be able to be placed on a node
    - They can be preempted by other higher priority pods
- PreemptLowerPriority: these can preempt other pods

[*]

[https://github.com/kubernetes/kubernetes/blob/97332c1edca5be0082414d8a030a408f91bed003/pkg/kubelet/preemption/preemption.go#L4](https://github.com/kubernetes/kubernetes/blob/97332c1edca5be0082414d8a030a408f91bed003/pkg/kubelet/preemption/preemption.go#L4)

It makes use of a critical pod admission handler which handles the logic in how to decide to evict the pods such that there is the least amount of impact. )

e.g. the fewest pods evicted  > the fewest amount of total requests from pods

Ordering of priorities:

1. minimal impact for pods
2. minimal impact for burstable pods → QoS
3. minimal impact for best effort pods → Qos

These priorities are measured by the QoS that we can provide for the pods in its object manifest

[+] [https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)

[+] [https://www.youtube.com/watch?v=-QAyxUWI7Fs](https://www.youtube.com/watch?v=-QAyxUWI7Fs)

When Pod A preempts one or more pod on its Node B, a nominatedNodeName annotation is set to be Node B.

- This is what the scheduler uses to track what resources are reserved for A and gives the users information about preemptions on the cluster

Pod A might not be scheduled on the nominated node N. While N would be tried first and then other nodes.

Say Pod B is preempted but has a long gracefulTerminationSeconds added. In this time, say another node M is freed up and meets the requirements for B, then pod B will be scheduled onto M.

[+] [https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#user-exposed-information](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#user-exposed-information)

→ Best Practice: K8s recommends setting lower termination periods for lower priority Pods to zero or a small number.

→ Best Practice: create inter-Pod affinity between pods with equal priority or ones with higher priorities compared to other pods.

<aside> 💡

→ While scheduler will do its best to not disrupt PDB as best effort, but if it has to happen as a last resort, the pods (even with a PDB) with a lower priority will be preempted to make way for higher priority pods

→ In cases where Inter-Pod affinity is in play, if a pending Pod A has affinity to a lower-priority Pod B, C, etc... then the scheduler will not try to remove those pods with inter-affinity. Instead, it will just look for another node. (e.g. it is possible for lower priority pods to be evicted for a higher priority one)

→ Cross node preemptions do not happen

</aside>

## Using Multiple Schedulers

[Configure Multiple Schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)

- default-scheduler is configured by default from a file scheduler-config-yaml
- Additional schedulers must have a different name

```bash
# We can deploy the kubernetes provided binary for the scheduler
$ wget <https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-scheduler> 

# Or alternatively write our own
```

Nowadays, with kubeadm deployment, schedulers and control plane components run as pods on the control plane nodes.

How to deploy our custom scheduler:

```yaml
# Sample scheduler yaml file

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    component: scheduler
    tier: control-plane
  name: my-scheduler
  namespace: kube-system
spec:
  selector:
    matchLabels:
      component: scheduler
      tier: control-plane
  replicas: 1
  template:
    metadata:
      labels:
        component: scheduler
        tier: control-plane
        version: second
    spec:
      serviceAccountName: my-scheduler
      containers:
      - command:
        - /usr/local/bin/kube-scheduler
        # define the scheduler config 
        - --config=/etc/kubernetes/my-scheduler/my-scheduler-config.yaml
        image: gcr.io/my-gcp-project/my-kube-scheduler:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10259
            scheme: HTTPS
          initialDelaySeconds: 15
        name: kube-second-scheduler
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10259
            scheme: HTTPS
        resources:
          requests:
            cpu: '0.1'
        securityContext:
          privileged: false
        volumeMounts:
          - name: config-volume
            mountPath: /etc/kubernetes/my-scheduler
      hostNetwork: false
      hostPID: false
      volumes:
        - name: config-volume
          configMap:
            name: my-scheduler-config
```


## Admission Controllers

- A way to enforce more security measures than RBAC / authorization (ex: permit certain registries, enforce labels, etc.)
- e.g. the admission controllers performs additional operations before the request to create a new pod gets processed and the pod is created
- Built-in admission controllers
	- AlwaysPullImages (ensure image is always pulled)
	- DefaultStorageClass
		- Observes creation of PVCs and adds the default if not specified
	- EventRateLimit (limit flooding of req. to API Server)
	- NamespaceExists (reject req. to non-existent ns) 
	- NamespaceAutoProvision (not enabled by default) --> deprecated in favour of namespacelifecycle
		- Auto creates namespaces that does not exists (e.g. when calling `kubectl run pod --namespace not-existing`)

```BASH
# Viewing enabled Admissions Controllers
$ kube-apiserver -h | grep enable-admission-plugins
# for kubeadm clusters
$ kubectl exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins 
```

To add Admission Controllers, modify:
`kube-apiserver.service` or `/etc/kubernetes/manifests/kubeapiserver.yaml`

`--enable-admission-plugins` to enable admission controllers
`--disable-admission-plugins` to disable default enabled admissions controllers

## Validating and Mutating Admission Controllers
Validating Admission Controller: validates an object and determines to allow or deny

Ex: DefaultStorageClass
- Request goes to the admission controller, and if a storage class is not defined, then it will modify the request to add this "default" storage class.
- So when the pvc is created, a `StorageClass: default` is added
- This is what we call a mutating admission controllers

Mutating Admission Controller: Changes the request as per our specification
- Generally, mutating **runs first** before the validating controller. 

What if we want our own admission controller with our own mutation and validation?
- `MutatingAdmissionWebhook`
- `ValidatingAdmissionWebhook`

We can configure these webhooks to point to a server (internal to cluster or external server) which takes care of processing the request. 

After a request goes through, and all the admission controllers processes, the webhook triggers.
- Then a call to the admission webhook server is called, with an `AdmissionReview` kind object passed in .json format
	- Info about the object, who / when made the req. are all provided

The Admission Webhook Server responds back with an `AdmissionReviewed` object containing a `response` section.
- If the `allowed` field inside `response` is true, req. is allowed
- If false, then it is rejected

Step 1: Deploy your webhook server (API Server on any platform) on K8s as a deployment + svc or on an external platform 

Step 2: Create a `ValidatingWebhook` kind object 


`ValidatingWebhookConfiguration`
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
webhooks:
  - name: my-webhook.example.com
   
    clientConfig: <--- Where we define the location of our server
	    url: "https://some-external-server-example.com"

	clientConfig: <-- When the service is defined inside the cluster
		service:  
			  namespace: "my-webhook-namespace"
			  name: "name of webhook service"
		caBundle: "some tls cert bundles"

	rules: <-- allows us to define when the validating webhook is triggered
      - operations: ["CREATE"]
        apiGroups: ["*"]
        apiVersions: ["*"]
        resources: ["*"]
        scope: "*"
```






```
# Imperative Commands
```bash
# create a pod
kubectl run nginx --image=nginx --dry-run=client -o yaml

# create a deployment 
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml

# add replicas to a deployment 
kubectl scale deployment nginx --replicas=4 

# create a service of type ClusterIP
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml

#for this command, selector will not use pod labels and instead assume selector as app="pod name"
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml

#type nodeport, however, we cannot specify the nodePort
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml

#alternatively, similar to the clusterIP version (above above)
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml

```

```bash
# useful jq commands

#get labels of node
kubectl get node node01 -o json | jq '.metadata.labels'

# taint a node with a command
kubectl taint nodes <nodename> key=value:NoSchedule-

# get the taints applied on the nodes
kubectl get nodes -o json | jq '.items[].spec.taints'

kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints \\
--no-headers 
```

Useful Kubectl commands (gen. workflow)

```bash
# get the log life along with log stream using the -f flag
kubectl logs -f <some pod name>

# If we have multiple containers in the pod, we must specify one
kubectl logs -f <pod name> <container name>
```

Useful Linux commands for CKA troubleshooting control plane
```bash
/var/log/pods # nothing
crictl logs # nothing

# syslogs:
tail -f /var/log/syslog | grep apiserver

journalctl | grep apiserver

# checking CRI (contianer runtime interface) (this is the main protocol for comm. between kubelet and container runtime)
watch crictl ps


#Gives you useful dir to config files
$ find / | grep kubeadm
/var/cache/apt/archives/kubeadm_1.31.0-1.1_amd64.deb
/var/lib/kubelet/kubeadm-flags.env
/var/lib/dpkg/info/kubeadm.list
/var/lib/dpkg/info/kubeadm.md5sums
/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
/usr/bin/kubeadm
/usr/share/doc/kubeadm
/usr/share/doc/kubeadm/README.md
/usr/share/doc/kubeadm/LICENSE

## Check kubelet status and such 

$ service keubelt status
$ grep kubelet /var/log/syslog 
#or 
$ journalctl -u kubelet

#check config in:
$ /var/lib/kubelet/kubeadm-flags.env

service kubelet restart
service kubelet status
```

