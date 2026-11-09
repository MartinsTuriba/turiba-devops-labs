# Lab 8 · Run the whole app

GitHub user: martinsturiba · Fallback image used: no · Fallback Mission 7 files used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste your password, not even the base64 form. -->

## 1 · Database with storage

- 🔮 Before deleting the database Pod: will your todo still be there afterwards? I predicted yes, because the data folder `/var/lib/postgresql/data` is on the PVC `db-data`, and the new Pod mounts the same claim. What happened: it was still there. Right after `apply`, the PVC was `Pending`; it became `Bound` once the database Pod used it (WaitForFirstConsumer).

```text
$ kubectl -n course get pvc,pv
NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/db-data   Bound    pvc-b1a4d167-c360-481b-b130-4da7574d56ba   1Gi        RWO            standard       <unset>                 60s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-b1a4d167-c360-481b-b130-4da7574d56ba   1Gi        RWO            Delete           Bound    course/db-data   standard       <unset>                          43s

$ curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"survive a restart"}' localhost:8080/api/todos
{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-09T14:27:14.741Z"}

$ kubectl -n course get pods -l app=db        (before the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-864878fdcf-9pbbz   1/1     Running   0          60s

$ kubectl -n course delete pod db-864878fdcf-9pbbz
pod "db-864878fdcf-9pbbz" deleted from course namespace

$ kubectl -n course get pods -l app=db        (after the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-864878fdcf-l6sln   1/1     Running   0          1s

$ curl -s localhost:8080/api/todos
[{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-09T14:27:14.741Z"}]
```

## 2 · Connect the API

```text
$ kubectl -n course logs deployment/api -c wait-for-db
Found 3 pods, using pod/api-5bb66f57f-x6b99
db:5432 - no response
waiting for the database
db:5432 - no response
waiting for the database
db:5432 - accepting connections

$ kubectl -n course logs deployment/api -c api
Found 3 pods, using pod/api-5bb66f57f-x6b99
course-api 0.1.0 starting (Node.js v24.22.0)
API listening on port 5000
Database ready
```

- 🔮 Before looking: will `kubectl -n course get secret db-secret -o yaml` show your password? I predicted no, because it's called a Secret. What I saw, in words: the value is there under `data.password`, only base64-encoded. `base64 -d` turned it back into my practice password at once, with no key. So a Secret keeps the password out of Git, but anyone who can read Secrets in the namespace can read it.

## 3 · Health and resources

- 🔮 On my working copy only, I pointed the **liveness** probe of the API at `/nope` and applied it. Will those Pods get traffic? Will they be restarted? I predicted: no traffic and restarted. What happened: they were restarted about every 40 seconds (`RESTARTS 1, 1, 2`), but between restarts they were `1/1` and did get traffic, because readiness still asked `/healthz`. Liveness restarts; it doesn't take a Pod out of the Service.

```text
$ kubectl -n course get pods -l app=api       (about a minute later)
NAME                   READY   STATUS    RESTARTS      AGE
api-76d66657f5-6fjdm   1/1     Running   1 (33s ago)   73s
api-76d66657f5-88hzb   1/1     Running   1 (31s ago)   71s
api-76d66657f5-fp974   1/1     Running   2 (5s ago)    75s

$ kubectl -n course describe pod api-76d66657f5-88hzb         (only the Events)
  Normal   Killing    32s                kubelet  spec.containers{api}: Container api failed liveness probe, will be restarted
  Warning  Unhealthy  31s (x2 over 71s)  kubelet  spec.containers{api}: Readiness probe failed: Get "http://10.244.0.13:5000/healthz": dial tcp 10.244.0.13:5000: connect: connection refused
  Warning  Unhealthy  2s (x5 over 52s)   kubelet  spec.containers{api}: Liveness probe failed: HTTP probe failed with statuscode: 404
```

- How I made Git and the cluster agree again afterwards: `git restore k8s/` and `kubectl apply -f k8s/`; `rollout status` said "successfully rolled out".

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: the API Pods wouldn't connect to the database, because the diff touched `DB_HOST`. They didn't even start: the init container waited.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | A new API Pod stuck in `Init:0/1`, old Pods still serving | `kubectl -n course logs api-79d76799c9-7bj9c -c wait-for-db` → `database:5432 - no response`. After fixing the ConfigMap the Pod still waited (it keeps its old variables) until `kubectl -n course rollout restart deployment/api` | `1ce8cbb` fix(k8s): point DB_HOST back to the db Service |
| 2 | The next new Pod: `CreateContainerConfigError` | `kubectl -n course describe pod api-7599fffc8c-5zjht` → `Error: couldn't find key pasword in Secret course/db-secret` | `4ba1295` fix(k8s): read DB_PASSWORD from the key password |
| 3 | The next new Pod: `0/1 Running`, the rollout timed out | `kubectl -n course describe pod api-6755fd49b8-bzptf` → `Readiness probe failed: Get "http://10.244.0.22:5001/healthz": … connection refused` | `33efa26` fix(k8s): readiness probe on port 5000, where the API listens |
