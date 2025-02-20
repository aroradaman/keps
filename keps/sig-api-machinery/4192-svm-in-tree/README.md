# KEP-4192: Move Storage Version Migrator in-tree

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [UNCLEAR Goals and/or Non-Goals](#unclear-goals-andor-non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1 <strong>[UNCLEAR]</strong>](#story-1-unclear)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [APIs to move](#apis-to-move)
    - [We will move following <a href="https://github.com/kubernetes-sigs/kube-storage-version-migrator/blob/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/apis/migration/v1alpha1/types.go">APIs</a> in-tree:](#we-will-move-following-apis-in-tree)
    - [[Undecided] Changes while we move above APIs in-tree:](#undecided-changes-while-we-move-above-apis-in-tree)
  - [<a href="https://github.com/kubernetes-sigs/kube-storage-version-migrator/tree/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/controller">Controller</a> to move](#controller-to-move)
    - [<a href="https://github.com/kubernetes-sigs/kube-storage-version-migrator/tree/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/migrator">Migrator Controller</a>](#migrator-controller)
    - [Streaming List](#streaming-list)
  - [RBAC for SVM](#rbac-for-svm)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary
Kubernetes relies on resource data being actively re-written to perform certain maintenance actives related to at rest storage.  Two prominent examples are the versioned schema of stored resources (i.e. the preferred storage schema changing from `v1beta1` to `v1` for a given resource) and encryption at rest (i.e. rewriting `stale` data based on a change in how the data should be encrypted).

The simplest way to rewrite data is to issue no-op `update` requests via `kubectl get <resource> | kubectl replace -`.  This is problematic for any resource that can contain a large amount of data such as Kubernetes `secrets` and it is also impractical to perform without automation as the number of resources that need migration is always growing.

Unlike most Kubernetes API interactions, during storage migration, it is safe to ignore conflicts during `update` requests and it is safe to use inconsistent continue tokens during paginated `list` operations (since we only need data to be re-written, we do not care how it is re-written).

This KEP aims to make it easy for users to perform storage migrations without having to worry about the details.

## Motivation

### Goals

- Make it easy for an admin to trigger a storage migration without having to manually run `kubectl get | kubectl replace`
- Enable infrastructure providers to use internal details of their environment to trigger storage migration (ex: triggering storage migration of Kubernetes secrets when the KMS v2 key ID has changed)
- Make it easy for existing users of SVM to migrate to the in-tree controller

### Non-Goals

- Any automatic or direct integration with KMS v2
- Any modification regarding `StorageVersion` API for HA API servers
- Adding logic that relies on the hashed storage versions exposed via the discovery API

### UNCLEAR Goals and/or Non-Goals

- Automatic storage version migration for CRDs
- Make it easy for Kubernetes developers to drop old API schemas by guaranteeing that storage migration is run automatically on SV hash changes (should this also be on a timer or API server identity?)
- Automated Storage Version Migration via the hash exposed by the `StorageVersion` API

## Proposal

- Move the existing SVM controller logic in-tree into KCM
- Move the existing SVM REST APIs in-tree (possibly under a new API group to avoid conflicts with the old API being run concurrently)
- For Alpha, we will not trigger automatic storage version migration, and it will be deferred to the user.

### User Stories (Optional)

#### Story 1 **[UNCLEAR]**
As an end user of Kubernetes, we get automatic storage version migration whenever the storage version changes due to an api server upgrade/downgrade.

#### Story 2
As an end user using encryption at rest, whenever the key change is detected we can run the storage migration to use the new key for encryption. 

### Notes/Constraints/Caveats (Optional)

### Risks and Mitigations

## Design Details

### APIs to move
#### We will move following [APIs](https://github.com/kubernetes-sigs/kube-storage-version-migrator/blob/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/apis/migration/v1alpha1/types.go) in-tree:
- `v1alpha1` of `storageversionmigrations.migration.k8s.io`  

    ```go
    // StorageVersionMigration represents a migration of stored data to the latest
    // storage version.
    type StorageVersionMigration struct {
        metav1.TypeMeta `json:",inline"`
        // +optional
        metav1.ObjectMeta `json:"metadata,omitempty"`
        // Specification of the migration.
        // +optional
        Spec StorageVersionMigrationSpec `json:"spec,omitempty"`
        // Status of the migration.
        // +optional
        Status StorageVersionMigrationStatus `json:"status,omitempty"`
    }

    // Spec of the storage version migration.
    type StorageVersionMigrationSpec struct {
        // The resource that is being migrated from. The migrator sends requests to
        // the endpoint serving the resource.
        // Immutable.
        OriginResource GroupVersionResource `json:"resource"`
        // The token used in the list options to get the next chunk of objects
        // to migrate. When the .status.conditions indicates the migration is
        // "Running", users can use this token to check the progress of the
        // migration.
        // +optional
        ContinueToken string `json:"continueToken,omitempty"`
        // The storage version hash to avoid races.
        StorageVersionHash string `json:"storageVersionHash,omitempty"`
    }

    // The names of the group, the version, and the resource.
    type GroupVersionResource struct {
        // The name of the group.
        Group string `json:"group,omitempty"`
        // The name of the version.
        Version string `json:"version,omitempty"`
        // The name of the resource.
        Resource string `json:"resource,omitempty"`
    }
    
    // Status of the storage version migration.
    type StorageVersionMigrationStatus struct {
        // The latest available observations of the migration's current state.
        // +optional
        // +patchMergeKey=type
        // +patchStrategy=merge
        Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,1,rep,name=conditions"`
    }

    // StorageVersionMigrationList is a collection of storage version migrations.
    type StorageVersionMigrationList struct {
        metav1.TypeMeta `json:",inline"`
        // +optional
        metav1.ListMeta `json:"metadata,omitempty"`
        // Items is the list of StorageVersionMigration
        Items []StorageVersionMigration `json:"items"`
    }
    ```

- APIs in-tree will be _converted to `built-in types`_ from CRD.

#### [Undecided] Changes while we move above APIs in-tree:
To avoid any conflicts with the Storage Version Migrators running out of tree, we will change the _`group`_ from `migration.k8s.io` to `storagemigration.k8s.io`.

The final APIs that will be moved in-tree are:
- `v1alpha1` of `storageversionmigrations.storagemigration.k8s.io`

### [Controller](https://github.com/kubernetes-sigs/kube-storage-version-migrator/tree/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/controller) to move
#### [Migrator Controller](https://github.com/kubernetes-sigs/kube-storage-version-migrator/tree/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/migrator)
Currently, the Storage Version Migrator comprises two controllers: the `Trigger` controller and the `Migrator` controller. The Trigger controller performs resource discovery, identifying supported resources with the preferred server version every `10 minutes`. Subsequently, the Trigger controller creates the `StorageVersionMigration` resource to initiate the migration process. The Migrator controller then picks up this resource and executes the actual migration.

When transitioning the Storage Version Migrator in-tree, we will exclusively move the Migrator controller as a component of KCM. The creation of the Migration resource will be deferred to the user, instead of being triggered automatically.

#### Streaming List
Currently, the Migrator controller uses the `chunked List` method to retrieve the list of resources from the API server and subsequently perform storage migrations as needed. However, chunked lists are [resource-intensive]((https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/3157-watch-list/README.md#motivation)) and can lead to a significant overload on the API server, potentially resulting in it being terminated due to out-of-memory (OOM) issues. To address this concern, we have proposed the adoption of an Alpha feature introduced in Kubernetes _v1.27_, known as [`Streaming List`](https://kubernetes.io/docs/reference/using-api/api-concepts/#streaming-lists).

When the Migrator controller is integrated in-tree, it will leverage the `Streaming List` approach to obtain and monitor resources while executing storage version migrations, as opposed to using the resource-intensive `chunked list` method. In cases where, for any reason, the client cannot establish a streaming watch connection with the API server, it will fall back to the standard `chunked list` method, retaining the older LIST/WATCH semantics.
    
- _Open Question_:
    - Does `Streaming List` support inconsistent lists? Currently, with chunked lists, we receive inconsistent lists. We need to determine if we can achieve the same with Streaming List. Depending on the outcome, we may consider removing the [ContinueToken](https://github.com/kubernetes-sigs/kube-storage-version-migrator/blob/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/apis/migration/v1alpha1/types.go#L63) from the API.

### RBAC for SVM
- Storage Version Migrator Controller
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: system:controller:storage-version-migrator-controller
    rules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: 
      - get 
      - list
      - watch
      - update
      
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: system:controller:storage-version-migrator-controller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:controller:storage-version-migrator-controller
    subjects:
    - kind: ServiceAccount
      name: storage-version-migrator-controller
      namespace: kube-system
    ```

### Test Plan

- [x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

##### Unit tests
Existing out-of-tree unit tests will be moved in-tree and we will improve unit test coverage.

Current unit tests coverage:
```shell
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/helpers.go:25:         HasCondition                            100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/helpers.go:29:         indexOfCondition                        100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/helpers.go:38:         resource                                0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/indexer.go:39:         migrationStatusIndexFunc                87.5%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/indexer.go:53:         NewStatusIndexedInformer                100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/indexer.go:57:         ToIndex                                 100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/indexer.go:62:         migrationResourceIndexFunc              75.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/indexer.go:70:         NewStatusAndResourceIndexedInformer     100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/kubemigrator.go:47:    NewKubeMigrator                         0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/kubemigrator.go:56:    Run                                     0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/kubemigrator.go:66:    process                                 0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/kubemigrator.go:94:    processOne                              0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/controller/kubemigrator.go:142:   updateStatus                            0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/initializer/discover.go:39:       NewDiscovery                            100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/initializer/discover.go:71:       FindMigratableResources                 86.5%
sigs.k8s.io/kube-storage-version-migrator/pkg/initializer/discover.go:127:      findCustomGroups                        85.7%
sigs.k8s.io/kube-storage-version-migrator/pkg/initializer/discover.go:139:      findAggregatedGroups                    87.5%
sigs.k8s.io/kube-storage-version-migrator/pkg/initializer/initializer.go:44:    NewInitializer                          0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/initializer/initializer.go:67:    migrationCRD                            0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/initializer/initializer.go:191:   migrationForResource                    0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/initializer/initializer.go:212:   initializeCRD                           0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/initializer/initializer.go:245:   Initialize                              0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:54:              NewMigrator                             100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:63:              get                                     0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:71:              put                                     100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:79:              list                                    100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:87:              Run                                     38.2%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:144:             migrateList                             95.5%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:182:             worker                                  83.3%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:196:             migrateOneItem                          72.7%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:235:             try                                     66.7%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/core.go:257:             inconsistentContinueToken               0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/errors.go:32:            Temporary                               0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/errors.go:40:            Temporary                               100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/errors.go:51:            isConnectionRefusedError                100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/errors.go:57:            interpret                               25.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/errors.go:87:            canRetry                                100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/progress.go:39:          NewProgressTracker                      0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/progress.go:46:          save                                    0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/migrator/progress.go:58:          load                                    0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/controller.go:57:         NewMigrationTrigger                     100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/controller.go:73:         dequeue                                 0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/controller.go:97:         addResource                             60.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/controller.go:106:        deleteResource                          27.3%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/controller.go:123:        updateResource                          0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/controller.go:127:        enqueueResource                         100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/controller.go:136:        Run                                     0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:36:  processDiscovery                        79.2%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:75:  toGroupResource                         100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:84:  cleanMigrations                         75.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:107: launchMigration                         100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:121: relaunchMigration                       66.7%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:129: newStorageState                         100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:143: updateStorageState                      71.4%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:181: staleStorageState                       100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:185: processDiscoveryResource                66.7%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:219: isMigrated                              100.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/discovery_handler.go:226: hasPendingOrRunningMigration            80.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/migration_handler.go:33:  storageStateName                        66.7%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/migration_handler.go:42:  markStorageStateSucceeded               0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/migration_handler.go:68:  processMigration                        0.0%
sigs.k8s.io/kube-storage-version-migrator/pkg/trigger/migration_handler.go:83:  processQueue                            0.0%
total:                                                                          (statements)                            45.9%
```

##### Integration tests

- [ ] Write a test which will identify the migratable resource, create Migration resource for the same and test whether Storage Version Migration is actually carried out.

##### e2e tests

- [ ] Test to validate functionality of the Migration controller.

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled

#### Beta

- Resolve potential kube-apiserver OOMs and cascading failure issues that may occur due to the use of paginated lists.
- Feature is enabled by default
- All of the above documented tests are complete

### Upgrade / Downgrade Strategy

### Version Skew Strategy

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: _`StorageVersionMigrator`_
  - Components depending on the feature gate:
      - Kube Controller Manager (KCM)
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?

###### Does enabling the feature change any default behavior?
No

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes

###### What happens if we reenable the feature if it was previously rolled back?

It should start performing Storage Migration for the given migration request.

###### Are there any tests for feature enablement/disablement?
We will add integration tests to validate the enablement/disablement flow.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?
This may impact workloads, as issuing too many Storage Version Migration requests can increase the latency on the API server. Also, the current implementation of the feature uses paginating lists and retains each item in memory. This issue becomes critical because temporary memory consumption can grow exponentially. When the system exhausts its available memory, it risks slowing down or even halting the API server. This disruption can affect running workloads and exert significant pressure on the node itself, potentially impacting other processes, including Kubelet. Such a situation can lead to workload disruption. We are currently exploring the Streaming Watch feature to determine if and how it can be utilized to address this issue.

###### What specific metrics should inform a rollback?
The metric `storage_migrator_core_migrator_migrations_total` with `failed` status can indicate how many failures there are and can help decide if a rollback is required.

The following metrics should not be available if we perform a rollback:
- storage_migrator_core_migrator_migrated_objects_total
- storage_migrator_core_migrator_remaining_objects_total
- storage_migrator_core_migrator_migrations_total

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?
This will be covered by integration tests.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?
No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?
The following metrics are available when Storage Version Migration is enabled:
- storage_migrator_core_migrator_migrated_objects_total
- storage_migrator_core_migrator_remaining_objects_total
- storage_migrator_core_migrator_migrations_total

###### How can someone using this feature know that it is working for their instance?
- [ ] Events
  - Event Reason: 
- [ ] API .status
  - Condition name: **_MigrationCondition.Type (Running | Succeeded | Failed)_**
  - Other field: **_MigrationCondition.LastUpdateTime_**
- [x] Other (treat as last resort)
  - Details:
      -    The following metrics are available when Storage Version Migration is enabled:
            - storage_migrator_core_migrator_migrated_objects_total
            - storage_migrator_core_migrator_remaining_objects_total
            - storage_migrator_core_migrator_migrations_total

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?
NA

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?
- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [x] Other (treat as last resort)
  - Details:
      - The metric `storage_migrator_core_migrator_migrations_total` with the status `Failed` can be observed to determine the health of the migrations.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No.

### Scalability

###### Will enabling / using this feature result in any new API calls?

Yes. Creation of the Migration Request which creates `StorageVersionMigration` resource. 

###### Will enabling / using this feature result in introducing new API types?

Yes, the Storage Version Migration API.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

Yes. In Kube Controller Manager (KCM) and API server.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

Maybe (Not exactly sure how does this affect)

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

This feature rely on the API server and etcd availability. If they are not available then it will results in errors.

###### What are other known failure modes?

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

This feature is already implemented in [out-of-tree](https://github.com/kubernetes-sigs/kube-storage-version-migrator). We are moving this in-tree.

## Drawbacks

NA

## Alternatives

NA

## Infrastructure Needed (Optional)

NA
