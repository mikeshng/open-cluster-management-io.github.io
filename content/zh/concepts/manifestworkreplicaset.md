---
title: 资源下发组
weight: 7
---

<!-- spellchecker-disable -->

{{< toc >}}

<!-- spellchecker-enable -->

## What is `ManifestWorkReplicaSet`

`ManifestWorkReplicaSet` is an aggregator API that uses [Manifestwork](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/blob/main/content/en/concepts/manifestwork.md) and [Placement](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/blob/main/content/en/concepts/placement.md) to create manifestwork for the placement-selected clusters. 

View an example of a `ManifestWorkReplicaSet` to deploy a CronJob and Namespace for a group of clusters selected by a placement.

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: mwrset-cronjob
  namespace: ocm-ns
spec:
  placementRefs:
    - name: placement-byname # Name of a created Placement
  manifestWorkTemplate:
    deleteOption:
      propagationPolicy: SelectivelyOrphan
      selectivelyOrphans:
        orphaningRules:
          - group: ''
            name: ocm-ns
            namespace: ''
            resource: Namespace
    manifestConfigs:
      - feedbackRules:
          - jsonPaths:
              - name: lastScheduleTime
                path: .status.lastScheduleTime
              - name: lastSuccessfulTime
                path: .status.lastSuccessfulTime
            type: JSONPaths
        resourceIdentifier:
          group: batch
          name: sync-cronjob
          namespace: ocm-ns
          resource: cronjobs
    workload:
      manifests:
        - kind: Namespace
          apiVersion: v1
          metadata:
            name: ocm-ns
        - kind: CronJob
          apiVersion: batch/v1
          metadata:
            name: sync-cronjob
            namespace: ocm-ns
          spec:
            schedule: '* * * * *'
            concurrencyPolicy: Allow
            suspend: false
            jobTemplate:
              spec:
                backoffLimit: 2
                template:
                  spec:
                    containers:
                      - name: hello
                        image: 'quay.io/prometheus/busybox:latest'
                        args:
                          - /bin/sh
                          - '-c'
                          - date; echo Hello from the Kubernetes cluster
```
The placement reference must be in the same namespace as the manifestWorkReplicaSet. 

## Status tracking

Let's assume the placement reference used in the previous example **placement-byname** has 10 clusters selected. The ManifestWorkReplicaSet monitors the Placement-selected clusters and creates/deletes ManifestWork for the placement clusters. The ManifestWorkReplicaSet tracks the status conditions of the created ManifestWorks and reports a summary for all ManifestWork statuses in the ManifestWorkReplicaSet status.

The ManifestWorkReplicaSet has two status conditions; 
1. **PlacementVerified** to verify the placementRefs (not exist or empty cluster selection). 
1. **ManifestWorkApplied** to verify all the created ManifestWork status conditions (applied, progressing, degraded or available).

Here is an example.

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: mwrset-cronjob
  namespace: ocm-ns
spec:
  placementRefs:
    - name: placement-byname # Name of a created Placement
  manifestWorkTemplate:
     ...
status:
 conditions:
   - lastTransitionTime: '2023-04-27T02:30:54Z'
     message: ''
     reason: AsExpected
     status: 'True'
     type: PlacementVerified
   - lastTransitionTime: '2023-04-27T02:30:54Z'
     message: ''
     reason: AsExpected
     status: 'True'
     type: ManifestworkApplied
 summary:
   applied: 10
   available: 10
   progressing: 0
   degraded: 0
   total: 10
```
## Release and Enable Feature

ManifestWorkReplicaSet is in alpha release and it is not enabled by default. In order to enable the ManifestWorkReplicaSet feature, it has to be enabled in the cluster-manager instance in the hub. Use the following command to edit the cluster-manager CR (custom resource) in the hub cluster.

```shell
$ oc edit ClusterManager cluster-manager
```
Add the workConfiguration field to the cluster-manager CR as below and save.

```yaml
kind: ClusterManager
metadata:
  name: cluster-manager
spec:
   ...
  workConfiguration:
    featureGates:
    - feature: ManifestWorkReplicaSet
      mode: Enable
```
In order to assure the ManifestWorkReplicaSet has been enabled successfully check the cluster-manager using the command below

```shell
$ oc get ClusterManager cluster-manager -o yml
```
You should find under the status->generation the cluster-manager-work-controller deployment has been added as below

```yaml
kind: ClusterManager
metadata:
  name: cluster-manager
spec:
   ...
status:
   ...
  generations:
    ... 
  - group: apps
    lastGeneration: 2
    name: cluster-manager-work-webhook
    namespace: open-cluster-management-hub
    resource: deployments
    version: v1
  - group: apps
    lastGeneration: 1
    name: cluster-manager-work-controller
    namespace: open-cluster-management-hub
    resource: deployments
    version: v1
```