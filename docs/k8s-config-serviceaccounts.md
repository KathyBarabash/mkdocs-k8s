---
title: Service Accounts
summary: 
authors:
    - Kathy Barabash
date: 2022-01-13
---

# ServiceAccounts

k8s [service account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) provides an identity for processes that run in a Pod. This allows processes in a pod to communicate to the api-server. Each pod by default gets assigned the default service account.

Let’s go ahead and test this out (instructions partly from this stackoverflow [post](https://stackoverflow.com/questions/30690186/how-do-i-access-the-kubernetes-api-from-within-a-pod-container)):

```
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
```
We’ll create this pod and connect to the kubernetes API:
```
kubectl create -f simplepod.yaml
kubectl exec -it simple-pod sh
KUBEHOST="aksworksho-akschallenge-d19ddd-c565b193.hcp.westus2.azmk8s.io" #change this to your cluster
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
      https://$KUBEHOST/api/v1/namespaces/default/pods/$HOSTNAME
```
The response to my curl (as you will see as well if your cluster is RBAC enabled) is:
```
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
```
Let’s try to solve this by [giving our default service account access to the API-server](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). As I am lazy, I’ll show you something new, how you can create multiple objects from a single yaml file, by using the `---` in between the objects.
```
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
```
What we’ve done here, is create a Role called pod-reader, that allows its subjects to get, watch and list pods in the default namespace. Next we create a RoleBinding, that links out default ServiceAccount to that role.
Let’s create this, and see what is looks like in our pod. Btw. Notice how we didn’t kill our pod, we only made api-level changes.
```
kubectl create -f solution.yaml
kubectl exec -it simple-pod sh
KUBEHOST="aksworksho-akschallenge-d19ddd-c565b193.hcp.westus2.azmk8s.io" #change this to your cluster
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
      https://$KUBEHOST/api/v1/namespaces/default/pods/$HOSTNAME
```
And our response is the full json definition of our pod. #SUCCESS

We can also create our own service accounts, and then give them permissions to pods. Let’s try the same, with a new service account, the same role, a new rolebinding and a new pod.
```
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
```
Then, we’ll exec into our new pod, and try our curl again:
```
kubectl exec -it thanos-pod sh
KUBEHOST="aksworksho-akschallenge-d19ddd-c565b193.hcp.westus2.azmk8s.io" #change this to your cluster
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
      https://$KUBEHOST/api/v1/namespaces/default/pods/$HOSTNAME

And again we get a full JSON definition of our pod. #SUCCESS
```

---

