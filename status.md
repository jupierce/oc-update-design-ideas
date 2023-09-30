# oc-update-design-ideas

Several CRDs serve as an API to aggregate and share insights about potential update operations (prechecks)
and active update operations.

Overview
1. UpdateInformer - An instance of this resource uniquely identifies an entity in the system that wishes to participate in informing administrators about ongoing or potential updates.
2. UpdateStatus - Used as a singleton to aggregate and confer status information pertinent to active update operations (and whether an update is even in progress). 
3. UpdateStatusInsight - UpdateInformers (components of the platform, optional operators) create and manage UpdateStatusInsight objects in order to indicate issues with updates. Like ImageStreamTag is to ImagesStream, UpdateStatusInsight is independently addressable but also part of an UpdateStatus object. 
4. UpdatePlan - `oc adm update pre-checks` instantiates one of these resources for every target release an administrator is interested in analyzing without actually triggering it.
5. UpdatePlanInsight - UpdateInformers create and manage UpdatePlanInsight objects to inform UpdatePlan objects. Like ImageStreamTag is to ImagesStream, UpdatePlanInsight is independently addressable but also part of an UpdatePlan object. 

# UpdateInformer
The CVO is not the only entity on the platform that has insights to share about a planned or active update. The
MCO, CCX AI, optional operators, Service Delivery, etc. may all have details to share. By allowing other components to contribute to the
overall insights available for an administrator, we reduce the pressure on OTA to be an omnipresent observer / gatekeeper with
respect to update status & prechecks.

One reason for UpdateInformer is that it denotes to the update status controller which parties want to have a say in the
overall status. The controller will wait (within reason) for each UpdateInformer to contribute to UpdateStatus and UpdatePlans.

Example:
```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateInformer
metadata:
  name: mco-update-informer
status:
  # The update status controller must be able to determine whether a given informer
  # has an operational controller backing it. The heartbeat value should be updated
  # at least every X seconds by the informer's controller. Heartbeats that are out
  # of range will not be considered as required for displaying up-to-date status
  # information.
  heartbeat: <datetime>

  # Some informers can be updated asynchronously to the platform. For example, 
  # optional operators that may pull in new pre-check rules. This datetime
  # indicates the last time a check was made for updated informer code.
  # "oc adm update pre-checks --update" tries to induce updatable informers
  # to update themselves.
  lastInformerUpdateCheck: <datetime>
  lastInformerUpdateStatus: success 
```

# UpdateStatus
UpdateStatus is the API for in-progress updates (`oc adm update status`). UpdateInformers are responsible for
contributing and managing status insights in the object. The CVO/update status controller manages fields
in the objects which are not UpdateStatusInsights.

Example:
```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateStatus
metadata:
  name: update

  # When the CVO undertakes a new update, it will update resourceVersion in UpdateStatus 
  # resource. UpdateStatusInsight objects must match this value in 
  # their spec they are to be considered relevant to an active update. UpdateInformers will only
  # update UpdateStatusInsight resources with a matching version (others are orphaned), and instances
  # with old versions will be pruned eventually.
  resourceVersion: 43980
spec:
  # Is there an update in progress?
  phase: "inactive|active"
  
  # A field that is bumped whenever the user requests informers be updated. This should induce
  # updatable informers to update themselves. When oc adm update asks informers to update themselves
  # it will wait until UpdateInformers lastInformerUpdateCheck is >= informerUpdateTarget.
  informerUpdateTarget: <datetime>
  
  # Just a thought that we might have informers generate finer grained status insights
  # if the UpdateStatus resource requests it.
  informersDebug: false
  
  insights: []UpdateStatusInsight

```

# UpdateStatusInsight

An UpdateStatusInsight can carry a range of information. From normal progress updates to 
a significant degradation in the platform. 

One commonality between all UpdateStatusInsight objects is that they must have a field
`updateStatusResourceVersion` set to the current `resourceVersion` in the UpdateStatus
object they are informing. If these values do not match, then the analysis will not
be considered relevant to the current upgrade status.

UpdateInformers should update or create a new UpdateStatusInsight for the new UpdateStatus `resourceVersion`.
UpdateStatusInsight resources with older non-matching resourceVersions will eventually be pruned.

```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateStatusInsight
metadata: 
  name: ...
spec:
  # When the CVO undertakes a new update, it will set a resourceVersion in the UpdateStatus 
  # resource. UpdateStatusInsight objects must reference this version if they are
  # to be considered relevant to an active update.
  updateStatusResourceVersion: "43980"
  
  ...details in the following sections...
```

## Normal Progress
During the update process, components can feed information about normal proceedings to the 
UpdateStatus object. 

The update status controller _could_ inspect this information itself (e.g. annotations on machine/node) and set 
fields accordingly, but it may simplify the overall controller implementation if all aspects of update analysis feed through
the same logic and are produced by the decoupled components most intimately familiar with their part of the system.

```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateStatusInsight
metadata: 
  name: update:mco-update-informer-486
  
  # creationTimestamp informs how long an issue has been impacting the
  # update.
  creationTimestamp: <datetime>
  
spec:
  updateStatusResourceVersion: "43980"
  
  # The update status controller must be able to wait for multiple
  # UpdateInformers to perform their analysis and potentially report multiple
  # insights back into the UpdateStatus object. The UpdateInformer
  # controller will denote the completion of its scan with 
  # informerComplete: true. If the update status controller finds informerComplete: true
  # with a matching updateStatusResourceVersion, the associated UpdateInformer
  # will be considered fully reported in. It can add subsequent insights,
  # but all Informers being fully reported in unblocks initial UI reports
  # (e.g. "oc adm update status" would block until all UpdateInformers
  # have reported in -- within a reasonable timeout).
  informerComplete: false

  scope:
    # Nodes impacted by the finding
    nodes:
    - ip-10-0-137-108.ec2.internal
  
  # Noisy informers that aren't ultimately providing value to the administartor
  # may be silenced. 
  ignoreInformers:
  - an-annoying-opetional-operator

  impact:
    # System level insights with no impact are not reported individually in update status UIs like
    # `oc adm update status`. They are instead synthesized into higher level
    # concepts like reporting on the state of a node.
    level: system
    type: none
    # System events have codes. `oc adm update status` can unambiguously interpret
    # the semantics of the issue.
    # A completely clean report from an UpdateInformer can be represented by
    # code: 0 and informerComplete: true.
    code: 18
    
    summary: "Rebooting"
    
  remediation:
    resolved: false
    # Estimated finish times feed into ETAs in the status UIs.
    estimatedFinish: <datetime>
```

## Abnormal Occurrence
There are certain events which may occur during an update that we want to call to the administrator's
attention in order to ensure smoother updates are possible in the future.

Example:
```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateStatusInsight
metadata: 
  name: update:mco-update-informer-486
spec:
  updateStatusResourceVersion: "43980"

  scope:
    # Nodes impacted by the finding
    nodes:
    - ip-10-0-137-108.ec2.internal

  impact:
    level: warning
    type: api-availability

    # Two or more UpdateStatusInsight objects owned by the same UpdateInformer,
    # with the same code (or summary string) can exist suggest information for
    # the same node, but only the latest is considered relevant for the current
    # state of the update. The older insight can be retained as a resolved
    # issue in case it is useful.
    # This allows multiple acute occurrences of a resolved problem on a node to be reviewed
    # after an update has completed (e.g. high CPU utilization).
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
```

## Abnormal State
The most important insights that can be delivered through update status UIs are those that report an 
active abnormal state (e.g. an inability to drain a node).

Example:
```yaml
apiVersion: config.openshift.io/v1beta1
kind: UpdateStatusInsight
metadata: 
  name: update:mco-update-informer-486
spec:
  updateStatusResourceVersion: "43980"

  scope:
    # Nodes impacted by the finding
    nodes:
    - ip-10-0-137-108.ec2.internal

    resources:
    # When displayed with --details, the full impact description as well as the contentious
    # resources will be listed.
    - kind: Pod
      namespace: something
      name: hard-to-drain
    - kind: PodDisruptionBudget
      namespace: something
      name: makes-it-hard-to-drain

  impact:
    level: warning
    type: stall

    summary: "Pod disruption budget preventing node drain"
    
    description: |
      A pod disruption budget is preventing a node from draining. The worker node
      update will not be able to proceed until the condition is reolved.

  remediation:
    
    # In the update concepts slides, it is suggested that we have various update policies. Some of
    # which can blast through draining issues. If one of these policies was active, an
    # eta could be provided. 
    # estimatedFinish: <datetime>
    
    # Where administrators can find information and workflows to 
    # prevent the warning in the future (e.g. scale your control plane
    # nodes before the next upgrade).
    reference: https://docs.redhat.com/openshift/updates/mco-issues#PodDisruptionBudgetStall
    
  resolved: false      
```

# UpdatePlan
UpdatePlans back the "oc adm update pre-checks" command. The objects are very similar to 
UpdateStatus. Like UpdateStatus, UpdateInformers are responsible for updating UpdatePlans. 
Noteable differences include:

1. Multiple UpdatePlans can exist.
2. They represent an analysis of a theoretical update vs an active one.
3. They can denote "accepted" risks that the administrator has signed off on.
4. By default, they are referenced by "oc adm update * start" to determine if all pre-check risks have been accepted.

The API would operate as follows:
1. `oc adm update status pre-checks --version 4.14.4` would create (or update) an UpdatePlan devoted to 4.14.4.
2. UpdateInformers, watching UpdatePlans, would react to the new resource by computing any concerns they have specific to updates in general and to 4.14.4 specifically. This can include everything from PDB issues to insights about risky graph edges.
3. `pre-checks` waits for UpdatePlanInsight resources to be created by each UpdateInformer (see `informerComplete`).
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
kind: UpdatePlanInsight
metadata: 
  name: update-plan-4.14.4:mco-update-informer-486
spec:
  updatePlanResourceVersion: "43980"
  planForVersion: "4.14.4""

  scope:
    # Nodes impacted by the finding
    nodes:
    - ip-10-0-137-108.ec2.internal

    resources:
    # When displayed with --details, the full impact description as well as the contentious
    # resources will be listed.
    - kind: Pod
      namespace: something
      name: hard-to-drain
    - kind: PodDisruptionBudget
      namespace: something
      name: makes-it-hard-to-drain

  impact:
    level: warning
    type: stall

    summary: "Pod disruption budget will prevent a node drain"
    
    description: |
      A pod disruption budget will prevent a node from draining. A worker node
      update will not be able to complete without manual intervention.
   
  # Administrators can accept the risks reported in pre-checks. Unless pre-checks
  # are bypassed, 'oc adm update * start' will prevent an update from being triggered
  # if there are unaccepted UpdatePlanInsight results for the target version.
  accepted: false      
```
