---
title: Secrets
summary: 
authors:
    - Kathy Barabash
date: 2022-01-13
---

# Secrets

k8s secrets are esseantially obfuscated `configmaps` and can be defined in two ways with respect to where the obfuscation is computed:

1. With kubectl `create secret` command, referencing either a file or literal values; in this case, k8s computes the base64 encoding
1. From resource YAML file; in this case base64 encoding must be computed for k8s

## Creating secrets

### From text file

```
echo 'love' >secretingredient.txt
kubectl create secret generic recipe --from-file=./secretingredient.txt 

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
```
As you can see, the text itself is obfuscated. We can decode this using base64:
```
echo "bG92ZQo=" | base64 -d -
love
```

### From literals

```
kubectl create secret generic recipe --from-literal=mysecret=ilovekubernetes

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
```

### From YAML 

```
echo "heavymetal" | base64 -
aGVhdnltZXRhbAo=
```

We can then specify `secret` resource in `secret.yaml` file and create it:
```
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
```

## Using secrets

The following pod specificiation uses all three secrets defined above - `recipe`, `literalsecret`, and `music`. All three secrets are used to define environmant variable. The `recipe` secret is also used to define volume name: 

```
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
```
We can then deploy our pod, exec into it, and have a look at the environment variables that are set and access the storage volume as file:
```
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
```
---


