# Kubelet Pod API

Authors: Dan Gillespie (dan.gillespie@coreos.com)
## Introduction

The function of the kubelet is to manage Pods and their containers, volumes, etc. While it primarily acts as a client of the API server, the kubelet can also act as a standalone manager of Pods. This property gives Kubernetes the ability to bootstrap itself by running core Kubernetes components (etcd, apiserver, scheduler, etc..) in Pods managed by the kubelet. An examination of the current solutions for standalone Kubelet operation reveals that the tradeoffs involved make them poorly suited for bootstrapping. We propose the addition of an API endpoint intended for bootstrapping which would allow the creation and deletion of arbitrary Pods. When an API server becomes available, these Pods are persisted to it and placed under the server's control.

## Existing Solutions

Methods already exist to run Pods on a standalone Kubelet however they each have their own downsides which make them unideal for the bootstrap use case.

### Static Pods

The existing solution integrated in the Kubelet for running standalone results in the creation of “static” Pods. Static pods can be created in two ways, both require the Pod source to be specified at Kubelet start:
* **File based:** The Kubelet periodically checks a path (either file or directory) specified with `--pod-manifest-path` for Pod manifests. This means that modification require filesystem write access.
* **HTTP based**: The Kubelet periodically polls an URL specified with `--manifest-url` for Pod manifests.

The major downside of using static Pods is that they are a special class of Pod. While visible in the API server (as “mirror” Pods), they are read-only and cannot be controlled by it. This makes them a poor candidate for bootstrapping self-hosting, where the goal is for Kubernetes to manage itself.

### API Server Bootstrap

The kubelet can be bootstrapped by using it as a client and pointing it at the API server of a temporary local control plane which has the desired control plane Pods (etcd, apiserver, scheduler, controller-manager) created on it. After the pods have started, the kubelet transitions to use the Pod based control plane and the temporary one dies. This functionality is demonstrated in the bootkube project.

The major downside of this method is that the cluster version is limited by the version of the initial components (API server, controller manager, scheduler) and would require migrations to be able to run arbitrary versions. This would also require a release of bootkube be put out for every Kubernetes release, which is an additional maintenance burden. Additionally, bootstrapping using a temporary control plane adds non-trivial complexity which increases the chance of failure in cluster startup.

## Implementation

A new Pod source for the kubelet would be created which would allow for the creation and deletion of Pods via a server running on the kubelet in the absence of an API server. When an API server becomes available, the Pods would attempt to be created on it.

### Pod API
#### Creation

Pods that are POSTed to a newly created HTTP endpoint are sent as ADD events to kubelet Pod storage. This should result in the creation of Pods, however no guarantees are provided as they may get rejected in the sync loop.

#### Deletion

Requests sent with the DELETE HTTP verb for a Pods UID, will trigger a DELETE event to be sent to kubelet Pod storage to attempt to delete the Pod.

#### Security

Since this endpoint would have the ability to run arbitrary code with root access and push Pods to an API server, measures must be taken to protect it from unauthorized access. A feature flag will be required to enable the API. The use of access protected domain sockets and piggybacking on other Kubelet authentication efforts has been raised. *For this reason until it’s protected, this API should be exposed separately than the current API.*
#### Naming

In order to reduce the risk of name collisions between bootstrapped Pods and other Pods in a cluster, the names of created Pod will suffixed with the Nodes name. The name would be in the form of `<PodName>-<NodeName>`, which is the same naming scheme used by mirror Pods.

#### Source Tagging

Before Pods are passed into the sync loop, they are given a source annotation marking them as being created by the bootstrap API. This takes the form of `kubernetes.io/config.source: kubelet-api`. This fits the scheme used for existing Pod sources.

### Persisting to API Server

After Pods have been created using the new API, a new loop is added to the kubelet which perpetually attempts to POST any Pod in storage with source `kubelet-api`. Any errors should be logged but should otherwise be unhandled.

### Kubelet Pod Sync Loop

The kubelet synchronizes modification of Pods on a Node through a run loop which makes changes to Pods in response to events from Pod sources. Implementing the bootstrap API requires a few changes to the Pod sync loop.

#### Add

The add handler should be changed to only create static Pods only when the source (kubernetes.io/config.source annotation) is either `http` or `file`. Currently, Pods with sources other than `apiserver` are made static.

#### Delete

When an API server connection exists after a Pod has been created with the kubelet Pod API, Pod deletion events will be sent for the Pod because it does not yet exist in the API server. In order to keep the Pod existing long enough to be created on the API server, the delete handler should be modified for any Pod with a `kubelet` source annotation to ignore the event and log the occurrence.

#### Update

The update handler shouldn’t require any changes but it’s worth noting that it will replace Pods with a bootstrap server source with ones from the API server. Currently, there is no way to define client-side a UID to the API server. This means that Pods created on the API server will have different UIDs than Pods running locally on the kubelet. *This has the consequence that any Pod created with the Kubelet Pod API will be restarted when connected to an API server.*

**Alternatives to Restarts**

Requiring that Pods be restarted in order to use the Kubelet Pod API is not ideal for the bootstrapping case since it introduces instability at a fragile time in system startup. Two options have been identified to avoid this:
Similar to mirror Pods, create a representational Pod on the API server and maintain a mapping with the locally running Pod. Mirror Pods were written to be read-only so it would be difficult to reuse their implementation.
Modify the API server to allow clients to specify UIDs

### Bootstrap Workflow

1. Start kubelet on Node with flags to enable kubelet Pod API and its API server set to a free port on localhost
1. Use Kubelet Pod API to create Pods for the apiserver, scheduler, and controller-manager. The apiserver should be configured to listen on the free port and the other components should point at the apiserver.
1. Pods start
1. kubelet registers itself with API server
1. kubelet persists bootstrap Pods
1. Create higher level objects (ie. ReplicaSet) with selectors matching the bootstrapped Pods labels to ensure new Pods to replace the existing ones if they are deleted. The objects will adopt the bootstrapped Pods.
1. Use API server Pod as a normal API server and bring up other components such as DNS.
