---
title: Application Resource Requirements
summary: 
authors:
    - Kathy Barabash
date: 2022-01-13
---

# Application resource requirements

k8s [Application resource requirements](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) construct is used to provide to the kubernetes system information on how many resources should be ‘reserved’ for each container in the pods in a deployment and to constrain resource allocation. In addition, resource allocation quotas can be defined at namespace level.
K8s allows defining resource requirements for CPU and memory, in form of `request` to define the requisted resource amount and `limits` to define the maximal resource amount to be allocated. Both `requests` and `limits` can be defined for pods and containers while `limits` can also be defined for namespaces. `CPU requests` and `CPU limits` are expressed as a fraction of a core that is reserved for the container, either as a decimal number (0.1) or as a ‘milli’ value (100m); 0.1 and 100m are equivalent. `Memory requests` and `memory limits` are expressesed as a number of bytes of memory to be reserved for the container, either in 10-based units, e.g. E, P, T, G, M, K or in 2-based units, e.g. Ei, Pi, Ti, Gi, Mi, Ki. 

k8s scheduler works with the requests and limits to schedule pods across the cluster. The sum of the requests for all containers running on a node will never exceed the total available resources on that node. This means that on a 2 core machine, a maximum of 4 pods with a request of 500m will be scheduled.
The requests are reserved, and are not linked to the actual resource usage on a cluster. With incorrect requests, the cluster can remain underutilized.

## CPU requests and limits

The following example defines a pod with CPU limit of `1` CPU and CPU reservation of`0.5` CPU that runs stress container that tries to use up `2` CPUs:
```
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo
spec:
  containers:
  - name: cpu-demo-ctr
    image: vish/stress
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: "0.5"
    args:
    - -cpus
    - "2"
```
Resource usage can be verified with:
```
kubectl create -f cpu-requests.yaml
pod/cpu-demo created
kubectl top pods
NAME       CPU(cores)   MEMORY(bytes)
cpu-demo   990m         1Mi
```
When pod tries to exceed the limits, it keeps running but won't be given more cycles.

## Memory settings

The following defines a pod that needs `150`MB of memory while being allotted minimun of `100` and the maximum of `200`:
```
apiVersion: v1
kind: Pod
metadata:
  name: memory-low
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```
Run the following to watch the memory utilization:
```
kubectl create -f memory-low.yaml
kubectl top pods
NAME         CPU(cores)   MEMORY(bytes)
memory-low   40m          151Mi
```
And if we check a kubectl get pods, we’ll see zero restarts (as expected):
```
NAME         READY   STATUS    RESTARTS   AGE
memory-low   1/1     Running   0          3m56s
```
We can now edit the pod definition, and put a memory limit of 125Mi in place. This will create issues, and will cause kubernetes to kill and restart our pod.
```
apiVersion: v1
kind: Pod
metadata:
  name: memory-high
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      limits:
        memory: "125Mi"
      requests:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```
We can create this pod, and we can quickly see that I’ll get killed and restarted:
```
kubectl create -f memory-high.yaml
kubectl get pods --watch

NAME          READY   STATUS      RESTARTS   AGE
memory-high   0/1     OOMKilled   0          6s
memory-low    1/1     Running     0          6m
memory-high   0/1     OOMKilled   1          7s
memory-high   0/1     CrashLoopBackOff   1          8s
memory-high   0/1     OOMKilled          2          21s
```
As you can see, the container is killed due to exceeding it’s memory requirements. Notice how this is different from our earlier CPU scenario?

---