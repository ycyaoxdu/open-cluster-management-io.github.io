---
title: ManifestWork
weight: 2
---

<!-- spellchecker-disable -->

{{< toc >}}

<!-- spellchecker-enable -->

## What is `ManifestWork`

`ManifestWork` is used to define a group of Kubernetes resources on the hub to be applied to the managed cluster. In the open-cluster-management project, a `ManifestWork` resource must be created in the cluster namespace. A work agent implemented in [work](https://github.com/open-cluster-management-io/work) project is run on the managed cluster and monitors the `ManifestWork` resource in the cluster namespace on the hub cluster.

An example of `ManifestWork` to deploy a deployment to the managed cluster is shown in the following example.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: hello-work-demo
spec:
  workload:
    manifests:
      - apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: hello
          namespace: default
        spec:
          selector:
            matchLabels:
              app: hello
          template:
            metadata:
              labels:
                app: hello
            spec:
              containers:
                - name: hello
                  image: quay.io/asmacdo/busybox
                  command:
                    ["sh", "-c", 'echo "Hello, Kubernetes!" && sleep 3600']
```

## Status tracking

Work agent will track all the resources defined in `ManifestWork` and update its status. There are two types of status in manifestwork. The `resourceStatus` tracks the status of each manifest in the `ManifestWork` and `conditions` reflects the overall status of the `ManifestWork`. Work agent currently checks whether a resource is `Available`, meaning the resource exists on the managed cluster, and `Applied` means the resource defined in `ManifestWork` has been applied to the managed cluster.

Here is an example.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata: ... 
spec: ...
status:
  conditions:
    - lastTransitionTime: "2021-06-15T02:26:02Z"
      message: Apply manifest work complete
      reason: AppliedManifestWorkComplete
      status: "True"
      type: Applied
    - lastTransitionTime: "2021-06-15T02:26:02Z"
      message: All resources are available
      reason: ResourcesAvailable
      status: "True"
      type: Available
  resourceStatus:
    manifests:
      - conditions:
          - lastTransitionTime: "2021-06-15T02:26:02Z"
            message: Apply manifest complete
            reason: AppliedManifestComplete
            status: "True"
            type: Applied
          - lastTransitionTime: "2021-06-15T02:26:02Z"
            message: Resource is available
            reason: ResourceAvailable
            status: "True"
            type: Available
        resourceMeta:
          group: apps
          kind: Deployment
          name: hello
          namespace: default
          ordinal: 0
          resource: deployments
          version: v1
```

### Fine-grained field values tracking

Optionally, we can let the work agent aggregate and report certain fields from 
the distributed resources to the hub clusters by setting `FeedbackRule` for 
the `ManifestWork`:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata: ...
spec:
  workload: ...
  manifestConfigs:
    - resourceIdentifier:
        group: apps
        resource: deployments
        namespace: default
        name: hello
      feedbackRules:
        - type: WellKnownStatus
        - type: JSONPaths
          jsonPaths:
            - name: isAvailable
              path: '.status.conditions[?(@.type=="Available")].status'
```

The feedback rules prescribe the work agent to periodically get the latest 
states of the resources, and scrape merely those expected fields from them,
which is helpful for trimming the payload size of the status. Note that the
collected feedback values on the `ManifestWork` will not be updated unless
the latest value is changed/different from the previous recorded value. 
Currently, it supports two kinds of `FeedbackRule`:

- `WellKnownStatus`: Using the pre-built template of feedback values for those
  well-known kubernetes resources.
- `JSONPaths`: A valid [Kubernetes JSON-Path](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
  that selects a scalar field from the resource. Currently supported types are 
  **Integer**, **String** and **Boolean**.

The default feedback value scraping interval is 30 second, and we can override 
it by setting `--status-sync-interval` on your work agent. Too short period can
cause excessive burden to the control plane of the managed cluster, so generally
a recommended lower bound for the interval is 5 second.

In the end, the scraped values from feedback rules will be shown in the status:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata: ...
spec: ...
status:  
  resourceStatus:
    manifests:
    - conditions: ...
      resourceMeta: ...
      statusFeedback:
        values:
        - fieldValue:
            integer: 1
            type: Integer
          name: ReadyReplicas
        - fieldValue:
            integer: 1
            type: Integer
          name: Replicas
        - fieldValue:
            integer: 1
            type: Integer
          name: AvailableReplicas
        - fieldValue:
            string: "True"
            type: String
          name: isAvailable
```

## Garbage collection

To ensure the resources applied by `ManifestWork` are reliably recorded, the work agent creates an `AppliedManifestWork` on the managed cluster for each `ManifestWork` as an anchor for resources relating to `ManifestWork`. When `ManifestWork` is deleted, work agent runs a `Foreground deletion`, that `ManifestWork` will stay in deleting state until all its related resources has been fully cleaned in the managed cluster.

### Delete options

User can explicitly choose not to garbage collect the applied resources when a `ManifestWork` is deleted. the user should specify the `deleteOption` in the `ManifestWork`. By default, `deleteOption` is set as `Foreground` 
which means the applied resources on the spoke will be deleted with the removal of `ManifestWork`. User can set it to
`Orphan` so the applied resources will not be deleted. Here is an example:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata: ...
spec:
  workload: ...
  deleteOption:
    propagationPolicy: Orphan
```

Alternatively, user can also specify a certain resource defined in the `ManifestWork` to be orphaned by setting the
`deleteOption` to be `SelectivelyOrphan`. Here is an example with `SelectivelyOrphan` specified. It ensures the removal of deployment resource specified in the `ManifestWork` while the service resource is kept.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: selective-delete-work
spec:
  workload: ...
  deleteOption:
    propagationPolicy: SelectivelyOrphan
    selectivelyOrphans:
      orphaningRules:
      - group: ""
        resource: services
        namespace: default
        name: helloworld
```

## Resource Race and Adoption

It is possible to create two `ManifestWorks` for the same cluster with the same resource defined.
For example, the user can create two `Manifestworks` on cluster1, and both `Manifestworks` have the
deployment resource `hello` in default namespace. If the content of the resource is different, the
two `ManifestWorks` will fight and it is desired since each `ManifestWork` is treated as equal and
each `ManifestWork` is declaring the ownership of the resource. If there is another controller on 
the managed cluster that tries to manipulate the resource applied by a `ManifestWork`, this
controller will also fight with work agent.

When one of the `ManifestWork` is deleted, the applied resource will not be removed no matter
`DeleteOption` is set or not. The remaining `ManifestWork` will still keep the ownership of the resource.

 To resolve such conflict, we introduce `updateStrategy` in `0.8.0` release. User can choose a different 
 update strategy to alleviate the resource conflict.

- `CreateOnly`: with this strategy, the work-agent will only ensure creation of the certain manifest if the 
  resource does not exist. work-agent will not update the resource, hence the ownership of the whole resource
  can be taken over by another `ManifestWork` or controller.
- `ServerSideApply`: with this strategy, the work-agent will run server side apply for the certain manifest. The
  default field manager is `work-agent`, and can be customized. If another `ManifestWork` or controller takes the
  ownership of a certain field in the manifest, the original `ManifestWork` will report conflict. User can prune
  the original `ManifestWork` so only field that it will own maintains.

An example of using `ServerSideApply` strategy as following:

1. User creates a `ManifestWork` with `ServerSideApply` specified:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: hello-work-demo
spec:
  workload: ...
  manifestConfigs:
    - resourceIdentifier:
        group: apps
        resource: deployments
        namespace: default
        name: hello
      updateStrategy:
        type: ServerSideApply
```

2. User creates another `ManifestWork` with `ServerSideApply` but with different field manager.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: hello-work-replica-patch
spec:
  workload:
    manifests:
      - apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: hello
          namespace: default
        spec:
          replicas: 3
  manifestConfigs:
    - resourceIdentifier:
        group: apps
        resource: deployments
        namespace: default
        name: hello
      updateStrategy:
        type: ServerSideApply
        serverSideApply:
          force: true
          fieldManager: work-agent-another
```

The second `ManifestWork` only defines `replicas` in the manifest, so it takes the ownership of `replicas`. If the
first `ManifestWork` is updated to add `replicas` field with different value, it will get conflict condition and
manifest will not be updated by it.

In stead of create the second `ManifestWork`, user can also set HPA for this deployment. HPA will also takes the ownership
of `replicas`, and the update of `replicas` field in the first `ManifestWork` will return conflict condition.
