The first line of defence comes from securing the API Server of the Kubernetes cluster, which is at the heart of all existing operations. 

Authentication
- Who can access the server?
	- Roles, Service accounts, etc...

Authorization
- What can they do?
	- RBAC
	- Node Authorization
	- Webhooks
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




```
Useful command (check .crt file in text format)

$ openssl x509 -in <cert>.crt -text -noout

$ openssl req -in <certificate signing request>.csr -noout -text

$ crictl ps -a
$ crictl logs <container id>
```
