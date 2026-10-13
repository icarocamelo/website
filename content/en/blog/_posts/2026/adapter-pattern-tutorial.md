---
layout: blog
title: "The adapter pattern in Kubernetes: a hands-on tutorial"
draft: true # will be changed to date: YYYY-MM-DD before publication
slug: adapter-pattern-tutorial
author: >
  Icaro Camelo
---

Say your application writes logs in whatever ad-hoc format it's always
written them in, and your log pipeline expects structured JSON lines. You
could rewrite the app's logging code. Or, if the app is something you don't
control, can't easily change, or just don't want to touch for a logging
format, you can't.

The **adapter pattern** handles this by adding a second container to the same
Pod whose only job is translation: it reads the main container's output in
whatever shape it's in, and re-emits it in the shape something else expects.
The app keeps writing logs exactly the way it always has. This tutorial
builds one: an app container that appends plain-text access log lines to a
shared file, and an adapter container that tails that file and turns each
line into a JSON object on its own stdout, ready for a log collector to pick
up.

For the conceptual background, see
[multi-container Pods](/docs/concepts/workloads/pods/multi-container-pods/#adapter)
in the Kubernetes documentation. This is the second in a short series of
tutorials based on that page; the first covered the
[ambassador pattern](/blog/2026/ambassador-pattern-tutorial/). You'll need a
cluster (kind or minikube both work) and `kubectl` configured against it.

## What you'll build

One Pod, two containers, sharing a volume instead of a network port this
time:

```
┌────────────────── Pod: adapter-demo ───────────────────────┐
│                                                               │
│   main-app                                      adapter       │
│  (writes plain-text    ───▶  shared emptyDir  ───▶ (tails the  │
│   access log lines)         volume: /var/log       file, emits │
│                              /app/access.log        JSON lines  │
│                                                      on stdout)  │
└───────────────────────────────────────────────────────────────┘
```

`main-app` only knows how to append lines to a file. It has no idea anything
is reading that file, let alone transforming it. The `adapter` container owns
all of the parsing and format conversion.

## Step 1: Write the Pod manifest

Save this as `adapter-pod.yaml`. Both containers use the small, dependency-free
`busybox` image so you don't need to build anything.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter-demo
  labels:
    app: adapter-demo
spec:
  initContainers:
  - name: adapter
    image: registry.k8s.io/busybox:1.27.2
    restartPolicy: Always
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
    command:
      - sh
      - -c
      - |
        touch /var/log/app/access.log
        tail -F -n0 /var/log/app/access.log | while read -r ts method path status duration; do
          duration_ms=${duration%ms}
          printf '{"timestamp":"%s","method":"%s","path":"%s","status":%s,"duration_ms":%s}\n' \
            "$ts" "$method" "$path" "$status" "$duration_ms"
        done
  containers:
  - name: main-app
    image: registry.k8s.io/busybox:1.27.2
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
    command:
      - sh
      - -c
      - |
        i=0
        while true; do
          i=$((i + 1))
          ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)
          case $((i % 4)) in
            0) method=GET;  path=/;         status=200; duration=12 ;;
            1) method=GET;  path=/checkout; status=200; duration=42 ;;
            2) method=POST; path=/cart;     status=201; duration=58 ;;
            3) method=GET;  path=/missing;  status=404; duration=7  ;;
          esac
          echo "$ts $method $path $status ${duration}ms" >> /var/log/app/access.log
          sleep 2
        done
  volumes:
  - name: shared-logs
    emptyDir: {}
```

`main-app` writes lines like this, which is exactly the kind of plain,
space-separated format a lot of small services default to:

```
2026-10-13T10:00:02Z GET /checkout 200 42ms
```

The `adapter` container is defined under `initContainers` with
`restartPolicy: Always`, which makes it a **native sidecar**: the kubelet
starts it before `main-app` and keeps it running for the Pod's whole
lifetime (this needs Kubernetes v1.29 or later). It `tail -F`s the shared
file and reshapes each line into JSON.

## Step 2: Deploy and watch it work

```shell
kubectl apply -f adapter-pod.yaml
kubectl wait --for=condition=Ready pod/adapter-demo --timeout=60s
```

Look at what the app container is actually writing:

```shell
kubectl exec adapter-demo -c main-app -- tail -n 4 /var/log/app/access.log
```

```
2026-10-13T10:01:02Z GET /checkout 200 42ms
2026-10-13T10:01:04Z POST /cart 201 58ms
2026-10-13T10:01:06Z GET /missing 404 7ms
2026-10-13T10:01:08Z GET / 200 12ms
```

Now look at what the adapter container is emitting from the same data:

```shell
kubectl logs -f adapter-demo -c adapter
```

```json
{"timestamp":"2026-10-13T10:01:02Z","method":"GET","path":"/checkout","status":200,"duration_ms":42}
{"timestamp":"2026-10-13T10:01:04Z","method":"POST","path":"/cart","status":201,"duration_ms":58}
{"timestamp":"2026-10-13T10:01:06Z","method":"GET","path":"/missing","status":404,"duration_ms":7}
{"timestamp":"2026-10-13T10:01:08Z","method":"GET","path":"/","status":200,"duration_ms":12}
```

Same events, two shapes. Because the adapter writes to its own stdout, any
collector that reads container logs the normal way (`kubectl logs`, a
node-level log agent, whatever your cluster already runs) picks up
structured JSON from this Pod without knowing or caring that the app itself
never produced JSON.

## Step 3: Confirm the app container is genuinely unaware

This is the point of the exercise, so it's worth checking directly. Open a
shell in the app container and confirm it has no access to, or knowledge of,
the adapter's output:

```shell
kubectl exec -it adapter-demo -c main-app -- sh -c "ps aux; cat /var/log/app/access.log | tail -n 1"
```

You'll see the app container's own process list (just its `sh` loop) and the
plain-text line it wrote. There's no JSON anywhere in its filesystem or
process tree. The transformation exists entirely in the other container.

## Step 4: Clean up

```shell
kubectl delete pod adapter-demo
```

## Adapter vs. a cluster-wide log processor

A natural question: why not just run Fluent Bit or a similar tool as a
DaemonSet and parse every Pod's logs centrally? That's a completely valid
default, and for a fleet of services emitting a handful of common log
formats, it's usually the better choice — one parsing config to maintain
instead of one adapter container per workload.

The adapter pattern earns its keep in narrower cases:

- the transformation is specific to one application and doesn't belong in a
  shared, cluster-wide config
- different replicas or versions of the same app emit different formats
  during a migration, and you want the translation versioned and deployed
  alongside the specific Pod spec that needs it
- you're adapting something other than logs — a metrics format, a health
  check protocol, a legacy API response shape — where a generic node-level
  agent wouldn't help at all

If none of those apply, prefer the node-level or cluster-wide tool. An
adapter container is still one more process per Pod to schedule, monitor,
and keep patched.

See the [multi-container Pods](/docs/concepts/workloads/pods/multi-container-pods/)
page for the other patterns covered in this series, including a section on
good practices and anti-patterns worth reading before using any of them in
production.
