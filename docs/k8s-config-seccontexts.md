---
title: K8s Security Contexts
summary: 
authors:
    - Kathy Barabash
date: 2022-01-13
---

# Security Contexts

k8s [securitycontext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) construct is used to describe the privileges and access control that a pod will use. Multiple types of security contexts can be defined to:

1. __Discretionary Access Control__ define permissions to access an object, e.g. file, based on User ID (UID) and Group ID (GID)
1. __Security Enhanced Linux (SELinux)__ assign security labels to objects
1. __Running as privileged or unprivileged__
1. __Linux Capabilities__ control process privileges
1. __AppArmor__ define program profiles to restrict the capabilities of individual programs
1. __Seccomp__ filter a process’s system calls
1. __AllowPrivilegeEscalation__ control whether a process can gain more privileges than its parent process. This bool controls whether the `no_new_privs` flag gets set on the container process. `AllowPrivilegeEscalation` is true always when the container is: 1) run as Privileged OR 2) has CAP_SYS_ADMIN.

Most often, security contexts are defined for pods and container as shown in examples below:

## Pod with no security context

We’ll start of with creating a simple pod, without any security context. First, save this to `simple-security.yaml`:
```
apiVersion: v1
kind: Pod
metadata:
  name: simple-security
spec:
  containers:
    - name: basic
      image: nginx
```
Then deploy the pod  and then exec into it to see the running processes: 
```
kubectl create -f simple-security.yaml
kubectl exec -it simple-security sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sleep 3600
   59 root      0:00 sh
   64 root      0:00 ps aux
/ # id
uid=0(root) gid=0(root) groups=10(wheel)
/ # exit
```

## Pod level security context

Now, repeat the example with additional security context:
```
apiVersion: v1
kind: Pod
metadata:
  name: user-security
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: basic
      image: busybox
      command:
        - sleep
        - "3600"

kubectl create -f user-security.yaml
pod/user-security created
kubectl exec -it user-security sh
/ $ ps aux
PID   USER     TIME  COMMAND
    1 1000      0:00 sleep 3600
    6 1000      0:00 sh
   11 1000      0:00 ps aux
/ $ id
uid=1000 gid=3000 groups=2000
```

## Pod and container level security context

Let’s have a look at what we did here, we defined at the pod level a securitycontext. This means all containers in our pod will run as a certain user. Let’s have a look at what happens when we combine both, a pod securitycontext and one specifically on our container:
```
apiVersion: v1
kind: Pod
metadata:
  name: container-security
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: basic
      image: busybox
      command:
        - sleep
        - "3600"
      securityContext:
        runAsUser: 666

kubectl create -f container-security.yaml
pod/container-security created
kubectl exec -it container-security sh
/ $ ps aux
PID   USER     TIME  COMMAND
    1 666       0:00 sleep 3600
    6 666       0:00 sh
   11 666       0:00 ps aux
/ $ id
uid=666 gid=3000 groups=2000
```

We see an outcome as expected. We use the groupid and groups of the pod definition, but the more finegrained container policy is applied.

---
