---
layout: blog
title: "The ambassador pattern in Kubernetes: a hands-on tutorial"
draft: true # will be changed to date: YYYY-MM-DD before publication
slug: ambassador-pattern-tutorial
author: >
  Icaro Camelo
---

Say you have an application container that only knows how to open a plain TCP
connection to `localhost`. Maybe it's an off-the-shelf tool, maybe it's old
code nobody wants to touch, maybe you just don't want every service to
reimplement retries, TLS, and failover. You still need that container to talk
to a real backend somewhere in the cluster.

The **ambassador pattern** solves this by putting a small proxy container in
the same Pod as your application. The app keeps connecting to `localhost`; the
ambassador container is the only part of the system that knows where the real
backend lives. This tutorial builds one end to end: a Postgres-backed app
container that never learns a hostname other than `localhost`, and an
ambassador container that does the routing for it. You'll deploy it, verify
it's working, then swap the backend out from under the app without touching
the app container at all.

If you want the conceptual background first, see
[multi-container Pods](/docs/concepts/workloads/pods/multi-container-pods/#ambassador)
and [sidecar containers](/docs/concepts/workloads/pods/sidecar-containers/) in
the Kubernetes documentation. This post assumes you already have a cluster
(kind or minikube both work fine) and `kubectl` configured against it.

## What you'll build

One Pod, two containers, sharing a network namespace:

```
┌─────────────────── Pod: ambassador-demo ───────────────────┐
│                                                              │
│   main-app             connects to            ambassador    │
│  (psql client)  ──────  localhost:5432  ──────▶ (socat)      │
│                                                     │         │
└─────────────────────────────────────────────────── │ ───────┘
                                                       ▼
                                          Service: postgres-backend
                                                       │
                                                       ▼
                                          Deployment: postgres-backend
```

`main-app` never addresses `postgres-backend` directly. It only ever speaks to
`localhost:5432`. The `ambassador` container owns the actual Service name,
so that's the only place a backend change needs to show up.

## Step 1: Deploy the real backend

First, the thing the app actually needs to reach: a single Postgres instance,
exposed through a Service.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-backend
  template:
    metadata:
      labels:
        app: postgres-backend
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "demo-password"
        - name: POSTGRES_DB
          value: "demo"
        ports:
        - containerPort: 5432
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-backend
spec:
  selector:
    app: postgres-backend
  ports:
  - port: 5432
    targetPort: 5432
```

Save that as `postgres-backend.yaml` and apply it:

```shell
kubectl apply -f postgres-backend.yaml
kubectl rollout status deployment/postgres-backend
```

This is a throwaway, single-replica database for the tutorial; it has no
persistent volume, so don't use this manifest as-is for anything you care
about keeping.

## Step 2: Create the ambassador Pod

Now the Pod the app actually runs in. The ambassador is defined as a
**native sidecar container**: an entry under `initContainers` with
`restartPolicy: Always`, which the kubelet starts before the app container
and keeps running for the Pod's whole lifetime (this needs Kubernetes v1.29
or later; the `SidecarContainers` feature gate has been on by default since
then). It runs [`socat`](https://linux.die.net/man/1/socat) to forward every
connection it receives on port 5432 to the `postgres-backend` Service.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ambassador-demo
  labels:
    app: ambassador-demo
spec:
  initContainers:
  - name: ambassador
    image: docker.io/alpine/socat:1.8.0.3
    restartPolicy: Always
    args:
      - "tcp-listen:5432,fork,reuseaddr"
      - "tcp-connect:postgres-backend:5432"
    ports:
    - containerPort: 5432
  containers:
  - name: main-app
    image: postgres:16-alpine
    command:
      - sh
      - -c
      - |
        until pg_isready -h localhost -p 5432 -U postgres; do
          echo "waiting for the ambassador to be reachable..."
          sleep 2
        done
        PGPASSWORD=demo-password psql -h localhost -U postgres -d demo -c \
          "CREATE TABLE IF NOT EXISTS visits (id serial PRIMARY KEY, seen_at timestamptz DEFAULT now());"
        while true; do
          PGPASSWORD=demo-password psql -h localhost -U postgres -d demo -c \
            "INSERT INTO visits DEFAULT VALUES;"
          PGPASSWORD=demo-password psql -h localhost -U postgres -d demo -c \
            "SELECT count(*) AS total_visits FROM visits;"
          sleep 10
        done
```

Notice what `main-app` does *not* contain: no Service name, no connection
pooling, no retry loop beyond "wait until the port is open." Everything it
knows is `localhost:5432`. Apply it:

```shell
kubectl apply -f ambassador-pod.yaml
kubectl wait --for=condition=Ready pod/ambassador-demo --timeout=60s
```

## Step 3: Verify it's working

Follow the app container's logs:

```shell
kubectl logs -f ambassador-demo -c main-app
```

After a few iterations you should see the row count climbing every ten
seconds:

```
 total_visits
--------------
            1
(1 row)

 total_visits
--------------
            2
(1 row)
```

Each of those `INSERT`/`SELECT` pairs travelled: `main-app` →
(`localhost:5432`, inside the Pod) → `ambassador` container → the
`postgres-backend` Service → the actual database. You can confirm the
ambassador is the one doing the forwarding by checking its own logs:

```shell
kubectl logs ambassador-demo -c ambassador
```

## Step 4: Swap the backend without touching the app

This is the part the pattern is for. Say you're migrating to a new Postgres
instance: a different Deployment, maybe a different version, or a managed
database outside the cluster. Deploy it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-backend-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-backend-v2
  template:
    metadata:
      labels:
        app: postgres-backend-v2
    spec:
      containers:
      - name: postgres
        image: postgres:17-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "demo-password"
        - name: POSTGRES_DB
          value: "demo"
        ports:
        - containerPort: 5432
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-backend-v2
spec:
  selector:
    app: postgres-backend-v2
  ports:
  - port: 5432
    targetPort: 5432
```

```shell
kubectl apply -f postgres-backend-v2.yaml
kubectl rollout status deployment/postgres-backend-v2
```

Now point the ambassador at it. A running Pod's `initContainers` entries
aren't mutable in place, so re-create the Pod from a copy of
`ambassador-pod.yaml` with exactly one line changed, saved as
`ambassador-pod-v2.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ambassador-demo
  labels:
    app: ambassador-demo
spec:
  initContainers:
  - name: ambassador
    image: docker.io/alpine/socat:1.8.0.3
    restartPolicy: Always
    args:
      - "tcp-listen:5432,fork,reuseaddr"
      - "tcp-connect:postgres-backend-v2:5432"   # <- the only changed line
    ports:
    - containerPort: 5432
  containers:
  - name: main-app
    image: postgres:16-alpine
    command:
      - sh
      - -c
      - |
        until pg_isready -h localhost -p 5432 -U postgres; do
          echo "waiting for the ambassador to be reachable..."
          sleep 2
        done
        PGPASSWORD=demo-password psql -h localhost -U postgres -d demo -c \
          "CREATE TABLE IF NOT EXISTS visits (id serial PRIMARY KEY, seen_at timestamptz DEFAULT now());"
        while true; do
          PGPASSWORD=demo-password psql -h localhost -U postgres -d demo -c \
            "INSERT INTO visits DEFAULT VALUES;"
          PGPASSWORD=demo-password psql -h localhost -U postgres -d demo -c \
            "SELECT count(*) AS total_visits FROM visits;"
          sleep 10
        done
```

```shell
kubectl delete pod ambassador-demo
kubectl apply -f ambassador-pod-v2.yaml
kubectl logs -f ambassador-demo -c main-app
```

Compare the two files and the only difference is that one `tcp-connect`
argument. The `main-app` container block is byte-for-byte identical. The row
count starts over from 1, because `postgres-backend-v2` is a fresh database,
but the app never needed a code change, a rebuild, or even a restart signal
beyond the Pod recreation itself. It still just connects to `localhost:5432`
and has no idea a migration happened.

If you want to see this without a brief outage, you could instead update
only the ambassador's connection target via `kubectl set image` and a
feature-gated in-place resize, or front both backends with a single Service
and migrate traffic server-side; either is beyond the scope of this tutorial,
but the core point stands either way: the swap stayed entirely inside the
ambassador container.

## Step 5: Clean up

```shell
kubectl delete pod ambassador-demo
kubectl delete deployment postgres-backend postgres-backend-v2
kubectl delete service postgres-backend postgres-backend-v2
```

## When this is worth it

An ambassador container is one more process to run, one more hop for every
request, and one more container to monitor. For a single app that's free to
call a client library directly, that overhead usually isn't worth it. It
tends to pay off when:

- the app genuinely can't be changed (vendored binary, legacy code, a
  language without a good client library for the backend), or
- you want the connection logic — retries, TLS, backend selection — owned
  once, outside the app, so every app in the Pod gets it for free.

The [multi-container Pods](/docs/concepts/workloads/pods/multi-container-pods/)
page in the Kubernetes documentation covers the other common patterns
(sidecar, adapter) and has a section on good practices and anti-patterns
worth reading before you reach for any of them in production.
