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
  - A ConfigMap is used to store non-sensitive configuration data in key-value pairs. ConfigMap allows us to inject configuration values into Pods dynamically **without modifying the container image**. 
  - Is specific to a namespace.
  - Can be used in environment variables, command-line arguments, or mounted as files inside a pod.
    ### Creating ConfigMap:
    - using Yaml
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: my-config
      namespace: My-namespace #evry pod in this namespace can access it.
    data:
      APP_ENV: "production"
      LOG_LEVEL: "info"

    ```
    - using kubectl:
      
      `kubectl create configmap my-config --from-literal=database_url="postgres://db.example.com:5432" --from-literal=log_level="debug"`
      `kubectl get configmap my-config `
    - using a file:
      - env file:
        **cmd**: `kubectl create configmap my-config --from-env-file=config.properties`
        every line becomes a key value pair.

      - file:
        **cmd**: `kubectl create configmap my-config --from-file=config.properties`
        file name-> key | file-content->value
      
    ### Use:
    - **As env variables**:
      use `valueFrom` and `configMapKeyRef`.
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: env-configmap-pod
      spec:
        containers:
          - name: my-app
            image: busybox
            env:
              - name: DATABASE_URL
                valueFrom:
                  configMapKeyRef:
                    name: my-config
                    key: database_url
              - name: LOG_LEVEL
                valueFrom:
                  configMapKeyRef:
                    name: my-config
                    key: log_level
            command: ["sh", "-c", "echo $DATABASE_URL && echo $LOG_LEVEL && sleep 3600"]

      ```

    - **As Command Arguments**:
      same as env variable but with extra steps.
      ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-app
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: my-app
        template:
          metadata:
            labels:
              app: my-app
          spec:
            containers:
              - name: my-container
                image: my-app-image:latest
                command: ["my-app"]  # Command to run
                args:
                  - "--log-level=$(LOG_LEVEL)"
                  - "--max-connections=$(MAX_CONNECTIONS)"
                env:
                  - name: LOG_LEVEL
                    valueFrom:
                      configMapKeyRef:
                        name: my-config
                        key: log-level
                  - name: MAX_CONNECTIONS
                    valueFrom:
                      configMapKeyRef:
                        name: my-config
                        key: max-connections
      ```
      
    - **As Volume**
      creates a file for each key and saves its value as content.
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: volume-configmap-pod
      spec:
        volumes:
          - name: config-volume
            configMap:
              name: my-config
        containers:
          - name: my-app
            image: busybox
            volumeMounts:
              - mountPath: /config
                name: config-volume
            command: ["sh", "-c", "ls -l /config && cat /config/database_url && sleep 3600"]


      ```
- **secret**:
  Secrets are used to store and manage sensitive data securely, such as passwords, API keys, SSH keys, or TLS certificates. Unlike ConfigMaps, Secrets are base64-encoded and not exposed directly in plaintext.

  Types:
  - `Opaque (Default)` – Stores arbitrary key-value pairs.
  - `TLS Secret` – Stores TLS certificates and private keys.
  - `Docker Registry Secret` – Stores Docker authentication credentials.
  - `Basic Authentication Secret` – Stores username/password pairs.
  - `SSH Key Secret` – Stores SSH private/public keys.
  - `Service Account Token Secret` – Stores Kubernetes API tokens for service accounts.
 ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: my-secret
  type: Opaque
  data:
    username: dXNlcm5hbWU=  # base64("username")
    password: cGFzc3dvcmQ=  # base64("password")
  ```

 **cmd**: `kubectl create secret generic my-secret --from-literal=username=username --from-literal=password=password`

 similar to configMap, it can be used as env variable, function arguments and volume.

### Persistant Storage
- **Persistant Volume:**
A PV is a piece of storage in the cluster that has been provisioned by an administrator or dynamically created using a StorageClass. It represents actual storage that exists on a node or in external storage systems. It is simply a promise that certain storage will be made available once the claim is made.

- `Static Provisioning or Dynamic Provisioning`: whether to specify storage or let k8s automatically decide.
- `local (node-specific) or external (NFS, AWS EBS, etc.)`: hostPath or NFS(or others)
- `capacity, access modes, and reclaim policy`


    







