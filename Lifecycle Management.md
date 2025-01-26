# Rolling Updates

Making use of rolling updates to incrementally update the pod with the newer container versions (one by one)

```bash
# Get status of rollout
$ kubectl rollout status deployment/my-deployment

# history of rollout
$ kubectl rollout history deployment/my-deployment

#undo a rollout
$ kubectl rollout undo deployment/my-deployment
```