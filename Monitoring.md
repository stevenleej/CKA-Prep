- We want to evaluate various things such as node CPU usage, memory usage, disk space and utilization, Network or even pod level metrics

In Kubernetes, since it does not come in with a built-in monitoring solution, there are a number of Open Source software such as Metrics server, Prometheus and Elastic Stack, as well as proprietary enterprise solutions such as Data Dog.

<aside> 💡

We can only have 1 Metrics Server deployed on our Kubernetes Cluster!

</aside>

Metrics Server retrieves metrics from the Nodes and Pods and aggregates them in memory. (It is an In-Memory monitoring solution)

- You will require other more complex solutions for Historical data

Metrics are generated via the Kubelet agent’s cAdvisor (Container Advisor)

- This component is responsible to make scrape performance metrics from pods and expose them through the Kubelet API

```yaml
# clone metrics server
$ git clone <https://github.com/kubernetes-incubator/metrics-server.git>

# deploy metrics server along with the required Kind
$ kubectl create -f metric-server/deploy/1.8+/

# You can now use these metrics
$ kubectl top nodes
$ kubectl top pod
```

