---
layout: blog
title: "Three multi-container Pod anti-patterns, and how to see them for yourself"
draft: true # will be changed to date: YYYY-MM-DD before publication
slug: multi-container-pod-anti-patterns
author: >
  Icaro Camelo
---

Multi-container Pods are useful exactly because a Pod gives every container
in it the same network namespace, the same volumes, and the same fate at
the scheduler's hands. That's also precisely how they go wrong: people reach
for a second container to solve a problem that sharing a Pod doesn't
actually solve, and the coupling that made the pattern useful starts working
against them.

This is the third post in a short series based on the Kubernetes
[multi-container Pods](/docs/concepts/workloads/pods/multi-container-pods/)
documentation; the first two covered the
[ambassador](/blog/2026/ambassador-pattern-tutorial/) and
[adapter](/blog/2026/adapter-pattern-tutorial/) patterns. This one covers
three things to avoid, and instead of just listing them, each one gets a
small manifest you can apply yourself and watch go wrong, then fixed.
You'll need a cluster (kind or minikube) and `kubectl`.

## 1. Bundling unrelated containers into one Pod

It's tempting to add a second, unrelated container to an existing Deployment
rather than write a whole new one. Here's a web server with a "nightly
report" container bolted on, both behind the same Service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-overloaded
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-overloaded
  template:
    metadata:
      labels:
        app: web-overloaded
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
      - name: nightly-report
        image: registry.k8s.io/busybox:1.27.2
        command: ["sh", "-c", "echo generating report; sleep 5; exit 1"]
---
apiVersion: v1
kind: Service
metadata:
  name: web-overloaded
spec:
  selector:
    app: web-overloaded
  ports:
  - port: 80
    targetPort: 80
```

The report container is written badly on purpose here (it exits 1 every
five seconds) to stand in for any unrelated container that occasionally
fails: a batch job, a one-off migration script, anything that isn't nginx.
Apply it and watch:

```shell
kubectl apply -f web-overloaded.yaml
kubectl get pods -l app=web-overloaded --watch
```

You'll see the Pod cycle `1/2 Running` → `1/2 CrashLoopBackOff`, never
reaching `2/2`. nginx itself never crashes. But check the Service:

```shell
kubectl get endpoints web-overloaded
```

While `nightly-report` is crash-looping, this comes back empty. A Pod only
counts as a Service endpoint while its `Ready` condition is true, and that
condition requires *every* container to be ready, not just the one
actually serving traffic. The unrelated container's bug just took a healthy
nginx out of load balancing.

**The fix** is to not share a Pod between things that don't need to: give
nginx its own Deployment and Service, and run the report as what it actually
is, a scheduled one-off task:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: nightly-report
            image: registry.k8s.io/busybox:1.27.2
            command: ["sh", "-c", "echo generating report; exit 1"]
```

Now the report container can fail as many times as it wants; `web`'s Service
endpoint doesn't know it exists.

## 2. Using a sidecar to fake a feature Kubernetes already has

Here's a Deployment where someone needed a periodic backup task and, rather
than learn about CronJobs, added a loop to a sidecar container:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-with-fake-cron
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-with-fake-cron
  template:
    metadata:
      labels:
        app: api-with-fake-cron
    spec:
      containers:
      - name: api
        image: registry.k8s.io/busybox:1.27.2
        command: ["sh", "-c", "while true; do sleep 3600; done"]
      - name: fake-cron
        image: registry.k8s.io/busybox:1.27.2
        command:
          - sh
          - -c
          - |
            # sleeping 20s instead of a day so you can see this in under a minute
            while true; do
              echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) running backup on $(hostname)"
              sleep 20
            done
```

Apply it at 3 replicas, which is a perfectly ordinary scaling decision for
the `api` container, and look at what the sidecar does:

```shell
kubectl apply -f fake-cron-deployment.yaml
kubectl logs -l app=api-with-fake-cron -c fake-cron --prefix --since=30s
```

```
[pod/api-with-fake-cron-7f9d9b6c4-abcde/fake-cron] 2026-10-13T11:02:01Z running backup on api-with-fake-cron-7f9d9b6c4-abcde
[pod/api-with-fake-cron-7f9d9b6c4-fghij/fake-cron] 2026-10-13T11:02:01Z running backup on api-with-fake-cron-7f9d9b6c4-fghij
[pod/api-with-fake-cron-7f9d9b6c4-klmno/fake-cron] 2026-10-13T11:02:01Z running backup on api-with-fake-cron-7f9d9b6c4-klmno
```

The backup ran three times, once per `api` replica, because the "scheduled
task" was never actually one task. It was a copy baked into every Pod.
Scale `api` to handle more traffic and you've just scaled your backup job to
match, for no reason connected to backups at all. Roll the Deployment and
the timer resets on every new Pod, so you lose any guarantee about when it
last ran, too.

**The fix** is to use the object Kubernetes already has for this: a
`CronJob`, decoupled entirely from how many `api` replicas exist:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: registry.k8s.io/busybox:1.27.2
        command: ["sh", "-c", "while true; do sleep 3600; done"]
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: backup
            image: registry.k8s.io/busybox:1.27.2
            command: ["sh", "-c", "echo running backup"]
```

Scale `api` from 3 replicas to 30 and the backup still runs exactly once per
schedule tick.

## 3. Leaving a sidecar's resources unset

This one doesn't crash anything, which is exactly why it's easy to miss. Two
near-identical Pods, differing only in whether the sidecar declares
resources:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-surprise
spec:
  containers:
  - name: app
    image: registry.k8s.io/busybox:1.27.2
    command: ["sh", "-c", "while true; do sleep 3600; done"]
    resources:
      requests: {cpu: 250m, memory: 128Mi}
      limits: {cpu: 250m, memory: 128Mi}
  initContainers:
  - name: log-shipper
    image: registry.k8s.io/busybox:1.27.2
    restartPolicy: Always
    command: ["sh", "-c", "while true; do sleep 3600; done"]
    # no resources block at all
```

```shell
kubectl apply -f qos-surprise.yaml
kubectl get pod qos-surprise -o jsonpath='{.status.qosClass}{"\n"}'
```

```
Burstable
```

The `app` container alone looks like a textbook `Guaranteed` Pod: CPU and
memory requests exactly matching limits. But a Pod's
[QoS class](/docs/concepts/workloads/pods/pod-qos/) is computed from *every*
container, sidecars and init containers included, and `log-shipper` has no
requests or limits at all, so the Pod as a whole doesn't qualify. That
matters under real node memory pressure: `Guaranteed` Pods are the last to
be evicted, `Burstable` Pods are evicted before them. A sidecar you forgot to
budget for can quietly move your whole Pod into an earlier eviction
tier, and its own memory use was never counted toward anything you planned
for in the first place.

**The fix** is to give the sidecar matching requests and limits too, sized
for what it actually needs. Save this as `qos-guaranteed.yaml`, identical
to `qos-surprise.yaml` above except for the `log-shipper` resources block:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
  - name: app
    image: registry.k8s.io/busybox:1.27.2
    command: ["sh", "-c", "while true; do sleep 3600; done"]
    resources:
      requests: {cpu: 250m, memory: 128Mi}
      limits: {cpu: 250m, memory: 128Mi}
  initContainers:
  - name: log-shipper
    image: registry.k8s.io/busybox:1.27.2
    restartPolicy: Always
    command: ["sh", "-c", "while true; do sleep 3600; done"]
    resources:
      requests: {cpu: 100m, memory: 64Mi}
      limits: {cpu: 100m, memory: 64Mi}
```

```shell
kubectl apply -f qos-guaranteed.yaml
kubectl get pod qos-guaranteed -o jsonpath='{.status.qosClass}{"\n"}'
```

```
Guaranteed
```

## Clean up

```shell
kubectl delete -f web-overloaded.yaml
kubectl delete deployment web api api-with-fake-cron
kubectl delete service web
kubectl delete cronjob nightly-report nightly-backup
kubectl delete pod qos-surprise qos-guaranteed
```

(Adjust to whichever of the manifests above you actually applied.)

None of these three are exotic. They come from reaching for the one
Kubernetes primitive you already know (a Pod with another container in
it) instead of the one actually built for the job: a separate Deployment
and Service, a CronJob, or just filling in a `resources` block. The
[multi-container Pods](/docs/concepts/workloads/pods/multi-container-pods/#good-practices-and-anti-patterns)
page has the short version of all three, plus the good practices that are
the flip side of this post, if you want the reference version to keep
nearby.
