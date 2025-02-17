
# Networking Fundamentals in Kubernetes

## Pod Networking

In the context of ADD command standardized by the CNI 
- The container runtime creates the containers
- The runtime looks at the cni config in `etc/cni/net.d/net-script.conflist` to find our network script name
- Then with the name, it looks at the `/opt/cni/bin/net-script.sh` and executes the script via 
	- `./net-script.sh add <container> <namespace>`


### Container Network Interface (CNI)

Responsibilities:
- Support ADD/DEL/CHECK arguments for container IP allocations
- Support params container id, network namespaces, and so on
- Manage IP Address assignment to PODs (e.g. assign, ensure there's no IP conflict, delete, and so on)
- Return results in a standardized format

- The Container Runtime creates the network namespace
- Identifies the network to attach the container, runs the network plugin (bridge) when container is added via ADD, or deleted via DEL
- Tracks the network config in `.json` format

Numerous CNI solutions exists for K8s, and these are usually installed in path `opt/cni/bin` and which plugins to use are instructed in `/etc/cni/net.d`


```bash
#default directory for installed cni
/opt/cni/bin

# conf
/etc/cni/net.d
```

the `host-local` plugin is responsible for managing IP addresses.

Types of plugins for the CNI to use can be managed in directory `/etc/cni/net.d/net-script.conf`
```json
//example of cni in cka environment
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "log_file_path": "/var/log/calico/cni/cni.log",
      "datastore_type": "kubernetes",
      "nodename": "controlplane",
      "mtu": 0,
      "ipam": {
          "type": "host-local",
          "subnet": "usePodCidr"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    },
    {
      "type": "bandwidth",
      "capabilities": {"bandwidth": true}
    }
  ]
}
```

```bash
#check network config for cni 
$ ip addr show <cni>

#check the default gateway configured for a node
$ ssh <into a specific node>
$ ip route

# example output
#default via 172.25.0.1 dev eth1 
#10.244.0.0/16 dev weave proto kernel scope link src 10.244.192.0  <---
#172.25.0.0/24 dev eth1 proto kernel scope link src 172.25.0.82 
#192.1.121.0/24 dev eth0 proto kernel scope link src 192.1.121.12 
```


## Service Networking

At the bare level,  pods can communicate with each other via their IP addresses. You can also use Service objects to expose applications running in your cluster behind a single outward-facing endpoint.

The goal of using a SVC is that you do not need to modify your existing container applications to use an unfamiliar service discovery mechanism. (e.g. consider deployment pods where these IPs are transient and are not "fixed" for another pod to call via directly the IP)

The service gets an IP and a name assigned, which other pods can send requests to. Additionally, consider situations where we need to communicate between nodes. Well, services can communicate within all of the cluster. (note: bound within the cluster only) This is what we call a type `ClusterIP`

### ClusterIP
```YAML
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

### NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80
      # By default and for convenience, the `targetPort` is set to
      # the same value as the `port` field.
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane
      # will allocate a port from a range (default: 30000-32767)
      nodePort: 30007
```

NodePort gets an IP assigned to it, similarly to the type `ClusterIP`. But, it also exposes a port between **all nodes**. Each node in the cluster proxies the defined port into the created Service. NodePort allows you to set up your own load balancing solution, or even expose one or more nodes' IP addresses directly. 


## How are Services Created by the cluster?

![[Screenshot 2025-02-16 at 11.47.06 PM.png]]

1. Every time a Pod is created, kubelet agent running on each node will watch the API server for new requests and create a pod on it's node. It invokes the CNI plugin to assign an IP address for the Pod.

2. Kube-proxy also watches the API Server for Service objects to be created (services are cluster-wide)
	1. kube-proxy creates a forward rule in each node, to route the IP of the service to forward to the Pod's IP address, accessible in any node of the cluster
	2. kube-proxy will delete, create new rules when pods are evicted.

These "rules" are created by kube-proxy via
1. userspace
2. ipvs
3. iptables (default)

This is defined via `kube-proxy --proxy-mode [userspace | ipvs | iptables ]` flag

IP Range configured for services within a cluster can be found in `cat /etc/kubernetes/manifests/kube-apiserver.yaml` under flag `--service-cluster-ip-range`
(default: 10.0.0.0/24)

Note: CIDRs for Pods and SVCs **must not** overlap

```
$ iptables -L -t nat | grep <our svc>

$ cat /var/log/kube-proxy.log 
```


## DNS in Kubernetes

![[Screenshot 2025-02-17 at 12.15.20 AM.png]]
For each namespace, a dns subdomain is created for it. We can then further group it via the type `svc`, and further, the app is grouped via `cluster.local` (and all together gives us the FQDN)

SVC:
backend-svc->back->svc->cluster.local

Pods:
say back-end is IP 10.244.2.5
`curl http://10-244-2-5.apps.pod.cluster.local`

< pod IP > -> back -> pod -> cluster.local


## CoreDNS - How Kubernetes implements DNS

The easiest way to wire domains is to modify each pod's `etc/hosts`, but doing so for each and every one of them is not an optimal solution. So, the solution came to create a centralized DNS server and point all of these Pods to it via Kubelet modifying their `etc/resolv.conf` to add the nameserver at the dns server's IP address.

Remember that names are only mapped for services

the kube-dns is used to forward to CoreDNS.
```BASH
controlplane $ k get svc -A -o wide
NAMESPACE     NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE     SELECTOR
default       kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  5d12h   <none>
kube-system   kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   5d12h   k8s-app=kube-dns.  <--- the dns server
```

The CoreDNS pods are deployed within the kube-system namespace, running the CoreDNS executable.
- CoreDNS reuires a Corefile under `etc/coredns/Corefile`

Example coredns configmap (Corefile)
```
Name:         coredns
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
Corefile:
----
.:53 {
    errors
    health {
       lameduck 5s
    }
    # after "kubernetes" is the configured root domaoin
    ready      [root domain]
    kubernetes cluster.local in-addr.arpa ip6.arpa {.   <--- "kubernetes" plugin allows us to run CoreDNS in Kubernetes
       pods insecure    <-- (Any record that the DNS server cannot solve, it is forwarded in the nameservers specified in the CoreDNS pods etc/resolv.conf file)
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}

```


## Ingress in Kubernetes

