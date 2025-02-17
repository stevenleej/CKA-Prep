
# Network Policies

## Traffic
[K8s docs](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

Ingress: Incoming traffic to current
Egress: Outgoing traffic from current

```YAML
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
      - protocol: TCP
        port: 8080
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
      - protocol: TCP
        port: 3306
  - ports: <--- applied on the internal pod itself
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP

```

We link a network policy to a pod and create a set of rules for it. 

### Ingress

Consider a DB pod opened in port 3306 which accepts incoming communication from other pods
```YAML
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db <-- give a label to the pod that we will match our networking policy with
  policyTypes: <-- define our rules
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api-pod
    ports: <--- port to allow traffic on
    - protocol: TCP
      port: 3306
```

Note: ingress of egress isolation only comes to effect if you have ingress / egress in the policy type.  (e.g. in our policy above, only ingress traffic is affected)
-> for egress, in this case, you also need to explicitly add it. 

What about in the case where the same labels from other pods exist in other namespaces?
- We can add a `namespaceSelector` field

```YAML
  ingress:
  - from:
    - podSelector:  <-- specify ingress rule on these pods
        matchLabels:
          role: api-pod
     namespaceSelector:
	     matchLabels:
			name: prod <--- match the namespace of prod 
these two matchers works like an AND operation. Both podSelector and namespaceSelector rules MUST pass
    ports: 
    - protocol: TCP
      port: 3306
```

We can also add de-limits traffic of IP addresses
```YAML
  ingress:
  - from:
	- ipBlock:
		cidr: 192.168.5.10/32
    ports: 
    - protocol: TCP
      port: 3306
```

When we stack these rules, it acts as an OR operation. 

```YAML
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api-pod
    - namespaceSelector:
	     matchLabels:
			name: 
	- ipBlock:
		cidr: 192.168.5.10/32
    ports: 
    - protocol: TCP
      port: 3306
```
In this case, each works as an OR operation (separated by the `-` in front)


### Egress
```YAML
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: db-policy
spec:
	podSelector:
	    matchLabels:
	      role: db 
	policyTypes: <-- define both of our types 
	- Ingress
	- Egress 
	ingress:
	- from:
    - podSelector:
        matchLabels:
          role: api-pod
	ports: 
	- protocol: TCP
      port: 3306
	egress:
	- to: <-- since we're defining traffic going outside
		- ipBlock:
			  cidr: 192.168.5.10/32
    ports: 
    - protocol: TCP
      port: 80
```


Network policies are additive. If there is any NetworkPolicy allowing a certain type of traffic, that traffic will be allowed even if another Network Policy would block it. This is due to the principle of **explicit allow** over **implicit deny** 

A good tip to setting up Network Policies are to ensure deny all, and then allow granularly traffic per design of workloads

## Examples

Default Deny all policy
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

This default policy ensures no unauthorized access occurs in any of the pods (in this case, in the NS default)


```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress-null
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress: [] ---> this is "null", denoting no traffic
```

