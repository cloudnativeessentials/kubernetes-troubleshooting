# Kubernetes Troubleshooting

Kubernetes troubleshooting includes examples of Kubernetes resources in error and how to troubleshoot on Kubernetes. Each section should have a failure condition, an investigation, and a remediation. This is meant to be followed using an OpenShift cluster.

##  Table of Contents

- [Prerequisites](#1-prerequisites)
- [Image Pull Errors](#2-image-pull-errors)
- [Pod Stuck Creating](#3-pod-stuck-creating)
- [Using logs to debug](#4-debugging-with-pod-logs)
- [Scheduling](#5-scheduling)
- [Storage issues](#6-storage)
- [Database Problems](#7-database-issues)
- [Cleanup](#8-cleanup)


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
View the instance of the pod that was created and see which error/cause we have in this case.

Typical causes of this issue:
- Is the container image correct in the manifest?
- Is the container image available from the registry?
- Does the cluster have network access to the registry?
- If a private image registry is used, does the cluster have access to the private registry (with the proper Secret)?

In this case we have an Access Denied, error. One important thing to note, is that some registries will provide this response when an image does not exist. So it still makes sense to verify the defined image first.

We have the following in the manifest:
- `image: nginxoops:1.27.0`

So it seems like there is an incorrect image definition in the manifest. Most likely this is an issue created in the CICD pipelines and either the wrong image name is used altogether, or a tag was updated but the new image is not available.

#### Remediate

In our case, the image itself is just incorrect. We can fix this by editing the yaml manifest, and changing the image definition to:
- `image: nginx:1.27.0`

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

Use the following command to deploy a pod that will generate a pod that is stuck with a `CreatingContainer` status:

```shell
oc apply -f https://raw.githubusercontent.com/cloudnativeessentials/kubernetes-troubleshooting/main/pod-configmap-notexist.yaml
```

#### Inspect

#### Remediate

#### Cleanup

### 4. Debugging With Pod Logs

### 5. Scheduling

### 6. Storage

### 7. Database issues

### 8. Cleanup
