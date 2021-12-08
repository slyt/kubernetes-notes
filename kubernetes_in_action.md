# Kubernetes In Action Notes
This file contains notes about [Kubernetes In Action](https://learning.oreilly.com/library/view/kubernetes-in-action/9781617293726/)

## Chapter 1. Introducing Kubernetes
Kubernetes helps solve the problems created when going from monolithic software to complex and decoupled microservices.

_NoOps_: Have developers deploy applications themselves without knowing anything about the hardware infrastructure and without dealing with the ops team.

Containers run in the host's operating system, unlike VM's where processes run in separate operating systems. VM's use more resources because of the extra OS resources needed instead of containers. VM's are better for security because of their truly isolated nature.

Secret sauce to containers: `Linux Namespaces` to isolate system processes such as process ids and network interfaces and `cgroups` (Linux Control Groups) to isolate resources such as CPU and memory.

__Kubernetes__ was created by Google and open sourced based on learnings from their internal orchestration tools, `Borg` and `Omega`.

Kubernetes is composed of a master node and any number of worker nodes. K8s abstracts the underlying infrastructure as if all nodes are single, enormous computer.

__Control Plane__ controls the cluster and consists of multiple components that can run on a single master node or split across multiple nodes and replicated for high-availability.
1. __Kubernetes API Server__: How users communicate with Kubernetes.
2. __Scheduler__: Schedules how applications are deployed.
3. __Controller Manager__: Performs replication, tracks worker nodes, handles node failures...
4. __etcd__: distributed data store that persists the cluster configuration and state.

__Worker Nodes__ are the machines that run the containerized applications.
1. __Container Runtime__: Runs containers such as Docker, rkt, etc...
2. __Kubelet__: Talks to API server and manages containers on its node
3. __Kubernetes Service Proxy (kube-proxy)__: Load balances between applications.

Sometimes you do care about the underlying hardware (like SSH over HDD or GPU availability). In which case, you just tell Kubernetes that you want nodes with that type of resource. You don't need to go out and search for hardware that has those resources.

"If your infrastructure has enough spare resources to allow normal system operation even without the failed node, the ops team doesn’t even need to react to the failure immediately, such as at 3 a.m. They can sleep tight and deal with the failed node during regular work hours." -- nice!

## Chapter 2. First Steps with Docker and Kubernetes
Running busybox Docker image:
`docker run busybox echo "Hello world"`

Running `ps aux` within the container will show processes executing. Note that running `ps aux` on the host OS will also show processes running within the container, but with different ID, due to different namespace.

Mus t tag image with Docker Hub ID before pushing it to the repo:
`docker tag kubia fatsquirrel/kubia`


__Pods__ are groups of containers that are co-located. Each pod will run on the same worker node and in the same Linux namespace(s) with its own IP, hostname, processes...

__Service__: Used to expose Pod externally. Specificially __LoadBalancer__ for external access as opposed to __ClusterIP__ which is only accessible from inside the cluster.

Note that --generator has been deprecated, so run without:\
`$ kubectl run kubia --image=luksa/kubia --port=8080 --generator=run/v1`\
`$ kubectl run kubia --image=luksa/kubia --port=8080` Runs without a ReplicationController

Answer on Stack Overflow: [Error when trying to expose a docker container in minikube](https://stackoverflow.com/questions/64306744/error-when-trying-to-expose-a-docker-container-in-minikube).

Use LoadBalancer instead of `--generator`:\
`kubectl run kubia --image=luksa/kubia --port=8080`
`kubectl expose pod kubia --type=LoadBalancer --name kubia-http`\

_Note_: Minikube doesn’t support LoadBalancer services, so need to expose it via `minikube service kubia-http`.

Better yet, create a __Deployment__ instead:/
`kubectl create deployment <name_of_deployment> --image=<image_to_be_used>`/
`kubectl create deployment kubia-deployment --image=fatsquirrel/kubia`/
`kubectl expose deployment kubia-deployment --type=LoadBalancer --name kubia-http`

__ReplicationController__: Ensures that the pods you specify are always running.
__Service__: Solves the problem of ever-changing pod IP addreses and exposes multiple pods as a single contant IP and port. Services have static IP for the lifetime of the service wheareas pods are ephemeral.

## Chapter 3. Pods: running containers in Kubernetes

### 1. Introducing pods
- All containers of a pod run on the same node.
- Containers are designed to run only a single process per container (unless the process itself spawns child processes)
- Since all processes share the same stdout, it is difficult to figure out what process logged what.
- All containers in a pod run under the same Network and UTS (UNIX Time-Sharing) namespaces. They all share the same hostname and network interfaces.
- Filesystem of each container is fully isolated (not shared within the pod), unless you use `Volumes` (chapter 6)
- All containers in a pod share the same Network namespace, so they also share IP address and port space. Therefore you can have port conflict between two separate containers. They also share the same loopback interface, so a container can communicate with other containers in the same pod through localhost.
- There is no NAT between pods, the network is flat. So pods can communicate similar to computers on a LAN.
- By splitting components into pods, you can distribute them across multiple nodes and make use of all compute resources.
- If components scale differently (stateful backend database vs stateless frontend), then they should be in separate pods.
- Multiple containers in a single pod can happen if one main process is supported by complimentary processes (sidecar container).

### 2. Creating pods from YAML or JSON descriptors
- Need to know [Kubernetes API](https://kubernetes.io/docs/reference/) object definitions
- Get full YAML of a deployed pod: `kubectl get po kubia-zxzij -o yaml`
- Specifying port in pod spec isn't necessary because containers in a pod that are bound to 0.0.0.0 always allow other pods to connect to it. However, explicitely setting the port in the pod spec makes it clear for people using the cluster what port the pod uses. Setting the port allows you to assign a name to the port, which is handy.
- Can use `kubectl explain` instead of reading API docs to see which attributes are supported by various objects. (for example, `kubectl explain pod` or for more details, `kubectl explain pod.spec`)
- Can create pod via YAML file with `kubectl create -f kubia-manual.yaml`. Note that `kubectl create -f` creates any resource from a file, not just pods.
- Can get the full YAML of the pod with `kubectl get po kubia-manual -o yaml`. Can also get json with `-o json`.

## Chapter 4. Replication and other controllers: deploying managed pods

Main topics from Chapter 4:
- Liveness Probes
- ReplicationControllers
- ReplicaSets (replace ReplicationControllers)
- DaemonSets
- Jobs
- Scheduling Jobs


### 4.1 Keeping pods healthy (Liveness Probes)
How to check health of app from outside of the app? `Liveness probes` via

Different kinds of __Liveness probes__:
- __HTTP GET__ probe on the containers IP and specific port. 2xx and 3xx are success, otherwise restart container because it is unhealthy.
- __TCP Socket__ probe tries to open a socket and if unsuccessful, then restart container.
- __Exec probe__ executes arbitrary command in the container. If command exits with status code 0, then container is healthy, otherwise restart container.

Creating a container that throws 500 errors after 5 requests.

Can see probe failure events in Events section of describe output: `kubectl describe po kubia-liveness`:
```
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  9m37s                default-scheduler  Successfully assigned default/kubia-liveness to minikube  Normal   Pulled     3m11s                kubelet            Successfully pulled image "luksa/kubia-unhealthy" in 6m25.1481273s
  Normal   Killing    117s                 kubelet            Container kubia failed liveness probe, will be restarted  Normal   Pulling    87s (x2 over 9m37s)  kubelet            Pulling image "luksa/kubia-unhealthy"
  Normal   Created    71s (x2 over 3m11s)  kubelet            Created container kubia
  Normal   Started    71s (x2 over 3m11s)  kubelet            Started container kubia
  Normal   Pulled     71s                  kubelet            Successfully pulled image "luksa/kubia-unhealthy" in 16.5600803s
  Warning  Unhealthy  7s (x5 over 2m17s)   kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
```

Can see logs of why the previous container errors by doing: `kubectl logs mypod --previous`. This is necessary because K8's will restart the container, so you want to the logs of the one that crashed.

Other parameters for liveness probe that can be configured when defining the probe:
- `delay=0s`: probing begins immediately after container start. __Note that it is important to set the `initialDelaySeconds` so that your app has time to start up and properly respond to liveness probes.__
- `timeout=1s`: container must respond within 1 seconds to be considered healthy
- `period=10s`: The container is probed every 10 seconds.
- `failure=3`: Containers is restarted after 3 failed probes.

You can see the probe parameters in the `kubectl describe`:
```
    Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3
```

You can look at the exit codes in `kubectl describe` to see if the application was terminated externally (exit codes `137` or `143`). For example:
```
Containers:
  kubia:
    Container ID:   docker://c6f14ac7479810493c2de59375ea289a066f8149066c85949a5768f3a352bd5b
    Image:          luksa/kubia-unhealthy
    Image ID:       docker-pullable://luksa/kubia-unhealthy@sha256:5c746a42612be61209417d913030d97555cff0b8225092908c57634ad7c235f7
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 08 Dec 2021 10:03:02 -0600
    Last State:     Terminated
      Reason:       Error
      Exit Code:    137
      Started:      Wed, 08 Dec 2021 10:00:46 -0600
      Finished:     Wed, 08 Dec 2021 10:02:30 -0600
    Ready:          True
    Restart Count:  3
    Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-x8dcn (ro)
```

A simple liveness probe does wonders, but ideally you'd have a `/health` path where the app does an internal health check of it's own vital components and responds to Kubernetes. Ensure that `/health` does not require authentication, otherwise the probe will fail.

In order to keep apps independent, __do not throw liveness errors if a dependent service is unavailable__. For instance, the FE should not throw errored `/health` check if it cannot reach the dependent backend database as restarting the FE container will not fix the database issues.

Keep liveness probe computations simple. The probe's CPU time is counted in the containers CPU quota, so having a computationally intensive probe will take away from other processes in the container.

Liveness probes are conducted by `kubelet` process on the worker node itself with no intervention from the `kubernetes control plane`. however, if the entire node goes down, then the cotnrol plane will intervene, but only if pods are created via `ReplicationController` or similar mechanism, not if they were created manually.


### 4.2 Introducing ReplicationControllers
__ReplicationControllers__ is a Kubernetes resource that keeps its pods always running. For example, a managed pod exists on Node 1, but then Node 1 goes down, the ReplicationController will start a new pod on Node 2.

In general, ReplicationControllers are meant to manage multiple __replicas__ of a pod, hence the namesake.

__ReplicationControllers__ constantly monitor the number of pods of a given "type" and creates or removes pods to match the desired number.

__ReplicationControllers__ are deprecated and replaced with __ReplicaSets__. Usually __ReplicaSets__ are created automatically by the higher-level __Deployment__ resource (chapter 9).


### 4.3 Using ReplicaSets instead of Replication Controllers
__ReplicaSets__ can match pods by multiple labels (env=prod AND env=dev for example, or even all pods with key env=*).

`kubia-replicaset.yaml` example:
```
apiVersion: apps/v1beta2            
kind: ReplicaSet                    
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:                    
      app: kubia                    
  template:                         
    metadata:                       
      labels:                       
        app: kubia                 
    spec:                           
      containers:                  
      - name: kubia                 
        image: luksa/kubia          
```

This uses the `matchLabels` Selector, however there is a more expressive way to select labels that will be discussed later.

`apiVersion` containers two parts here: __API group__ (`apps`) and __API version__ `v1beta2`. Some Kubernetes resources are in the core API group, while newer Kubernetes versions have API groups to categorize the API.

Use `kubectl create` to create the ReplicaSet, the do `kubectl get rs` to get the ReplicaSet.

`ReplicaSets` can use the more expressive `matchExpressions` selector, for example:
```
selector:
  matchExpressions:
    - key: app                      
      operator: In                 
      values:                       
        - kubia              
```

Operators that can be used:
- `In`—Label’s value must match one of the specified values.
- `NotIn`—Label’s value must not match any of the specified values.
- `Exists`—Pod must include a label with the specified key (the value isn’t important). When using this operator, you shouldn’t specify the values field.
- `DoesNotExist`—Pod must not include a label with the specified key. The values property must not be specified.

You can specify multiple expressions and all must be true to match.

Deleting a ReplicaSet will delete the pods that it is overseeing: `kubectl delete rs kubia`


### 4.4 Running exactly one pod on each node with DaemonSets
ReplicationControllers and ReplicaSets are used for running a number of pods on ANY node. __DaemonSets__ can be used to deploy a SINGLE instance of a pod on all nodes.

__DaemonSets__ are typically used for infrastructure-related pods that perform system-level operations. Historically these types of processes would be run with init scripts or systemd daemon during node boot up. __DaemonSets__ doesn't have a notion of desired replica count, because it will run a single pod instance on all nodes.

By default, DaemonSet will run pods on all nodes, unless you specify a subset of pods with the `NodeSelector` property.

Note that nodes can be made un-schedulable, however DaemonSet will still deploy to these nodes because usually DaemonSet services are needed to run on all nodes.


Example would be to assign an `ssd` label to all nodes with `ssd` then in the __DaemonSet__ YAML definition, deploy an `ssd-monitor` process pod to each node with the `ssd` label. `ssd-monitor-daemonset.yaml`
```
apiVersion: apps/v1beta2           
kind: DaemonSet                    
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:                
        disk: ssd                  
      containers:
      - name: main
        image: luksa/ssd-monitor
```

- Can create a __DaemonSet__ with `kubectl create -f ssd-monitor-daemonset.yaml`
- Can see created __DaemonSet__ with `kubectl get ds`

Note that if you haven't actually added the `ssd` label to any nodes, then there will be no pods deployed for the __DaemonSet__ when you run `kubectl get po`. You can label nodes after the creating a __DaemonSet__ and pods will automatically be deployed to those nodes.

- Getting list of cluster nodes:
```
$ kubectl get node
NAME       STATUS    AGE       VERSION
minikube   Ready     4d        v1.6.0
```
- Adding a label to a node: ` kubectl label node minikube disk=ssd`

Now the __DaemonSet__ detects the new label on the node and deploys to the node:
```
$ kubectl get po
NAME                READY     STATUS        RESTARTS   AGE
ssd-monitor-hgxwq   1/1       Terminating   0          4m
```

If you remove/modify a label from a node, __DaemonSet__ automatically removes the pod if the label no longer matches.
```
$ kubectl label node minikube disk=hdd --overwrite
node "minikube" labeled

$ kubectl get po
NAME                READY     STATUS        RESTARTS   AGE
ssd-monitor-hgxwq   1/1       Terminating   0          4m
```

Removing a __DaemonSet__ also removes the pods that it was responsible for handling:
```
$ kubectl delete ds ssd-monitor
daemonset.apps "ssd-monitor" deleted
```

### 4.5 Running pods that perform as single completable task
The __Job__ resource runs tasks that are _completable_ and should not be restarted when the process in the pods container exits. If a node fails, __Jobs__ are still reschedules on alternate nodes. If a __Job__ has an error exit code, you can configure it to restart or just exit.

Example definition of a __Job__:
```
apiVersion: batch/v1                  
kind: Job                            
metadata:
  name: batch-job
spec:                                 
  template:
    metadata:
      labels:                         
        app: batch-job               
    spec:
      restartPolicy: OnFailure        
      containers:
      - name: main
        image: luksa/batch-job
```

Note that __Jobs__ cannot use the default restart policy of `Always`, therefore you must explicitly specify the restart policy for __Jobs__ as either `OnFailure` or `Never`.

Once the __Job__ completes you won't be able to see it in the `kubectl get pods` command unless you specify the `--show-all` (`-a`) switch.

__Jobs__ Jobs can be configured to run _sequentially_ or in _parallel_ in the Job spec.

Example Job spec of 5 _sequential_ completions:
```
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5                                
  template:
    <template is the same as above>
```

Example Job spec of 5 total _sequential_ completions but with 2 pods allowed to be run in _parallel_ :
```apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5                      ❶
  parallelism: 2                      ❷
  template:
    <template is same as above>
```

You can __Scale__ a Job while it is running by changing the job spec. This will immediately add more pods in parallel:
```
$ kubectl scale job multi-completion-batch-job --replicas 3
job "multi-completion-batch-job" scaled
```

`activeDeadlineSeconds` property can be set in the Job spec will be marked as failed if it doesn't complete in time.

### 4.6 Scheduling Jobs to run periodically or once in the future
Similar to `CronJob` in Linux, Kubernetes __Jobs__ can be scheduled periodically or in some time in the future using the __CronJob__ resource. Note that 'cron' comes from the Greek word "Chronos" (aka "time").

Example CronJob resource spec:
```
apiVersion: batch/v1beta1                  
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"           
  jobTemplate:
    spec:
      template:                            
        metadata:                          
          labels:                          
            app: periodic-batch-job        
        spec:                              
          restartPolicy: OnFailure         
          containers:                      
          - name: main                     
            image: luksa/batch-job   
```

Remember that Cron Schedule goes: "Minute, Hour, Day of Month, Month, Day of Week"

There is a `startingDeadlineSeconds` property that can be set in `CronJob` spec that will prevent the Job from running (show `Failed`) if it cannot start within the configured time.

`ChronJobs` should be __idempotent__ meaning that running them multiple times instead of once will not cause undefined behavior.
