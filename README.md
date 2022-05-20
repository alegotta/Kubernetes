# Kubernetes

A little research conducted over this tool. See the [presentation](kubernetes-pres/slides.pdf) for more details.

*Credits to [UniNA-Beamer](https://github.com/luistar/unina-beamer) for the theme*

## Overview

> Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation.

> Kubernetes provides you with a framework to run distributed systems resiliently. It takes care of scaling and failover for your application, provides deployment patterns, and more. For example, Kubernetes can easily manage a canary deployment for your system.

## History

[images/container_evolution.svg](Container evolution)

Initial release: 2014, Google
Now: Cloud Native Computing Foundation: Google, Red Hat, Intel, Cisco, Docker, VMWare

containerd: standalone container runtime. Donated by Docker
etcd: distributed key-value store
helm: package manager
prometheus: monitoring tool

## Features

- **Service discovery and load balancing**: expose container with IP/DNS; load balance and distribute the network traffic so that the deployment is stable
- **Storage orchestration**: automatically mount a storage system of your choice, such as local storages and public cloud providers
- **Automated rollouts and rollbacks**: describe the desired state for your deployed containers, and it can change the actual state to the desired state at a controlled rate.
- **Automatic bin packing** You provide Kubernetes with a cluster of nodes that it can use to run containerized tasks. You tell Kubernetes how much CPU and memory (RAM) each container needs. Kubernetes can fit containers onto your nodes to make the best use of your resources.
- **Self-healing**: restarts containers that fail, replaces containers, kills containers that don't respond to your user-defined health check, and doesn't advertise them to clients until they are ready to serve
- **Secret and configuration management**: store and manage sensitive information, such as passwords, OAuth tokens, and SSH keys. You can deploy and update secrets and application configuration without rebuilding your container images, and without exposing secrets in your stack configuration

## Architecture

### Object

General unit to represent desired state of a cluster.

- `spec`: desired state
- `status`: current state
- `kind`: type of object
- `metadata`: uniquely identify the object (name, generated UID, namespace -> isolate group of resources, labels, annotations -> available to pods)
- Hierarchy with owner

#### Reccomended Labels

Prefix: app.kubernetes.io

- name
- version
- instance
- component
- part-of
- managed-by
- created-vy

|Element|Description|Components|
|:--:|:--:|:--:|
|Cluster|Whole infrastructure|Every other element down below|
|Nodes|Machines that perform the actual work|Agent, Pods, Network Proxy|
|Pods|A single instance of a workload (group of containers with shared namespace and volumes|Container runtime (e.g. docker)|
|Workload|Application running on Kubernetes|
|Kubelet|Manager for a single node|
|Proxy|Allow intra-pods communication|
|PersistentVolumeClaim|Special kind of Pod which handles storage requests|
|Control Plane|Manages all Nodes|API Server, Key-Value store, Scheduler, Controllers|
|API Server|Expose the HTTP API to manage everything|
|etcd|Key-value store|
|Scheduler|Assign workload to Pods|
|(Cloud) Controller Manager|Check node status, Manage endponints (service+pod), manage tokens, establish services and routes|

## Workload

Several built-in workloads exist:
- `Deployment`: stateless applications (any pod is interchangeable)
- `ReplicaSet`
- `StatefulSet`: pods tracking state
- `Job` / `CronJob: tasks that run to completion and then stop

Pod lifecycle:
- pending
- running
- succeeded
- failed

Container lifecycle:
- waiting
- running
- terminated

Deployment status:
- progressing (scaling up or down)
- complete (all updated, no old replicas are running)
- fail (conf errors)

## Examples

### Commands

|Command|Description|
|:--:|:--:|
|`kubectl apply -f <file_or_url>`|Apply the specified to the infrastructure|
|`kubectl describe <type> <name>`|Show complete details of a given workload (e.g. deployment, pvc, ...)|
|`kubectl rollout status deployment/<name>`|Show status of deployment|
|`kubectl edit deployment/<name>`|Edit configuration by hand. Better modifying the file and re-applying|
|`kubectl rollout history deployment/<name>`|Show changeset of deployment|
|`kubectl rollout undo deployment/<name>`|Undo last edit|
|`kubectl scale deployment/<name> --replicas=[number]`|Manually scale|
|`kubectl autoscale deployment/<name> --min=[n] --max=[n] --cpu-percent=[]`|Manually scale|
|`kubectl get pods [-l app=<app_name>] --show-labels`|Show list of specified application|
|`kubectl delete deployment <name>`|Delete the specified deployment|

### Single Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.16.1
    #command:
    ports:
    - containerPort: 80
```

### Stateless Deployment

Mandatory:
- template (equal to Pod configuration)
- selector: specify pods labels

Others:
- replicas (default 1)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
        envFrom:
          - configMapRef:
            name: db-credentials
        ports:
        - containerPort: 80
          hostPort: 8080
```

### Autoscaling

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      target:
        type: AverageValue
        averageValue: 1k
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday;
# │ │ │ │ │                                   7 is also Sunday on some systems)
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```

### Secrets

kustomization.yaml:

```
secretGenerator:
- name: db-user-pass
  envs:
  - .env.secret
```

apply -k .

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp	# refers to app we are creating
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

### Stateful - Single Instance

#### Persistent Volume

- A **PersistentVolume** (PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. - A **PersistentVolumeClaim** (PVC) is a request for storage by a user. Claims can request specific size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany or ReadWriteMany, see AccessModes).


```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

### Stateful - Multiple Instances (one RW, others RO)

See [here](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/).

### Other Things

- Helm package manager
- [Bitnami production runtime](https://kubeprod.io/)
- [Kubefed](https://faun.pub/multi-cloud-multi-region-kubernetes-federation-part-1-3f2b5f7db62c)
- [Postgres Operator](https://access.crunchydata.com/documentation/postgres-operator/v5/)
