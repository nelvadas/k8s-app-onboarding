# Onboarding applications on Kubernetes

This document describes a practical guide to onboard applications on a kubernetes based Platform.
it explores a way for cluster maintenance team to  automatically create and initialize  application namespaces with custom policies before handling over the namespace to an application team.  
This process relies on [kustomize](https://github.com/kubernetes-sigs/kustomize) and sed commmand to procude ready to go Yaml files for your namespaces.



1. Namespace Setup
2. RBAC
3. ResourceQuota
4. Network Policies
5. Pod Security Policies
6. Ingress Rules Templates


## Namespace Constraints
Before releasing a namespace to the application team , make sure the different labels and annotations are defined.
e.g Annotations to select nodes on which the current namespace's pods can be deployed

```
$ git clone https://github.com/nelvadas/k8s-app-onboarding.git
$ cd  k8s-app-onboarding
$ kustomize build . | sed -e 's/$(NAMESPACE)/fdp-1234-dev-01/g'
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/node-selector: role=apps
  labels:
    applicationn: fdp
    environment: dev
    organization: IT
    size: S
  name: fdp-1234-dev-01
```
This command generates the YAML file to create the `fdp-1234-dev-01` namespace .
Pods created on this namespace will be scheduled only on nodes with `role=apps`

The namespace template is generated from a kustomize
```
$ cat kustomization.yaml
commonLabels:
  environment: dev
  organization: IT
  applicationn: fdp
  size: S
commonAnnotations:
  scheduler.alpha.kubernetes.io/node-selector: "role=apps"
resources:
- namespace.yaml

 define by the can be created using the following command
```
$ kustomize build . | sed -e 's/$(NAMESPACE)/fdp-1234-dev-01/g' | kubectl apply -f  -
namespace/fdp-1234-dev-01 created
```


## RBAC


## Quota policies
In a multitenant cluster, it is very important to constraint application to not overconsummes resources,
we create three different quota to limit compute/storage/object in the created namespace

```
$ kustomize build base | sed -e 's/$(NAMESPACE)/fdp-1234-dev-01/g' | kubectl apply -f  -
resourcequota/compute-quota created
resourcequota/object-quota created
resourcequota/storage-quota created
```

### Compute quotas

Limits CPU and Memory `request` and `limits` for all pods running in the namespace
```
a$ cat base/quota/compute-quota.yaml
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
```


### Storage quotas
Control how much storage can be requested in a namespace, control can be done on a specific storageclass request, ephermeral storage used by containers
or the total number of PVC created
```
$ cat base/quota/storage-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
 name: storage-quota
spec:
 hard:
   requests.storage: "10Gi"
   requests.ephemeral-storage: "2Gi"

   limits.ephemeral-storage: "3Gi"

   elastifile.storageclass.storage.k8s.io/requests.storage: 4Gi
   elastifile.storageclass.storage.k8s.io/persistentvolumeclaims: 3
   pd-standard.storageclass.storage.k8s.io/requests.storage: 6Gi
   ssd-regional.storageclass.storage.k8s.io/persistentvolumeclaims: 7
   ```


### Object quotas
Limits the number of Object ( Services, Pods, PVC ..) we can have  in a namespace

```
$ cat base/quota/object-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
spec:
  hard:
    pods: 20
    replicationcontrollers: 10
    services: 10
    services.loadbalancers: 0
    services.nodeports: 0
    configmaps: 10
    resourcequotas: 3
    persistentvolumeclaims: 10
    ```

### Limit Ranges
ResourceQuota works at a namespace level, you can control resource request/limits at Pod/Container Level using `LimitRange`
LimitRange require a deep knowledge of the application details, the recommendation here will be to leave this step to the hand of the application team
unless you have a very specific and generic policy to apply at `Pod/ContainerWh level in all namespaces.

## Network Policies

### Default deny all Rule  

### Allow intra namespace communications  

### Allow flows from ingress namepasces to application namespaces

## Pod Securities policies  


## Ingress Rules
