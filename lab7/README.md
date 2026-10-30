# Lab 7 · Run it on a cluster

GitHub user: MartinsTuriba · Fallback image used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. -->

## 1 · Deploy

- 🔮 Before my first `apply`: how many Pods, and what their names start with? I predicted 3 Pods named `api-…` because of `replicas: 3`. What happened: 3 Pods named `api-7ff476cc44-…`: the Deployment name, then the ReplicaSet's hash.

```text
$ kubectl -n course get deploy,rs,pods
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api   3/3     3            3           23s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/api-7ff476cc44   3         3         3       23s

NAME                       READY   STATUS    RESTARTS   AGE
pod/api-7ff476cc44-2tl55   1/1     Running   0          23s
pod/api-7ff476cc44-drq78   1/1     Running   0          23s
pod/api-7ff476cc44-w6w64   1/1     Running   0          23s

$ curl -s localhost:8080/
{"name":"course-api","version":"0.1.0"}
```

## 2 · Heal and scale

- 🔮 After deleting one Pod, two seconds later I expected to see 3 Pods again, one with a new name, because the ReplicaSet keeps 3. What happened: exactly that, the new one was 2 s old.

```text
$ kubectl -n course get pods        (after the delete)
NAME                   READY   STATUS    RESTARTS   AGE
api-7ff476cc44-6jm8z   1/1     Running   0          2s
api-7ff476cc44-drq78   1/1     Running   0          29s
api-7ff476cc44-w6w64   1/1     Running   0          29s

$ kubectl -n course get pods        (with replicas: 5)
NAME                   READY   STATUS    RESTARTS   AGE
api-7ff476cc44-5jw67   1/1     Running   0          8s
api-7ff476cc44-6jm8z   1/1     Running   0          10s
api-7ff476cc44-cwmv2   1/1     Running   0          8s
api-7ff476cc44-drq78   1/1     Running   0          37s
api-7ff476cc44-w6w64   1/1     Running   0          37s
```

## 3 · Roll and roll back

- 3a: I rolled from the tag 0.2.0 to the tag a24ab0775c358e9a3fe77bfc8bc0379148a5af1b (CI's image for the newest main commit; I had to wait until that run on main was green).

```text
$ kubectl -n course rollout status deployment/api
Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 1 old replicas are pending termination...
deployment "api" successfully rolled out
```

- 3b: 🔮 With the tag 9.9.9, I predicted that all 3 old Pods keep running, because with 3 replicas maxUnavailable rounds down to 0 and the new Pod never becomes ready. What happened: 3 old Pods Running, 1 new Pod in ErrImagePull.

```text
$ kubectl -n course get pods        (during the stall)
NAME                   READY   STATUS         RESTARTS   AGE
api-66bfb5cf85-2w82j   0/1     ErrImagePull   0          55s
api-79f6f549cf-6q7nl   1/1     Running        0          61s
api-79f6f549cf-pf5n5   1/1     Running        0          57s
api-79f6f549cf-vbc5x   1/1     Running        0          57s

$ kubectl -n course rollout undo deployment/api
Warning: resource deployments/api was previously managed with 'kubectl apply'. Rolling back will not update the kubectl.kubernetes.io/last-applied-configuration annotation, which may cause unexpected behavior on future 'kubectl apply' operations. Consider using 'kubectl apply' with your previous configuration file instead.
deployment.apps/api rolled back
```

- What the `rollout undo` warning means, in my own words, and what I did so that Git and the cluster agree again: undo changed the cluster back, but kubectl still remembers 9.9.9 as the last thing I applied, and my file said 9.9.9 too; the next apply of that file would bring the broken tag back. I ran `git restore k8s/` and `kubectl apply -f k8s/`, so the file, the record and the cluster all say the newest main image again.

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: nothing, it looked like a tidy-up. It was the image.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | A new Pod stuck in ErrImagePull; the old three kept running | `kubectl -n course describe pod api-85fbbc455c-js8g7`: `Failed to pull image "…/api:0.2.O": … not found`. The tag has a letter O | b623eb5: back to the newest main image |
| 2 | All Pods Running, but nothing answered; port-forward printed nothing | `kubectl -n course get endpointslices -l kubernetes.io/service-name=api`: `ENDPOINTS <unset>`; `describe service api`: `Selector: app.kubernetes.io/name=api`, but the Pods are labelled `app: api` | 0769b6d: selector `app: api` |
| 3 | The EndpointSlice listed 3 Pods, but curl gave an empty reply (exit 52) | port-forward output: `Forwarding from 127.0.0.1:8080 -> 8080` then `connection refused`: the API listens on 5000 | f45409c: `targetPort: 5000` |
