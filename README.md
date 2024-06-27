# Kubernetes Troubleshooting

Kubernetes troubleshooting includes examples of Kubernetes resources in error and how to troubleshoot on Kubernetes. Each section should have a failure condition, an investigation, and a remediation. This is meant to be followed using an OpenShift cluster.

##  Table of Contents

- [Prerequisites](#1-prerequisites)
- [Image Pull Errors](#2-image-pull-errors)
- [Pod Stuck Creating](#3-pod-stuck-creating)
- [Using logs to debug an app](#4-app-debugging-with-pod-logs)
- [Using logs to debug a database](#4-app-debugging-with-pod-logs)
- [Scheduling](#6-scheduling-issues)
- [Storage issues](#7-storage-issues)


### 1. Prerequisites

An OpenShift developer Sandbox cluster. You may need to register for a developer account:
- [Developer Sandbox](https://developers.redhat.com/developer-sandbox)

### 2. Image Pull Errors

#### Overview
When deploying a workload, there are occasions when the image is not accessible to the container runtime. You will see the status of the pod, as well as the events inside of the OpenShift console. This usually manifests in one of two statuses in the pod instance:

- `ErrImagePull`
- `ImagePullBackOff`

#### Deploy 
Use the following command to deploy a pod that will generate an Image Pull issue:

```shell
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-badimage.yaml
```

#### Inspect 
View the instance of the pod that was created and see which error/cause we have in this case. First check the pod status and conditions. If nothing obvious, check the events.

Typical causes of this issue:
- Is the container image correct in the manifest?
- Is the container image available from the registry?
- Does the cluster have network access to the registry?
- If a private image registry is used, does the cluster have access to the private registry (with the proper Secret)?

In this case we have an Access Denied, error. One important thing to note, is that some registries will provide this response when an image does not exist. So it still makes sense to verify the defined image first.

We have the following in the manifest:
- `image: quay.io/nginx/nginx-unprivilegedoops:stable-alpine`

So it seems like there is an incorrect image definition in the manifest. Most likely this is an issue created in the CICD pipelines and either the wrong image name is used altogether, or a tag was updated but the new image is not available.

#### Remediate

In our case, the image itself is just incorrect. We can fix this by editing the yaml manifest, and changing the image definition to:
- `image: quay.io/nginx/nginx-unprivileged:stable-alpine`

Note that in most environments, this will need to be done via CICD, or the next deployment will overwrite this manifest.

#### Cleanup
Clean up the resources with the following command:
```shell
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-badimage.yaml
```

### 3. Pod Stuck Creating

#### Overview
A Pod may not get to a running state and be stalled at the `CreatingContainer` status. 
`ContainerCreating` is different from the `Pending` status. 
The `Pending` status usually means the pod has yet to be scheduled to a node while with a `CreatingContainer` state, the pod has been scheduled to a node and the kubelet is in the process of creating the container(s) for the pod.
When a pod is stuck with a `CreatingContainer` state, the container may need something but is not available.

#### Deploy
Use the following command to deploy a pod that is stuck with a `CreatingContainer` status:

```shell
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-configmap-notexist.yaml
```

#### Inspect
View the instance of the pod that was created and see which error/cause we have in this case. First check the pod status and conditions. If nothing obvious, check the events.

Typical causes of this issue are missing dependent resources, such as secrets, volumes, or configmaps.

#### Remediate

In this instance we have the following error message: `configmap "mmconfigymap" not found`. Remembering that configmaps are namespaced objects, let us validate whether there is a configmap with a different name that exists for this application, or whether the configmap does not exist at all. In our case, the configmap does not exist. So if we create an empty configmap, it will mount successfully and the pod will run.

#### Cleanup
Clean up the resources with the following command:
```shell
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-configmap-notexist.yaml
```
### 4. App Debugging With Pod Logs

#### Overview
Take an example application where we have a frontend and backend service. We will use wordpress here, as this represents this architecture. We will often see pods that are scheduled, begin running, but never successfully start. If we have a pod that ends up in `CrashLoopBackOff` or `Error` state, this usually indicates an application issue, though it can also be due to an improperly configured readiness or liveliness probe.

#### Deploy
Use the following command to deploy the wordpress application:

```shell
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/wordpress-basic.yaml
```

#### Inspect
<p>View the deployed wordpress application in the console. You will see that the wordpress pod is continuously restarting. Viewing the status, conditions, and events do not provide any useful information. The next step here is to then view the application logs themselves to see why the pod is restarting/crashing.

If we view the pod logs in the console, we can see that there is a `Permission denied` error when trying to bind to a privileged port. In OpenShift, non root containers are enforced, so similarly to a non root process, it cannot bind to a low port.

#### Remediate
The full details of eleveated capabilities is beyond the scope of this course, however there is a capability `CAP_NET_BIND_SERVICE` that we can attach to our pod, that should allow us to bind to privileged ports. Let us try and add the following block to our Deployment manifest for the crashing service:

```shell        
        securityContext:
          capabilities:
            add: ["CAP_NET_BIND_SERVICE"]
```

Alternatively, you can do this via cli with a pre-updated manifest:
```shell
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/wordpress-basic-add-linuxcapability.yaml
```
You can see that this does not seem to fix the issue, and now we have a new message, that we are violating a security policy, and that we are not allowed to arbitrarily add capabilities to our containers. This another layer of security involved. Instead we can add a sysctl entry that tells the base OS that this is not a privileged port by using the following:

```shell
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/wordpress-basic-add-sysctls.yaml
```

#### Cleanup
Clean up the resources with the following command:
```shell
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/wordpress-basic-add-sysctls.yaml
```

### 5. Database debugging with pod logs

#### Overview
Now let's deploy wordpress in a similar way. In this case we will include the frontend and backend in a single pod. This will show us how one container of a pod having issues starting, can fail the whole pod.

#### Deploy
Use the following command to deploy a multi container wordpress pod:
```shell
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/wordpress-badmysql.yaml
```

#### Inspect
Viewing the pod details, you can see that it is restarting, and in a `CrashLoopBackoff`. Since there is no indication of the cause in these details, we will need to dig into the container logs. Looking in the container logs, we can see that we are missing a required paramter for the database to properly start.

#### Remediate
Use the following commands to delete the bad pod definition, and re-deploy with the proper variable set:
```shell
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "mypassword"
```
```shell
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/wordpress-badmysql.yaml
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/wordpress-badmysql-fix.yaml
```

#### Cleanup
Clean up the resources with the following command:
```shell
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/wordpress-badmysql-fix.yaml
```

### 6. Scheduling issues

#### Overview
The kubernetes scheduler will find a node in the cluster for each pod that needs to run.  Nodes are filtered out if they do not meet the needs of the pod. The two most common being:

- `PodFitsResources` - Is there a node where the CPU and Memory requirements set in the spec are met.
- `PodToleratesNodeTaints` - Certain nodes are set with taints to prevent resources from being scheduled there. A pod must specifically tolerate this to allow scheduling. Most frequently used to prevent workloads from being scheduled to control plane nodes.

#### Deploy
Use the following command to deploy some example resource (you might have the highmem manifest blocked due to quota):

```shell
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-pvc-notexist.yaml
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-highmem.yaml
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-memstresshigh.yaml
```

#### Inspect
View the pods in the console, and you will see that the Status is stuck in `Pending`. Clicking on the `Pending` link or viewing the conditions will show the reason why it is in this state.

#### Remediate
For the `nginx` pod, we can see that we have the following message about a missing pvc: `persistentvolumeclaim "mypvc" not found`. We can fix this by creating the pvc with this name, in the same namespace.<br>

For the `nginx-highmem` pod, we can see that we do not have sufficient memory available to schedule the pod based on its Request of `128Gi`. Tuning pod requests is an important task for operations and development to work hand in hand on, as if we create requests that are higher than we need, we will reserve space that could be used otherwise and prevent efficient scheduling.<br>

For the `stress` pod, we can see that it is continuously getting OOM killed. This is because the configured memory limit is being reached. If this is normal behavior, please update the requests and limits for the application.

#### Cleanup
Clean up the resources with the following command:
```shell
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-pvc-notexist.yaml
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-memstresshigh.yaml
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-highmem.yaml
```

### 7. Storage issues

#### Overview
When using persistent storage, you can come across a couple of different scenarios. In many cases, this is either going to be an underlying storage problem, or incorrect configurations that need to be rectified. Let's discuss a couple of situations that might occur.

#### Deploy
Use the following command to deploy some example resource:

```shell
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-sc-notexist.yaml
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-pvc-notexist.yaml
```
#### Inspect
You can now see tha the three new pods have started and are stuck in Pending state. If we view the conditions, we can see the different error messages for each workload.

#### Remediate
- For `nginx-scnotexist` we can see that there is an unbound PVC. What this means is that a `PersistentVolumeClaim` exists, but cannot be fulfilled. The PVC itself shows `storageclass.storage.k8s.io "veryfaststorage" not found`. This means that an incorrect StorageClass was configured in the application. The fix for this is to use an appropriate StorageClass per guidance from the Container Platform team, and ensure that the application owners update their deployments. This is not something that is really resolved live, and also usually only a problem on initial deployment of a new application.
- For `nginx-nopvc` we see that similar to a previous exercise, that no PVC is defined with the application, even though it tries to mount a persistent volume. The resolution for this is to either remove the persistent volume in the application, or specify a PVC with an appropriate storageclass. This is also generally a first deploy issue only.
#### Cleanup
Clean up the resources with the following command:
```shell
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-pvc-notexist.yaml
oc delete -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-sc-notexist.yaml
```
