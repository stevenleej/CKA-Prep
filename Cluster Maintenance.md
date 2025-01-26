
If a node is down for more than 5 minutes, K8s considers these nodes as dead.

The `Pod Eviction timeout` defines the time K8s waits for the pod to come back online (default 5min) This is set on the kube-controller-manager via the config: pod-eviction-timeout. Once the time passes, all the pods are evicted and the node is dead.

when you drain a node, the pods are gracefully terminated and rescheduled on another. Nodes are cordoned (not schedulable), and when you run uncordon node, the pods can be scheduled on it again

```bash
kubectl drain <nodename>
kubectl uncordon <nodename>

# this command prevents the node from having workloads scheduled 
# but the workload is not evicted
kubectl cordon <nodename>
```

pods that are moved do not automatically come back to the rebooted node.

## Cluster Upgrade Process

- K8s manages releases via version numberings
- Components can be of different versions. BUT, none of the versions should be higher than the API Servers
    - Controller-manager and kube-scheduler can be one version below
    - Kubelet and kube-proxy can be at most 2 versions down
    - kubectl can be 1 above or 1 lower in version

### Upgrade Strategies

### Strategy 1: Upgrade all at once

- But pods will be down and users will not have access to apps

### Strategy 2: Upgrade one node at a time


### Strategy 3: Add new node with new versions (easier on Cloud providers)


## Upgrade via kubeadm

- kubeadm has a command to help this

```bash
# list all components and what versions it can be upgraded to and give you the kubeadm command to upgrade
kubeadm upgrade plan

#note: you can only go one minor version at a time

#Example 1.11 to 1.13
apt-get upgrade -y kubeadm=1.12.0-00
kubeadm upgrade apply v1.12.0
```