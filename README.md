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
This command generates the yaml file to create the `fdp-1234-dev-01` namespace .
Pods created on this namespace will be scheduled only on nodes with `role=apps`

## RBAC


## Quota policies

### Compute quotas

### Storage quotas   

### Object quotas

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
