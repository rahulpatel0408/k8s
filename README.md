# Kubernetes

Image->Pods->Deplyoment->service->external access

---
## Label and Selectors

### Labels:
A label is a key-value pair assigned to Kubernetes objects.
It helps in organizing and selecting resources.
```yaml
metadata:
  labels:
    app: web-server
    env: production
```
### Selectors:
A selector is used to filter resources based on labels.
It helps Kubernetes identify which resources should be affected.
```yaml
selector:
  matchLabels:
    app: web-server
```
Selector can be `matchLabel` or `matchExpresssion`.

In case of matchExpression there are 3 components:
- `key`: key of Label
- `operator`: selection criteria
  - `In`
  - `NotIn`
  - `Exists`
  - `DoesnotExist`
- `values`: list of values to filter from
  
```yaml
selector:
  matchExpressions:
    - key: env
      operator: In
      values: [production, staging]
```
---
## workloads
In Kubernetes, workloads refer to applications running on a cluster. They represent a set of processes that consume computing resources (CPU, memory, storage, etc.). Workloads are managed by Kubernetes using various controllers that ensure availability, scalability, and self-healing.

Types of Workloads:
- **ReplicaSet**
- **Deplyoment**
- **DaemonSet**
- **StatefulSet**
- **Jobs**
- **CronJobs**

### Deplyoment
A Deployment is used to manage replicated applications with rolling updates & rollbacks.
When updates are rolled, unlike ReplicaSet, all pods don't update at same time, few pods stay running throughout the update process.
```yaml
kind: Deplyoment
apiVersion: apps/v1
metadata:
  name: deplyoment-app
spec:
  replicas: 3 #change as per need
  selector:
    matchLabels:
      -app: my-app
  template:  # simply create pods here
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        -name: ngnix
         image: ngnix:latest
          ports:
            containerPort: 80
```
### ReplicaSet
same as deplyoment without rolling update feature. makes sure N pods are always running.
```yaml
kind: ReplicaSet
apiVersion: apps/v1
metadata:
 name: replicaSet-app
spec:
 replicas: 3
 selector:
  matchLabels:
    - app: my-app
 template:
  metadata:
    labels:
      app: my-app
  specs:
    containers:
      -name: ngnix
       image: ngnix:latest
       ports:
          containerPort: 80
```
### DaemonSet
A DaemonSet ensures one copy of a Pod runs on every node. Automatically creates a new Pod whenever a new node is added.
```yaml
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: daemon-app
spec:
  selector:
    matchLabels:
      - app: my-app
  template:
    metadata:
      labels: my-app
    spec:
      containers:
        - name: ngnix
          image: ngnix:latest
          ports:
            containerPort: 80

```
### StatefulSet

### Jobs
A Job runs a task once and stops.
```yaml
kind: Job
apiVersion: batch/v1
metadata:
  name: demo-job
spec:
  completions: 1
  parallelism: 1
  template:
    metadata:
      name: job-pod
      labels:
        app: job
    spec:
      containers:
        -name: batch-container
          image: busybox
          command: ["echo", "Hello K8s!"]
      restartPolicy: Never  #important to mention that it should terminate once completed
```
### CronJobs
A CronJob runs a Job on a schedule.
```yaml
kind: CronJob
apiVersion: batch/v1
metadata:
  name: demo-job
spec:
  schedule: "/2 * * * *" #every 2 minute 
  template:
    metadata:
      name: job-pod
      labels:
        app: job
    spec:
      containers:
        -name: batch-container
          image: busybox
          command:
          - sh
          - -c
          - >
            echo "hello k8s!";
      restartPolicy: Never  #can also use onFailure
```

`>` use this to write commands without using list 

---

## Storage
In kubernetes every container has its own temproray storage and it remains persistant as long as pod is running. 

Types:
- `Ephemeral Storage` (temporary, lost when Pod restarts)
- `Persistent Storage` (data survives Pod restarts)
  
### Ephemeral Storage
Ephemeral storage is tied to the lifecycle of a Pod. When the Pod is deleted, the storage is also lost.

- **emptyDir**:
 since every container has isolated storage we can create a shared storage volume for pods that is still temproray but allows data sharing between containers.
example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  volumes:
    - name: shared-storage
      emptyDir: {}  # Creates a shared directory
  containers:
    - name: writer
      image: busybox
      volumeMounts:
        - mountPath: /data
          name: shared-storage
      command: ["/bin/sh", "-c", "echo 'Shared Data' > /data/file.txt && sleep 3600"]

    - name: reader
      image: busybox
      volumeMounts:
        - mountPath: /data
          name: shared-storage
      command: ["/bin/sh", "-c", "cat /data/file.txt && sleep 3600"]

```

- **configMap**:
  A ConfigMap is used to store non-sensitive configuration data in key-value pairs. It allows you to decouple configuration from application code, making it easier to manage and modify configurations without **rebuilding container images**.
   







