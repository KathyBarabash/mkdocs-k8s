
# K8s Configuration

- ConfigMaps
- SecurityContexts
- Application resource requirements
- Secrets
- ServiceAccount

## Configmaps

[ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) are used to store non secret configuration data in the k8s server and to deliver it to the containers, while separating the configuration data from other resource definitions. ConfigMaps can deliver configuration that can be encoded as key-value pairs, e.g. command-line arguments, environment variables, ports, etc., and can be defined in several different ways: from literal, from .config file, with yaml, and using kustomization.yaml. Below are examples of using different ways to deliver environment variables `YEAR` with  value `2022` and `MONTH` with value `Jan` to a pod. We start with a single variable and then proceed extending to a number of variables.

### __From literal__
First, create a configmap named `date-from-literal` with a single key-value pair `YEAR=2022`:

```
kubectl create configmap date-from-literal --from-literal=YEAR=2022
```
Now specify a pod with a reference to the `date-from-literal` configmap in `simplepodconfig-from-literal.yaml`:
```
apiVersion: v1
kind: Pod
metadata:
  name: config-literal
spec:
  containers:
    - name: basic
      image: nginx
      env:
        - name: YEAR
          valueFrom:
            configMapKeyRef:
              name: date-from-literal
              key: YEAR

```
And create the pod:
```
kubectl create -f simplepodconfig-from-literal.yaml
```
VWe now can verify the environment variable is set as specified:
```
kubectl exec -it config-literal /bin/bash
echo $YEAR      #This is run form withing the container and should return 2022
```
> __Question__: Can multiple variables be defined with `--from-literal`

### __From .config file__

First, create .config file with the key-value pair:
```
echo 'YEAR=2022' > date.config
```
Create configmap named `date-from-file` from `date.config` file:
```
kubectl create configmap date-from-file --from-file=date.config
```
Now, specify a pod referencing `date-from-file` configmap in  a `simplepodconfig-from-file.yaml`:
```
apiVersion: v1
kind: Pod
metadata:
  name: config-file
spec:
  containers:
    - name: basic
      image: nginx
      env:
        - name: YEAR
          valueFrom:
            configMapKeyRef:
              name: date-from-file
              key: date.config
```
Deploy the pod:
```
kubectl apply -f simplepodconfig-from-file.yaml
```
Then exec into the container to get the value of the `YEAR` environmental variable:
```
kubectl exec -it config-file /bin/bash
echo $YEAR
```
Here, the output is `YEAR=2022` and not just `2022` as expected meaning that with this method we deliver the content of the file not the variable assignemnt. To deliver environment variable as desired, we need to create configmap with `--from-env-file` flag: 
```
kubectl create configmap date-from-envfile --from-env-file=date.config
```
This new config called `date-from-envfile` can now be referenced in pod specification files, e.g. `simplepodconfig-from-envfile.yaml`:
```
apiVersion: v1
kind: Pod
metadata:
  name: env-file
spec:
  containers:
    - name: basic
      image: nginx
      env:
        - name: YEAR
          valueFrom:
            configMapKeyRef:
              name: date-from-envfile
              key: YEAR

kubectl apply -f simplepodconfig-envfile.yaml
kubectl exec -it env-file /bin/bash
echo $YEAR
```
You can use the following `describe` command to see the differences between the configmaps created with the different kubectl flag:
```
kubectl describe configmap/date-fromfile configmap/date-from-envfile configmap/date-from-literal
```

Now lets see how to multiple environment variables can be set. First, we add another value to date.config, and recreate the all configmaps:
```
echo 'MONTH=Jan' >> date.config

kubectl delete configmap/date-from-file configmap/date-from-envfile

kubectl create configmap date-from-file --from-file=date.config
kubectl create configmap date-from-envfile --from-env-file=date.config
kubectl describe configmap/date-from-file configmap/date-from-envfile

```
We see again that `--from-file` flag actually loads the file contents while  `--from-env-file` loads the assignes multiple varieables defined in the config file.

### __From YAML File__

Configmaps can also be specified like other k8s resources, with YAML files.
First, specify the `data-config-from-yaml` configmap in `configmap.yaml` file:
```
kind: ConfigMap 
apiVersion: v1 
metadata:
  name: data-config-from-yaml
data:
  myname: this-is-yaml
```
And then create the `data-config-from-yaml` configmap by running:
```
kubectl apply -f configmap.yaml
```
This new config called `data-config-from-yaml` can now be referenced in pod specification files, e.g. `simplepodconfig-from-yaml.yaml`:
```
apiVersion: v1
kind: Pod
metadata:
  name: config-yaml
spec:
  containers:
    - name: basic
      image: nginx
      env:
        - name: MYNAMEFROMYAML
          valueFrom:
            configMapKeyRef:
              name: config-from-yaml
              key: myname
```
```
kubectl apply -f simplepodyaml.yaml
```
Test with:
```
kubectl exec -it config-yaml /bin/bash
echo $MYNAMEFROMYAML
```
The output should be
```
this-is-yaml
```

## SecurityContexts

A securitycontext describes the privileges and access control that a pod will use. Some settings here include (from the kubernetes docs):

    Discretionary Access Control: Permission to access an object, like a file, is based on user ID (UID) and group ID (GID).
    Security Enhanced Linux (SELinux): Objects are assigned security labels.
    Running as privileged or unprivileged.
    Linux Capabilities: Give a process some privileges, but not all the privileges of the root user.
    AppArmor: Use program profiles to restrict the capabilities of individual programs.
    Seccomp: Filter a process’s system calls.
    AllowPrivilegeEscalation: Controls whether a process can gain more privileges than its parent process. This bool directly controls whether the no_new_privs flag gets set on the container process. AllowPrivilegeEscalation is true always when the container is: 1) run as Privileged OR 2) has CAP_SYS_ADMIN.

This is actually a very important topic to dive into. As we’re prepping for the exam, I don’t want to dive too deep into the details of all the settings and the topics. Let’s have a look at some of the settings.

We’ll start of with creating a simple pod, without a security context.

apiVersion: v1
kind: Pod
metadata:
  name: simple-security
spec:
  containers:
    - name: basic
      image: nginx

Let’s create this with the command kubectl create -f simple-security.yaml – and then exec into our container with kubectl exec -it simple-security sh

Once in our container, let’s have a look at our current processes and user. You’ll see everything is running as root, and you’re logged as root:

/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sleep 3600
   59 root      0:00 sh
   64 root      0:00 ps aux
/ # id
uid=0(root) gid=0(root) groups=10(wheel)
/ # exit

Let’s now change our pod definition, to include a user and group id:

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

Let’s now do the same: create the container, exec into it, and look at out processes and user.

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

Let’s have a look at what we did here, we defined at the pod level a securitycontext. This means all containers in our pod will run as a certain user. Let’s have a look at what happens when we combine both, a pod securitycontext and one specifically on our container:

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

We see an outcome as expected. We use the groupid and groups of the pod definition, but the more finegrained container policy is applied.
Define an application resource requirements

Application resource requirements – and constraints – are critical within a kubernetes application definition. It provides information to the kubernetes system on how many resources should be ‘reserved’ for each container in the pods in a deployment – and it can set maxima as well. Additionally, it also allows you to set quota at namespace level; so a certain namespace cannot exceed a certain amount of resources.

There are four definitions for resource requirements:

    CPU requests: Expressed as a fraction of a core that is reserved for the container. This can be expressed as a decimal number (0.1) or as a ‘milli’ value (100m). 0.1 and 100m are equivalent.
    Memory requests: Expresses as a number of bytes of memory to be reserved for the container. You can express this either as a actual number of bytes, as a E, P, T, G, M, K value or as a Ei, Pi, Ti, Gi, Mi, Ki value. Wondering about the difference between M and Mi? M is 10-based, M i is 2 based, meaning M is 1,000,000 and Mi is 1,048,576.
    CPU limits: The fraction of a core that can actually be scheduled. The milli-value here is multiplied by 100, and this value is the amount of ms that can be run per 100ms. (100m, would mean this container can be scheduled on the cpu for 10ms per 100ms)
    Memory limits: The maximum amount of memory that a container is allowed to use. If this value is exceeded, your container might be terminated. 

The kubernetes scheduler will work with the requests and limits to schedule pods across your cluster. The sum of the requests for containers in the pods running on a node will never exceed the total available resources on that node. This means that on a 2 core machine, a maximum of 4 pods with a request of 500m will be scheduled.

Please take into account that requests are reserved, and are not linked to the actual usage on your cluster. With incorrect requests, you could run a very underutilized cluster.
CPU settings

Let’s start playing around a little with CPU requirements first:

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

What you see in the yaml above, is that we create a stress container. We limit it to 1CPU, with a reservation of 0.5CPU, while we’re running stress with 2CPUs. Let’s look at how much CPU is being used:

kubectl create -f cpu-requests.yaml
pod/cpu-demo created
kubectl top pods
NAME       CPU(cores)   MEMORY(bytes)
cpu-demo   990m         1Mi

One thing I want you to notice here: when exceeding the limits, the pod kept running (i.e. wasn’t killed), and was just limited. This is different with memory usage, let’s have a look:
Memory settings

Let’s start with creating a pod that will try to get 150MB of memory, while being allowed 200MB. This should work, without any glitches:

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

We can create this and then watch the memory utilization:

kubectl create -f memory-low.yaml
kubectl top pods
NAME         CPU(cores)   MEMORY(bytes)
memory-low   40m          151Mi

And if we check a kubectl get pods, we’ll see zero restarts (as expected):

NAME         READY   STATUS    RESTARTS   AGE
memory-low   1/1     Running   0          3m56s

We can now edit the pod definition, and put a memory limit of 125Mi in place. This will create issues, and will cause kubernetes to kill and restart our pod.

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

We can create this pod, and we can quickly see that I’ll get killed and restarted:

kubectl create -f memory-high.yaml
kubectl get pods --watch

NAME          READY   STATUS      RESTARTS   AGE
memory-high   0/1     OOMKilled   0          6s
memory-low    1/1     Running     0          6m
memory-high   0/1     OOMKilled   1          7s
memory-high   0/1     CrashLoopBackOff   1          8s
memory-high   0/1     OOMKilled          2          21s

As you can see, the container is killed due to exceeding it’s memory requirements. Notice how this is different from our earlier CPU scenario?
Create and consume secrets

If you’ve followed along with our Configmaps example earlier, you’ll notice a lot of similarities to secrets. In essence, secrets are configmaps that are base64 obfuscated. There’s a couple of ways to create secrets, with the clue here being who does the base64 encoding:

    Using kubectl, referencing a file. Kubernetes will do the base64 encoding
    Using kubectl, using literal values. Kubernetes will do the base64 encoding
    Using a YAML file, you need to do the base64 encoding.

Afterwards you can consume secrets in your pods, we’ll check that after we created a few. Let’s have a look at all three mechanisms:
Using kubectl, referencing a file:

Let’s quickly create a file with the value of a secret:

echo 'love' >secretingredient.txt

We can now create a secret referencing that file:

kubectl create secret generic recipe --from-file=./secretingredient.txt 

We can then look at the secret, which will look like this:

kubectl get secret recipe -o yaml
apiVersion: v1
data:
  secretingredient.txt: bG92ZQo=
kind: Secret
metadata:
  creationTimestamp: "2019-07-17T04:17:49Z"
  name: recipe
  namespace: default
  resourceVersion: "3404565"
  selfLink: /api/v1/namespaces/default/secrets/recipe
  uid: d63c536a-a849-11e9-aedb-0aee3919cf84
type: Opaque

As you can see, the text itself is obfuscated. We can decode this using base64:

echo "bG92ZQo=" | base64 -d -
love

Using kubectl, using literal values

We can do the same using a literal-value, using the following kubectl:

kubectl create secret generic recipe --from-literal=mysecret=ilovekubernetes

We can then do the same as before, get the secret and decode the value:

kubectl get secret literalsecret -o yaml
apiVersion: v1
data:
  mysecret: aWxvdmVrdWJlcm5ldGVz
kind: Secret
metadata:
  creationTimestamp: "2019-07-17T04:22:12Z"
  name: literalsecret
  namespace: default
  resourceVersion: "3404979"
  selfLink: /api/v1/namespaces/default/secrets/literalsecret
  uid: 731c8496-a84a-11e9-aedb-0aee3919cf84
type: Opaque

echo "aWxvdmVrdWJlcm5ldGVz" | base64 -d -
Ilovekubernetes

Using a YAML file

Last way to create a secret is to create it via a yaml file. This yaml file expects your secret to be base64 encoded already, so let’s first base64 encode a string and create a secret:

echo "heavymetal" | base64 -
aGVhdnltZXRhbAo=

We can then create this secret, and do the same as before, trying to decode it:

apiVersion: v1
kind: Secret
metadata:
  name: music
type: Opaque
data:
  myfavorite: aGVhdnltZXRhbAo=

kubectl create -f secret.yaml
secret/music created
kubectl get secret music -o yaml
apiVersion: v1
data:
  myfavorite: aGVhdnltZXRhbAo=
kind: Secret
metadata:
  creationTimestamp: "2019-07-17T04:26:20Z"
  name: music
  namespace: default
  resourceVersion: "3405365"
  selfLink: /api/v1/namespaces/default/secrets/music
  uid: 070d7b14-a84b-11e9-aedb-0aee3919cf84
type: Opaque

echo "aGVhdnltZXRhbAo=" | base64 -d -
heavymetal

Let’s now have a look on how we can use these secrets in our pods:
Using secrets

Let’s have a look at how we can use the three secrets we created:

    Using a file: recipe
    Using a literal: literalsecret
    Using a YAML file: music

We’ll consume all three secrets in a single pod. We’ll consume the file secret twice, once as an environmental variable and once as a volume.

apiVersion: v1
kind: Pod
metadata:
  name: secretsconsumed
spec:
  containers:
    - name: basic
      image: busybox
      command:
        - sleep
        - "3600"
      env:
        - name: FROMFILE
          valueFrom:
            secretKeyRef:
              name: recipe
              key: secretingredient.txt
        - name: FROMLITERAL
          valueFrom:
            secretKeyRef:
              name: literalsecret
              key: mysecret
        - name: FROMYAML
          valueFrom:
            secretKeyRef:
              name: music
              key: myfavorite
      volumeMounts:
        - name: secret-volume
          mountPath: "/tmp/supersecret"
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: recipe

We can then create our pod, exec into it, and have a look at the environment variables that are set and read our file.

kubectl create -f secretsconsumed.yaml
kubectl exec -it secretsconsumed sh
/ # echo $FROMFILE
love
/ # echo $FROMLITERAL
ilovekubernetes
/ # echo $FROMYAML
heavymetal
/ # cat /tmp/supersecret/secretingredient.txt
love

Understand ServiceAccounts

A service account provides an identity for processes that run in a Pod. This allows processes in a pod to communicate to the api-server. Each pod by default gets assigned the default  service account.

Let’s go ahead and test this out (instructions partly from this stackoverflow post):

apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
spec:
  containers:
    - name: basic
      image: radial/busyboxplus:curl
      command:
        - sleep
        - "3600"

We’ll create this pod and connect to the kubernetes API:

kubectl create -f simplepod.yaml
kubectl exec -it simple-pod sh
KUBEHOST="aksworksho-akschallenge-d19ddd-c565b193.hcp.westus2.azmk8s.io" #change this to your cluster
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
      https://$KUBEHOST/api/v1/namespaces/default/pods/$HOSTNAME

The response to my curl (as you will see as well if your cluster is RBAC enabled) is:

{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods \"simple-pod\" is forbidden: User \"system:serviceaccount:default:default\" cannot get resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "name": "simple-pod",
    "kind": "pods"
  },
  "code": 403

Let’s try to solve this by giving our default service account access to the API-server. As I am lazy, I’ll show you something new, how you can create multiple objects from a single yaml file, by using the --- in between the objects.

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: default 
  apiGroup: ""
roleRef:
  kind: Role 
  name: pod-reader  
  apiGroup: ""

What we’ve done here, is create a Role called pod-reader, that allows its subjects to get, watch and list pods in the default namespace. Next we create a RoleBinding, that links out default ServiceAccount to that role.

Let’s create this, and see what is looks like in our pod. Btw. Notice how we didn’t kill our pod, we only made api-level changes.

kubectl create -f solution.yaml
kubectl exec -it simple-pod sh
KUBEHOST="aksworksho-akschallenge-d19ddd-c565b193.hcp.westus2.azmk8s.io" #change this to your cluster
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
      https://$KUBEHOST/api/v1/namespaces/default/pods/$HOSTNAME

And our response is the full json definition of our pod. #SUCCESS

We can also create our own service accounts, and then give them permissions to pods. Let’s try the same, with a new service account, the same role, a new rolebinding and a new pod.

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: thanos-read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: thanos 
  apiGroup: ""
roleRef:
  kind: Role 
  name: pod-reader  
  apiGroup: ""
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: thanos
  namespace: default
---
apiVersion: v1
kind: Pod
metadata:
  name: thanos-pod
spec:
  serviceAccountName: thanos
  containers:
    - name: basic
      image: radial/busyboxplus:curl
      command:
        - sleep
        - "3600"

Then, we’ll exec into our new pod, and try our curl again:

kubectl exec -it thanos-pod sh
KUBEHOST="aksworksho-akschallenge-d19ddd-c565b193.hcp.westus2.azmk8s.io" #change this to your cluster
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
      https://$KUBEHOST/api/v1/namespaces/default/pods/$HOSTNAME

And again we get a full JSON definition of our pod. #SUCCESS
Summary of configuration

In part 3 of my CKAD series we dove into configuration. We touched on configmaps and secrets (both pretty similar), securitycontext, resource constraints and service accounts.

Up to part 4?
certification ckad kubernetes
One thought to “CKAD series part 3: Configuration”

    Sushi By 7-11
    July 23, 2019 at 7:29 pm

    Hi! I’ve been following your site for a long time now and finally got the courage to go ahead and
    give you a shout out from Austin Texas! Just wanted to say keep up the
    good work!
    Log in to Reply

Leave a Reply

You must be logged in to post a comment.
Post navigation
CKAD series part 2: Core Concepts
CKAD series part 4: Multi-container pods
About the author
I'm Nills, a cloud architect focused on cloud automation. I share my technical stories on this blog, mainly on Azure, Kubernetes and cloud networking.

Search
Search for:
Recent Posts

    Creating Kubernetes clusters on Azure using cluster API
    Setting up Kubernetes on Azure using kubeadm
    Using public IPs from a public IP prefix in Azure Kubernetes Service
    Automatically turning on diagnostic settings using Azure Policy
    GitHub SSO using password-protected SSH keys

Categories

    Azure (57)
    business (1)
    certification (2)
    CKAD series (9)
    Data Science (6)
    DevOps (26)
    Kubernetes (28)
    Management (23)
    Networking (13)
    Open Source (34)
    Personal Development (8)
    Security (7)
    Software Development (10)
    Uncategorized (28)
    Windows (11)
    Wordpress (2)

sparkling Theme by Colorlib Powered by WordPress	
