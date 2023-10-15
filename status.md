Update Status and Pre-Checks
============================

Several CRDs serve as an API to aggregate and share insights about potential update operations (pre-checks)
and active update operations. A new controller, the Update Status Controller (USC), written and maintained by the
OTA team interprets and operators on these CRDs.

Insights can be extended by local or remote systems which register themselves as an `UpdateInformer`.

Overview
1. `UpdateInformer` - An instance of this resource uniquely identifies an entity that wishes to participate in informing administrators about ongoing or potential updates. The informer describes a REST API endpoint the Update Status controller will query when assessing ongoing or planned updates.
2. `UpdateStatus` - Used as a singleton to aggregate and confer status information pertinent to active update operations (and whether an update is even in progress). 
3. `UpdatePlan` - `oc adm update pre-checks` instantiates one of these resources for every target release an administrator is interested in analyzing for update issues. `UpdatePlan` objects represent a possible update and not an active update.

# UpdateInformer
The CVO is not the only entity on the platform that has insights to share about a planned or active update. The
MCO, CCX AI, optional operators, Service Delivery, etc. may all have update specific information to share. 

By allowing other components to contribute to the overall insights available for an administrator, we reduce 
the pressure on OTA to be an omnipresent observer / gatekeeper with respect to update status & pre-checks.

One reason for UpdateInformer is that it denotes to the update status controller which parties want to have a say in the
overall status. The controller will wait (within reason) for each UpdateInformer to contribute to UpdateStatus and UpdatePlans.

UpdateInformers are modeled on the [CAPI ExtensionConfig](https://github.com/kubernetes-sigs/cluster-api/blob/main/docs/book/src/tasks/experimental-features/runtime-sdk/implement-extensions.md#extensionconfig) 
design -- which allows arbitrary plugins into the lifecycle of a CAPI cluster.

Example:
```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateInformer
metadata:
  annotations:
    runtime.cluster.x-k8s.io/inject-ca-from-secret: openshift-machine-api/mco-update-informer-svc-cert
  name: mco-update-informer
spec:  

  # Administrators may disable informers if they are not 
  # providing value or any other problematic situations.
  enabled: false

  # UpdateInformers must expose a local service with a ClusterIP, but they may proxy 
  # requests to remote systems, if desired.
  clientConfig:
    service:
      name: test-runtime-sdk-svc
      namespace: openshift-machine-api
      port: 443

status:
  
  # Informers can expose conditions that suggest when they are functional.
  # Informers can update themselves and be temporarily offline. This may
  # cause synchronous calls like `oc adm update status` to wait for 
  # Informers to come back online if they are in the middle of an update.
  # In other cases, Informers may report they are degraded. 
  # `oc adm update status` will report that an informer is Degraded
  # as a separate insight. For pre-checks, a Degraded UpdateInformer
  # would manifest as another risk that administrators should fix
  # or accept.
  conditions:
  - type: Online
    status: "False"
    reason: InformerUpdating|Degraded
    message: "Informer is updating. Please wait."
    lastHeartbeatTime: "2023-11-17T14:18:26Z"
    lastTransitionTime: "2023-10-22T16:27:53Z"
```

## GET `/hooks.config.openshift.io/v1alpha/discovery`
Returns to the UpdateStatusController the API level supported by this UpdateInformer. It also 
defines which handlers have been implemented (e.g. 'status', 'plan')

## GET `/hooks.config.openshift.io/v1alpha/updateInsights`
This is the API to receive insights about in-progress updates. It serves to provide information through
`oc adm update status`.

The Update Status Controller (USC) will query these UpdateInformer endpoints when there is an ongoing 
update. The aggregation of UpdateInformer insights will be aggregated in the `UpdateStatus` CRD. 

Endpoints may cache data to provide quick responses (if their detection loops are long), but
caching should be minimized to provide users with up-to-date and actionable insights.

Request body (described in YAML here, but implemented with JSON)
```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateInsightsRequest
spec:
  # The clusterID in case a remote insight system needs to be able to identify the
  # cluster asking for insights.
  "clusterID": "...",

  controlPlaneVersion: ...
  targetControlPlaneVersion: ...
  
  # The request includes a suggestion `informerUpdate` indicating whether the caller wants the informer to check on possible 
  # updates to its software version. This might include an optional operator installing a new version of itself to provide
  # the latest available insights. Informers do not need to support updates and might not be able to if the cluster is
  # disconnected.
  "informerUpdate": true
```


Response Body (described in YAML here, but implemented with JSON)
```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateInsightsResponse
spec:
  
  # Each Informer can return a list of zero or more insights for the udpate.    
  insights:
  # Each insight has a UID. Between responses, the Informer should keep the UID of an insight
  # consistent. The UpdateStatusController will treat insights with different UIDs as 
  # fundamentally different insights.
  # The UpdateStatusController will use the UID to create a UUID for the insight by combining:
  # UUID: [informer-name][scope-elements][uid]
  # If a UUID stops being reported, it is assumed to have been resolved. 
  # UUIDs are even more critical for pre-checks as administrators "Accept" the risk of an insight
  # at the UUID level. If the UID or scope were to change on every request, the administrator
  # would be unable to accept all risks as new ones would continuously be displayed.
  - uid: "..."
    
    # Timestamp at which the data was acquired. Used to 
    # ascertain freshness when there is confusion about
    # whether a condition is resolved or not.
    acquisitionTime: "..."
    
    # Scope is an optional list of each object involved in the insight. Scope
    # cannot be adjusted without changing the UID.
    scope:
      type: ControlPlane # Or WorkerPool
      resources:
      # If the scope was WorkerPool, the impacted MAPI or CAPI resources 
      # would also be identified (e.g. MachineSet).
      - kind: Node
        name: ip-10-0-137-108.ec2.internal
      - kind: Pod
        namespace: a-namespace
        name: api-server

    impact:
      level: warning
      type: api-availability
  
      summary: "High control plane CPU during upgrade"
      
      # Only shown when status is run with --details or selected with --node|--selector.
      description: |
        Abnormally high CPU utilization has been detected during the control plane node upgrade. 
  
    remediation:
      
      # Where administrators can find information and workflows to 
      # prevent the warning in the future (e.g. scale your control plane
      # nodes before the next upgrade).
      reference: https://docs.redhat.com/openshift/updates/mco-issues#HighCpuMasterDuringUpgrade
      
      # If we want the administrator's attention even after the upgrade, the issue can be
      # marked as unresolved.
      resolved: false      

      # IF the Informer can estimate when the condition is likely to resolve, 
      # then it can include an estimated finish time which will feed into ETAs 
      # in the status UIs. This should normally only be provided by system level
      # insights like "Node Rebooting".
      estimatedFinish: <datetime>

status:
  # Failure: When responding with Failure, if possible, the Response should include an
  # insight stanza explaining the problem.
  # Wait: If the Informer is building fresh data and wants the caller to try back later, it can respond with 
  # "Wait". This will cause the UpdateStatusController to check back after 5 seconds.
  response: "Success|Failure|Wait"
  message: "..."
```


## POST `/hooks.config.openshift.io/v1alpha/updatePlanInsights`
An `UpdatePlanInsightsRequest` and associated plan response is very similar to the previously described
`UpdateInsightsRequest` and response.

It differs semantically by asking "What might be a problem IF we triggered an update to version X?" instead 
"What are the insights about the current attempt to update?"

For example, a Plan insight by suggest the presence of an undrainable PDB/node, but it would not report
on the status of an ongoing draining problem.

## Normal Progress
Insights also represent normal progress that can provide insights to `oc adm update status`.

```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateInsightsResponse
spec:
  
  insights:
  - uid: "..."
    acquisitionTime: "..."
    scope:
      type: WorkerPool
      workerPool:
      - kind: MachineSet
        namespace: openshift-machine-api
        name: my-machine-set
      resources:
      - kind: Node
        name: ip-10-0-137-108.ec2.internal

    impact:
      # status level updates are informational and represent expected 
      # steps for the update.
      level: status
      summary: "Node rebooting"
        
    remediation:
      estimatedFinish: 15m

status:
  response: "Success"
```


# UpdateStatus
UpdateStatus is the API for aggregated insights about in-progress updates (`oc adm update status`). The 
Update Status Controller keeps this object populated by querying `UpdateInformers`.

Example:
```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateStatus
metadata:
  name: cluster

spec:

status:
  controlPlane:
    updateInformers:
    - name: machine-config-update-informer
      insights: 
      - ... # Insights shared by this informer
    conditions:
    - type: UpdateActive # See UpdateStatus Conditions Section
      
  workerPools:
  - apiVersion: ...
    kind: MachineSet
    namespace: openshift-machine-api
    name: myMachineSet
    uid: ...
    updateInformers:
    - name: machine-config-update-informer
      insights: 
      - ... # Insights shared by this informer
    conditions:
    - type: UpdatePending # See UpdateStatus Conditions Section
  
```

## Conditions
Conditions in UpdateStatus represent a current picture of an ongoing or necessary update
reconciliation.

The cluster control-plane and "workerPools" each expose their own conditions. Worker
pools can be represented by MAPI or CAPI objects.

### UpdatePending Condition
Whenever a resource has not yet fully reconciled with its target machines,
an update is considered Pending.

This example shows a cluster with a single MachineSet whether neither
the control-plane nor the MachineSet has fully reconciled.
```yaml
status:
  contolPlane:
    conditions:  
    - type: UpdatePending
      status: True
      reason: ClusterVersion release has not reconciled.

  workerPools:
  - apiVersion: ...
    kind: MachineSet
    namespace: openshift-machine-api
    name: myMachineSet
    uid: ...
    conditions:
    - type: UpdatePending
      status: True
      reason: WorkerDoesNotMatchRelease
      message: At least one worker (...) does not match target release.
```

### UpdateActive Condition
True whenever an update reconciliation can make progress. This
requires (a) that there be a pending update and (b) that nothing is 
pausing the reconciliation. 

A control-plane update can be paused by any of the following (see maintenance.md for details on pausing and maintenance windows):
- `ClusterVersion.spec.versionManagement.pausedUntil` set to "true" or a future datetime.
- Being outside of the enforcing maintenanceWindow.
- Being within an enforcing exclusion time period.

A MachineSet update can be paused by:
- The control-plane update being paused for any reason.
- `MachineSet.spec.versionManagement.pausedUntil` set to "true" or a future datetime.
- Being outside of the enforcing MachineSet maintenanceWindow.
- Being within an enforcing MachineSet exclusion time period.

The following show an up-to-date control-plane and a worker pool within an 
active update period.
```yaml
  # Conditions showing control-plane is actively updating
  # but Machines are not.
  contolPlane:
    conditions:  
    - type: UpdateActive
      status: False
      reason: ClusterVersion release has reconciled.

  workerPools:
  - apiVersion: ...
    kind: MachineSet
    namespace: openshift-machine-api
    name: myMachineSet
    uid: ...
    conditions:
    - type: UpdateActive
      status: True
      reason: WorkersUpdateActive
      message: At least one worker (...) is being reconciled with release.
```

When updates are pending, but not Active, `UpdateActive` should describe why it is not active.

Here, updates are paused by `ClusterVersion.spec.versionManagement.pausedUntil: true`
```yaml
  workerPools:
  - apiVersion: ...
    kind: MachineSet
    namespace: openshift-machine-api
    name: myMachineSet
    uid: ...
    conditions:
    - type: UpdateActive
      status: False
      reason: AwaitingWindow
      message: Updates paused at cluster level.
```

When updates are paused, but not indefinitely due to a flag like `pausedUntil: true`, the reason
should instead be `AwaitingWindow` (e.g. waiting for a maintenance window).

```yaml
  workerPools:
  - apiVersion: ...
    kind: MachineSet
    namespace: openshift-machine-api
    name: myMachineSet
    uid: ...
    conditions:
    - type: UpdateActive
      status: False
      reason: AwaitingWindow
      message: Inside MachineSet exclusion window.
```


# UpdatePlan
UpdatePlans back the "oc adm update pre-checks" command. The objects are very similar to 
UpdateStatus. Like UpdateStatus, UpdateInformers provide the content for UpdatePlans:

1. Multiple UpdatePlans can exist.
2. They represent an analysis of a theoretical update vs an active one.
3. They can denote "accepted" risks that the administrator has signed off on.
4. By default, they are referenced by "oc adm update * start" to determine if all pre-check risks have been accepted.

The API would operate as follows:
1. `oc adm update status pre-checks --version 4.14.4` would create (or update) an UpdatePlan devoted to 4.14.4.
2. UpdateInformers would be queried by the Update Status Controller to find both general (e.g. undrainable PDB) and version specific update insights (e.g. conditional edge warning). 
3. `pre-checks` waits for UpdatePlanInsight status to populated.
4. Various UI facilities can but used to output the analysis of the pre-checks.
5. Eventually the administrator decides to perform the update.
6. They attempt to start the update.
7. Unless the behavior is overridden with `--pre-checks=false`, `oc adm update * start` checks for an UpdatePlan associated with the target release. It will not proceed until all identified analyses are marked as accepted risks.
8. The administrator uses `oc adm update pre-checks --accept` to accept the update risks / findings.
9. The administrator triggers the update operation again. Since pre-checks have been accepted, the update is initiated.

Validation prevents more than one UpdatePlan from being created for the same target version. UpdatePlans are
pruned over time.

# UpdatePlanInsight
UpdatePlanInsight is quite similar to UpdateStatusInsight, but, like UpdatePlan, they analyze theoretical
update activity.   

Example:
```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdatePlan
metadata: 
  name: update-plan-4.14.4:mco-update-informer-486
spec:
  updatePlanResourceVersion: "43980"
  targetRelease: "4.14.4"

  acceptedInsights:
  # The spec contains plan insights for which the administrator
  # has accepted the risk.
  - updateInformer: machine-config-update-informer
    uid: ....
    accepted: true
  
status:
  # See UpdateStatus status content.
```
