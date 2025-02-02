# Security Mechanisms in Kubernetes

```bash
Useful command (check .crt file in text format)

$ openssl x509 -in <cert>.crt -text -noout

$ openssl req -in <certificate signing request>.csr -noout -text

$ crictl ps -a
$ crictl logs <container id>

$ k create role developer --namespace=default --verb=list,create,delete --resource=pods

$ k create rolebinding dev-user-binding --namespace=default --role=developer --user=dev-user

$ kubectl auth can-i <create> <pod> --as <some user> -n <namespace>
```

The first line of defence comes from securing the API Server of the Kubernetes cluster, which is at the heart of all existing operations. 

Authentication
- Who can access the server?
	- Roles, Service accounts, etc...

Authorization
- What can they do?
	- RBAC
	- Node Authorization
	- WebHooks
	- etc...

TLS Certificates (encryption) is used to secure control plane communications. 

## Authentication
- For a given cluster, we can have multiple roles, such as an Admin, Dev teams, bots for automation and cleanup jobs, and so on. 

We have two sets of "Accounts" to assign for users
- Humans: devs, admins, ops
- Bots: Service Accounts

While we cannot create users via kubectl / Kubernetes API. However, this is not the case for Service Accounts (e.g. `kubectl create serviceaccount example`)

### Authentication Mechanisms
- Static Password File
- Static Token File
- Certificates
- Identity Services

Static authentication files (1 & 2)  !!! Note that these are deprecated post K8s version 1.19 and up
- a file containing the passwords per username ==> csv file with following columns: password | username | userid
	- We then pass it in as a flag in our api server's configuration `kube-apiserver.service` --> `--basic-auth-file=<my-file>.csv`
	- And then restart the api server service to apply the option.
	- Alternatively, if using kubeadm tool, add the new config flag in the yaml file located in: `/etc/kubernetes/manifests/kube-apiserver.yaml`
- we can also do the same if say we throw a curl command: 
	- `$ curl -v -k http://master-node-ip:6443/api/v1/pods -u "user1:password123"`

Static Token File 
- pass a user token defined file instead for csv file as ==> token | username | userid | group
	- `--token-auth-file=<my-token-file>.csv`
-  `$ curl -v -k http://master-node-ip:6443/api/v1/pods --header "Authorization: Bearer <my token>`

## TLS
- TLS certifications allows you to guarantee the trust between the client and server
	- If we use plain text credentials, then it is possible a malicious actor intercepts the request payload and steal that information. 
	- As a solution, we can encrypt these credentials in some unknown format, using a key, generating a random set of letters and numbers which represents our credentials
	- As so, for our server to recognize us, we also send a "copy" of my key to the server, to decrypt and read the message
		- But the hacker can intercept this key...
	- This is Symmetric encryption (same key to encrypt and decrypt data, and key must be exchanged between client and server)

### Encryptions
Asymmetric encryption:
- Makes use a Private Key and Public Key. Private Key allows you to unlock the public key,

![[Screenshot 2025-01-26 at 3.29.58 PM.png]]


Then how do you validate whatever you receive is legitimate / from the actual institution? 
- The server doesn't send the key itself, but a certificate which has the key in it.
	- This contains information about the issuer, location of the server, validity, DNS names, ... etc.

But... anyone can generate this certificate. This is where knowing who issued the certificate is important. 
- The browser usually comes built-in with a certificate issuer verifier 
- Ex: certificate authorities (CA), self-signed certificate, and so on

You generate a Certificate Signing Request (CSR)
```bash
openssl req -new -key my-key.key -out my-key.csr -sub "/C-US/ST=CA/O=MyOrg, Inc./CN=my-key.com"
```
You send this to the CA, and the CA signs this and then sends it back to you.

- The CAs have their own private and public keys. The Browser takes these public keys and verifies the information of CAs
- For internal websites, you can always use a private offering of the CAs (e.g. anywhere service)

### Key Naming Conventions
```bash
# Public Keys naming conventions
*.crt or *.pem

# Private Keys
*.key or *.key.pem
```


## TLS in Kubernetes

Important paths:
```BASH
# root certificate location
/var/lib/kubernetes/ca.pem and /var/lib/kubernetes/ca.crt

# API Server certificate location in CPlane node
/var/lib/kubernetes/apiserver.crt and /var/lib/kubernetes/apiserver.pem

#ETCD config. locations
/var/lib/kubernetes/etcd/
```

- Communication between the control plane components, and all between client and the api-server must be secured.

Kubernetes requires you to have at least 1 Certificate Authority for your cluster.  (We can have more than one as well)
![[Screenshot 2025-01-26 at 4.24.05 PM.png]]


### Creating Certificates in Kubernetes

```BASH
# create keys
$ openssl genrsa -out ca.key 2048

# create certificate signing request
$ openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

# Sign the certificate 
$ openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

How about generating the certificates for the users? e.g. Client certificates

```BASH
# create keys
$ openssl genrsa -out admin.key 2048

# create certificate signing request (specify the name of the user. CN can be anything we want.) (/O flag defines the user's group)
$ openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr

# Sign the certificate (We sign with ca key pair, which is valid for the Kubernetes cluster)
$ openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

How do we differentiate between say an admin user and another type of user? Groups (ex: SYSTEM:MASTERS)


Two ways to use the generated certificate
```BASH
#via REST API call

$ curl https://kube-apiserver:6443/api/v1/pods --key admin.key --cert admin.cert --cacert ca.crt

# via kube-config file
ex:
apiVersion: v1
kind: Config
clusters:
	
users:
- name: arn:aws:eks:ca-central-1:123456789:cluster/ubelab
  user:
    exec:
```


To generate the certificate file for the API Server, it gets a bit more tricky as we need to pass in more information. Remember that almost all components talks to the API Server within the Kubernetes cluster. We need to provide it things such as the alt. DNS names, IP of host, and so on.

```BASH
$ openssl genrsa -out ca.key 2048

# Simple, to add additional options, we create a .cnf file to pass in the info
$ openssl req -new -key admin.key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf 

vim openssl.cnf ->
[req]
...
[v3_req]
...
[alt_names]
# Notice how these are the DNS names we can find in the resolv.conf file in our nodes?
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kuberentes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local 
IP.1 = 10.0.0.1
IP.2 = 172.0.0.1

$ openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -out apiserver.crt

# Example components in API Server yaml config
```


Each node has its own certificate and key defined under `kubelet-config.yaml` file.
- Server certificates are named after their node name (API Server talks to it to get information on the worker node)
- Client certificates of the kubelet are generated to communicate with the API Server
	-  Since the nodes are system components, we need to give it a name as `sysmte:node:node#` to assign the right set of permissions

### Analyzing Details of certificates
There is 2 ways to create a K8s cluster; 

the hard way (deployed as system services)
- `cat /etc/systemd/system/kube-apiserver.service`

the kubeadm way
- Generates the components for you (generated as pods)
- cat `/etc/kubernetes/manifests/kube-apiserver.yaml`

Some useful troubleshooting steps
1. Use the `openssl x509 -in` command to inspect the certificate file (Subject, DNS names, expiry dates, Issuer)
2.  Inspect the Service Logs to troubleshoot further `journalctl -u etcd.service -l`
3. If deployed via kubeadm, use `kubectl logs etcd-<abc> -n kube-system`
4.  Check the container logs (now that K8s do not use docker engine, you can use crictl commands)

From Mumshad's CKA course:
![[Pasted image 20250126211032.png]]


## Managing Certificates and Certificates API 

1. User first creates a key and generates a certificates signing request via their key with their name and sends the req to an admin

2. Admin takes the key and creates a certificates signing object using an object of kind: CertificateSigningRequest (add the signing request from the user (with base 64 encoding) to the spec: request: secti1on)

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: newadmin
spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request:
  <certificate-goes-here>
```

```BASH
# Can get CSR token via
$ cat newadmin.csr |base64
$ kubectl create -f newadmin.yaml
```
  

3. This can be seen using the `kubectl get csr` command
4. Approval is done via `kubectl certificate approve <user>`
5.  Config certificate via `kubectl get csr newadmin -o yaml` or `echo "<certificate>" | base64 --decode`

Approval and Signing are tasks delegated to the controller-manager (in fact, you have define ca.crt and ca.pem for the controller manager definition)


## Kubeconfig
Directory: `$HOME/.kube/config`

[Kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) files allows you  organize information of your clusters, your assumed user as well as the authentication mecahnism. It's used by the local user (via kubectl) to authenticate to the API server.

Structure:
Clusters - Contexts - Users
```YAML

apiVersion: v1
kind: Config

clusters:
- name: arn:aws:eks:us-west-2:123456789:cluster/ubelab
  cluster:
	  certificate-authoriy: <directory-to-ca.crt>
	  certificate-authority-data: <...>.gr7.us-west-2.eks.amazonaws <-- alternatively, can define the token here. BUT certifiacte-authority key should be removed. using $ cat ca.crt | base64
	  server: https://my-apiserver-url:6443
  
contexts:
- context:     <-- Instructs which cluster to access via which user
    cluster: arn:aws:eks:us-west-2:123456789:cluster/ubelab
    user: arn:aws:eks:us-west-2:123456789:cluster/ubelab
  name: arn:aws:eks:us-west-2:123456789:cluster/ubelab
current-context: arn:aws:eks:us-west-2:123456789:cluster/ubelab.  <-- default context

users:
- name: arn:aws:eks:ca-central-1:123456789:cluster/ubelab
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
[...]
```



How to configure the kubeconfig file:
```BASH
# Use non-default kubeconfig file
$ kubectl config --kubeconfig=<my file dir> use-context <context i want to switch to>

## As per kubectl config help manual, best way to set a custom default is to modify the ~/.<env>rc file and source <env>rc file after.
```


# Authorization

## API Groups
The Kubernetes API consists of multiple APIs to define information about numerous components of the inner-working of the Kubernetes cluster

Core groups: where the core functionalities exists

![[Pasted image 20250127005510.png]]

Named groups: newer features which will be made available to these named groups
![[Pasted image 20250127005558.png]]


Link to the [official Kubernetes API documentation](https://kubernetes.io/docs/reference/kubernetes-api/) (RESTful interface)
You can also verify the existing API via `curl http://localhost:6443/apis -k --key <mykey>.key --cert <mycert>.crt --cacert ca.crt`
- Alternatively, run `kubectl proxy` command to access the apiserver then via `curl http://localhost:8001 -k`

## Assigning permissions to resources
While administering our cluster, we will want to assign certain permissions to specific groups of users, as well as certain external applications (service accounts)

There are difference mechanisms Kubernetes support
- Node 
	- Kubelet acts as the "API Server" of the node (its status). This is handled by the Node Authorizer (System:Node)
- ABAC
	- External access to the API (Attribute based access control)
	- https://kubernetes.io/docs/reference/access-authn-authz/abac/
	- Everytime we make a change. the policy file MUST be modified manually and then the API Server must be restarted (thus harder to manage)
- RBAC
	- Role based access policies
	- Like IAM, map users to specific roles (e.g. security admin, dev users, admins, ops, etc)
- Webhook
	- Outsource authorization (Open Policy agent) -> K8s makes the API call to some 3rd party 
- AlwaysAllow
	- Allows every reqs.
- AlwaysDeny
	- Denies every reqs.

These modes are set via flag on the APIServer `--authorization` <- we can have multiple modes.
- The mode is set to `AlwaysAllowy` by default. 
- If we have multiple modes delimited by a comma, all are checked sequentially as long as one mode denies the request.
- Ex: Say we have `--authorization-mode=Node,RBAC,Webhook`
	- Every time one module denies, the next is checked (e.g. is node denies, rbac is checked next, and so on...

### RBAC
We can create RBAC roles via a yaml manifest.

Ex:
```YAML
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  # add the namespace in the metadata in order to limit access per namespace
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "list", "update", "delete", "create"]
  resourceNames: ["app1", "app2"] # <-- only allow access to certain pod names
- apiGroups: [""]      <- for core groups, leave it as blank. Otherwise, add the name
  resources: ["ConfigMap"]
  verbs: ["create"]

--
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding. <-- rolebinding binds a role to a user object
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

https://www.cncf.io/blog/2018/08/01/demystifying-rbac-in-kubernetes/
- Note that a role is either delimited per namespace or it is cluster-wide. If a user needs to access multiple namespaces, deploy the same role in multiple namespaces and map it to the user. 

### Cluster Roles and Cluster Role Bindings
cluster roles allows you to have cluster-scoped resources (e.g. cluster admins, storage admins)

Note: cluster roles are not cluster dependent. This can be namespaced scoped. 

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["nodes"]
  verbs: ["get", "list", "delete", "create"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

Example of subjects as a kind: Group
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: example-group-binding
subjects:
- kind: Group
  name: example-group # This is the name of the group
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-role # The name of the ClusterRole you are binding to
  apiGroup: rbac.authorization.k8s.io
```

### Service Accounts
- Service accounts are used by some "machine" (application) to interact with the cluster (e.g. prometheus, jenkins, metricsserver)

When a serviceaccount is created, a token is also created automatically. Then a Secret object is created to store the token, and linked to the service account. 
- This is used by the application while authenticating to the K8s API. The token is stored as a secret object. 
- Token is used as an authentication bearer token to authenticate to the APIServer.
	- `curl https://<apiserver address>:6443/api -insecure --header "Authorization: Bearer <our SA token>`

However, since version `1.24`, the token secret is not automatically created. Once the service account is created, a token must also be as well via the creator. 
```BASH
$ kubectl create serviceaccount myserviceacc
$ kubectl create token <my service account name>
```


What if the Pod making use of the service account is hoted within the cluster?
- We can mount the token secret as a PVC on the application pod

For every namespace, a default service account and token exists. These are automatically mounted to a pod by default. 
```YAML
## k describe pod example

Name:             example
Namespace:        default
Priority:         0
Service Account:  default
Node:             node01/172.30.2.2
[...]
Containers:
  example:
    Container ID:   containerd://6bda1c293573bc75dd3883ea9770765a69633471adddd4bcafd442484bd1c180
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:0a399eb16751829e1af26fea27b20c3ec28d7ab1fb72182879dcae1cca21206a
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sat, 01 Feb 2025 22:59:21 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-25k4p (ro)
Volumes:
  kube-api-access-25k4p:    <--- SA admission controller created token via token-request API (1.22 +) 
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort

```

We can also verify the default secret mounted with 3 separate files:
```BASH
$ k exec -it example -- ls /var/run/secrets/kubernetes.io/serviceaccount
> ca.crt  namespace  token
```

```YAML
apiVersion: v1
kind: Pod
metadata: example
	name: example
spec:
	containers:
		- name: example
		  image: image:example
	
	serviceAccountName: myserviceaccountname <-- define the utilized SA
	
	automountServiceAccountToken: false <--- this means to not add the default SA token by default
```

Decoding the token:
```BASH
$ jq -R 'split(".") | select(length > 0) | .[0],.[1] | @base64d | fromjson' <<< <token>
```

via [jwt.io](https://jwt.io/)
```
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1769986754,   <--- since 1.24
  "iat": 1738450754,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "jti": "72267c26-3682-4b83-a72d-b0aed768e698",
  "kubernetes.io": {
    "namespace": "default",
    "node": {
      "name": "node01",
      "uid": "fd3b5e2a-1f85-485d-929b-4e2077b580c2"
    },
    "pod": {
      "name": "example",
      "uid": "ee0024c4-c4c3-4c5d-9845-04cf41139081"
    },
    "serviceaccount": {
      "name": "default",
      "uid": "eaff45ae-5e1a-4ebd-99ca-ec140e252538"
    },
    "warnafter": 1738454361
  },
  "nbf": 1738450754,
  "sub": "system:serviceaccount:default:default"
}
```

- Since version 1.22, when a new pod is created, the service account's secret token is not used anymore. Instead, a token with a defined "life time" is generated via the token request API from the SA admission controller. 

For non-expiring token, a new annotation must be now provided `kubernetes.io/service-account.name`
```YAML
apiVersion: v1
kind: Secret
metadata:
  name: secret-sa-sample
  annotations:
    kubernetes.io/service-account.name: "sa-name"
type: kubernetes.io/service-account-token
data:
  extra: YmFyCg==
```

More info from K8s docs [Service Account token Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#serviceaccount-token-secrets)

> In Kubernetes v1.22 and later, the recommended approach is to obtain a short-lived, automatically rotating ServiceAccount token by using the [`TokenRequest`](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/) API instead


## Image Security
Public Repository - Private Repository inside container registries

How do we implement authentication to private registries?

```YAML
$ kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \ 
  --docker-username=registry-user \
  --docker-password=registry-password \
  --docker-email=registry-user@org.com
```

For our pod:
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: private-registry.io/apps/internal-app
  imagePullSecrets:
  - name: regcred
```

## Security Contexts



Provide security contexts:

Pod level:
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
```

Container level:
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      capabilities: 
        add: ["MAC_ADMIN", "SYS_TIME", "NET_ADMIN"]
```

whichever security context you define for the container will overrule the definition given at the pod level


