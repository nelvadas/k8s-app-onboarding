# Onboarding applications on Kubernetes

This document describes a practical guide to onboard applications on a kubernetes based Platform.
it explores a way for cluster maintenance team to  automatically create and initialize  application namespaces with custom policies before handling over the namespace to an application team.  
This process relies on [kustomize](https://github.com/kubernetes-sigs/kustomize) and sed commmand to produce  the final YAML files for your namespaces.



1. Namespace Constraints
2. RBAC: RoleBinding
3. ResourceQuota
4. Network Policies
5. Ingress Rules Templates
6. Pod Security Policies


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
```

Run the following command to create the namespace

```
$ kustomize build . | sed -e 's/$(NAMESPACE)/fdp-1234-dev-01/g' | kubectl apply -f  -
namespace/fdp-1234-dev-01 created
```



## RBAC
1. Define a rolebinding to assign admin role on the namespace to the application Owner
2. Define a rolebinding to assign view role on the namespace to the application LDAP Group
3. Define a rolebinding to assign the edit role to the cicd service account on this namespace


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
For services, if an ingress controller is setup, the cluster should restrict `loadbalancer` and `nodeports` Services

### Limit Ranges
ResourceQuota works at a namespace level, you can control resource request/limits at Pod/Container Level using `LimitRange`
LimitRange require a deep knowledge of the application details, the recommendation here will be to leave this step to the hand of the application team
unless you have a very specific and generic policy to apply at `Pod/Container` level in all namespaces.

## Network Policies

Nicolas Kabar wrote a nice blog post on network policies  [ check the post here ](https://medium.com/@nicolakabar/7-practical-steps-to-onboard-your-teams-into-docker-enterprise-3-0-5d77548de9c0)
The most important thing to remember here is that you should have three rules
1. Default deny Rule that deny all communications by default
2. A rule to Allow intra namespace communications : Pod of the same namespace should be able to communicate  
3. Ingress to App communications: Allow flows from ingress controller namespace to applications namespaces



## Ingress Rules
Inject a template Ingress resource file in the namespace for reference.
Application will rely on this Ingress resource to build their own.

```
$ kubectl get ingress
NAME                   HOSTS                                ADDRESS   PORTS     AGE
example-ingress-rule   example.apps.dev01.dockernetes.org             80, 443   28s
```

## Pod Securities policies  
PSP are cluster wide resource and they sould be initalized before creating namespaces.
