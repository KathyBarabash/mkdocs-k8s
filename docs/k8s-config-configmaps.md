---
title: Config Maps
summary: 
authors:
    - Kathy Barabash
date: 2022-01-13
---

# Configmaps

[ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) are used to store non secret configuration data in the k8s server and to deliver it to the containers, while separating the configuration data from other resource definitions. ConfigMaps can deliver configuration that can be encoded as key-value pairs, e.g. command-line arguments, environment variables, ports, etc., and can be defined in several different ways: from literal, from .config file, with yaml, and using kustomization.yaml. Below are examples of using different ways to deliver environment variables `YEAR` with  value `2022` and `MONTH` with value `Jan` to a pod. We start with a single variable and then proceed extending to a number of variables.

## __From literal__
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

## __From .config file__

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

## __From YAML File__

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
---

