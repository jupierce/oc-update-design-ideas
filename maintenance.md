Maintenance Windows and Draining
================================

# Design Goals
1. Ensure that HyperShift / OCM has a first class experience when managing maintenance windows.
2. Ensure that Service Delivery has a means of ensuring viable update windows (e.g. a customer cannot provide only 15 minutes a month).
3. Replace aspects of the ManagedUpgradeOperator (overriding PDBs on drain timeout / surge / etc) with core product.
4. Provide similar drain & surge functionality for standalone cluster installations.
5. Provide these experiences while furthering the goals of "oc adm update status", "pre-checks", and separating control-plane and worker node upgrades for standalone.
6. Minimize changes to HyperShift where HyperShift already supports a concept.

# Discussions
- Meeting with James and Seth: https://drive.google.com/file/d/1qWcdrMlntn4fobuYMkgQQimKuLnFP5GL/view?pli=1
- Meeting Notes: https://docs.google.com/document/d/1JE6Q5xKuTgszrrvj7igTWm32suK_UUG35Ep-beDir1U/edit

# Background

## Standalone Upgrades
Currently, when a standalone OpenShift cluster is updated, an updated control-plane is rolled
out and, subsequently, worker-nodes are updated by the MachineConfigOperator
which will continuously attempt to migrate worker-nodes to match the version of the
control-plane (unless the process is explicitly paused with a change to a 
node MachineConfigPool).

The MCO will drain nodes based on a `maxUnavailable` in each MachineConfigPool which represents to the MCO
the maximum number of nodes it can try to update at any given time (either as an
integer or percentage).


## HyperShift Upgrades
A HyperShift "management cluster" can manage a number of "hosted clusters". The HyperShift operator
will monitor for the creation of "[HostedCluster](https://hypershift-docs.netlify.app/reference/api/#hypershift.openshift.io/v1beta1.NodePool)" 
resources. In response, HyperShift will create a [CAPI](https://pradeepl.com/blog/kubernetes/kubernetes-cluster-api-capi-an-introduction/) 
"Cluster" resource -- to which the CAPI [cluster controller](https://cluster-api.sigs.k8s.io/developer/architecture/controllers/cluster) 
will react by ultimately creating control-plane components with no worker nodes associated with it. Much of the work of 
installing the control-plane is done by the [infrastructure provider](https://cluster-api.sigs.k8s.io/developer/architecture/controllers/cluster#infrastructure-provider)
which allows the generic CAPI resource to work across multiple disparate environments.

For this HostedCluster to run workloads, one or more [NodePool](https://hypershift-docs.netlify.app/reference/api/#hypershift.openshift.io/v1beta1.NodePool) 
resources must be associated with the HostedCluster.

Unlike Standalone clusers, NodePools already support the concept of a worker node update independent from the control-plane. Each NodePool has
a separately configurable [release payload](https://github.com/openshift/hypershift/blob/15c5cddfb44ca2636deca17fd6c3367336aff287/api/v1beta1/nodepool_types.go#L86).

The HyperShift controller will respond to the creation or update of a NodePool by creating or updating either a 
CAPI MachineDeployment or CAPI MachineSet (apparently this depends on the "platform").

NodePools (implemented under the covers by CAPI MachineDeployments/MachineSets) offer additional flexibility when it comes to draining and upgrading nodes.
NodePool Spec:
- [Node drain timeout](https://github.com/openshift/hypershift/blob/15c5cddfb44ca2636deca17fd6c3367336aff287/api/v1beta1/nodepool_types.go#L127-L134) .
- [Paused Until](https://github.com/openshift/hypershift/blob/15c5cddfb44ca2636deca17fd6c3367336aff287/api/v1beta1/nodepool_types.go#L145-L150) which prevents the NodePool from reconciling with CAPI resources until a specified date or indefinitely (if set to "true").
- [Node Pool Management](https://github.com/openshift/hypershift/blob/15c5cddfb44ca2636deca17fd6c3367336aff287/api/v1beta1/nodepool_types.go#L336-L361) which specifies whether nodes should be replaced xor update "in place".
  - For replacement, different options are supported:
    - "RollingUpdate" surge strategy which supports a (potentially 0 if surge>0) [maxUnavailable](https://github.com/openshift/hypershift/blob/15c5cddfb44ca2636deca17fd6c3367336aff287/api/v1beta1/nodepool_types.go#L270-L289) and [surge amount](https://github.com/openshift/hypershift/blob/15c5cddfb44ca2636deca17fd6c3367336aff287/api/v1beta1/nodepool_types.go#L291-L310) where the cluster can temporarily scale above the configured number of nodes. 
    - "OnDelete" where machines are replaced when the user deletes "Machine" resources.
  - For "in place", [no new nodes will be created or deleted](https://github.com/openshift/hypershift/blob/15c5cddfb44ca2636deca17fd6c3367336aff287/api/v1beta1/nodepool_types.go#L313-L314). In this configuration, a `maxUnavailable` must be set to a positive number or percentage. 

MachineDeployment offers a [machine deployment strategy](https://github.com/kubernetes-sigs/cluster-api/blob/efb06799d0efa082a74f39976ee29587014fcf15/api/v1beta1/machinedeployment_types.go#L122) 
configuration spec.
- [Rolling Update Strategy](https://github.com/kubernetes-sigs/cluster-api/blob/efb06799d0efa082a74f39976ee29587014fcf15/api/v1beta1/machinedeployment_types.go#L172-L210) including surge replacement. 
- There is also a strategy called "[OnDelete](https://github.com/kubernetes-sigs/cluster-api/pull/4346)" where machines within a MachineSet are only replaced when they are deleted.

Once HyperShift updates a MachineDeployment, it presently has no interaction at all with the rollout of worker nodes.

## CAPI Notes
When exploring how to manage maintenance windows, the following may be interesting tools in the toolbox:
- [Experimental Runtime SDK](https://cluster-api.sigs.k8s.io/tasks/experimental-features/runtime-sdk/) which allows extensions to deeply tie into the lifecycle of Clusters / Machines.
- Cluster and MachineDeployment have pause semantics. In MachineDeployment, both a spec field (probably [buggy right now](https://github.com/kubernetes-sigs/cluster-api/issues/8629)) and an annotation `cluster.x-k8s.io/paused`. It looks like MachineSet supports pausing reconciliation when the [Cluster OR MachineSet is paused](https://github.com/kubernetes-sigs/cluster-api/blob/efb06799d0efa082a74f39976ee29587014fcf15/internal/controllers/machineset/machineset_controller.go#L156). 
- [OnDelete](https://github.com/kubernetes-sigs/cluster-api/pull/4346) for strict manual control? It does not look like the [machine deployment strategy](https://github.com/kubernetes-sigs/cluster-api/blob/efb06799d0efa082a74f39976ee29587014fcf15/api/v1beta1/machinedeployment_types.go#L122) supports "[in place propagation](https://cluster-api.sigs.k8s.io/developer/architecture/controllers/machine-deployment#in-place-propagation)" (i.e. updating it might case a full rollout).

# Design
Standalone clusters use MAPI and HyperShift uses CAPI on the management cluster. As such, we strive to have a similar
feel for both environments, but each system will need to have its own controller implementation.

## HyperShift / CAPI Maintenance Window
HyperShift/CAPI already exposes several configuration options that allow us to achieve some of our goals:
- Drain timeouts.
- Configurable surge.

But neither has the concept of a maintenance window. We can try to implement that window using the pausing features
exposed by CAPI (though there may be [bugs to fix](https://github.com/kubernetes-sigs/cluster-api/issues/8629)).

The higher level HyperShift controller would interpret a new specification field in `HostedCluster`.

```yaml
kind: HostedCluster
spec:
  versionManagement:  
    # The maintenanceWindows of a HostedCluster controls the times during which 
    # reconciliation is permitted for either control plane or worker nodes.
    # Outside of maintenance windows (or inside any exclusion window), HyperShift
    # will not reconcile release changes with the associated CAPI Cluster object.
    # An administrator who wishes to completely disable update related reconciliation 
    # regardless of maintenanceWindows may set spec.versionManagement.pausedUntil to prevent
    # further updates.
    # When paused for any reason (maintenance or pausedUntil), HyperShift will
    # set the backing CAPI Cluster object to be paused. This should stop
    # MachineDeployments/Sets from reconciling.
    maintenanceWindows:
      permit: "FREQ=WEEKLY;INTERVAL=1;BYDAY=SU,SA"
      exclude:
      - begin: 2023-12-20
        end: 2024-01-03
        
    # Similar to NodePool.spec.pausedUntil, but specific to MachineDeployment
    # release updates. HostedCluster.pausedUntil pauses ALL reconciliation.
    pausedUntil: false
    
    # A boolean value indicating whether versionManagement options are in effect.
    # If false, pausedUntil and maintenanceWindows are ignored. This is intended
    # to provide administrators / OCM an easy way to allow an update to proceed
    # without having to delete spec.versionManagement stanzas. When the update
    # is complete, they can set enforcing=true, and their previous settings will
    # take effect once more.
    enforcing: true
          
status:
  versionManagement:
    nextWindow: "2023-09-01 00:00:00UTC"
    nextWindowDuration: "48h"   
    changesPending: true
```

`NodePool` will also be extended to support the maintenance window concept. `NodePool` maintenance
windows can only reduce the maintenance windows expressed by `HostedCluster`. `NodePool`, like
`HostedCluster` can be independently paused with `spec.pausedUntil`.

```yaml
kind: NodePool
spec:
  versionManagement:
    # When outside of a maintenance window, within an exclusion window,
    # or explicitly paused, the NodePool controller will pause the associated
    # CAPI MachineDeployment/Set so that they stop reconciliation.
    # CAPI MachineSet already respects pause at both Cluster and MachineSet.
    maintenanceWindows:
      exclude:
      - begin: 2023-12-20
        end: 2024-01-03
        
    # Similar to NodePool.spec.pausedUntil, but specific to MachineDeployment
    # release updates. NodePool.spec.pausedUntil pauses ALL reconciliation. 
    # NodePool.spec.versionManagement.pausedUntil field pauses only 
    # reconciliation related to version management.
    pausedUntil: true

    # Setting enforcing=false cannot override an enforcing window 
    # at the HostedCluster level.
    enforcing: true
      
status:
  versionManagement:
    # window calculation respects HostedCluster
    clusterConstraint:
      nextWindow: "2023-09-01 00:00:00UTC" 
      nextWindowDuration: "48h"
    # NodePool window calculation must fall within HostedCluster constraint.
    nextWindow: "None. Paused with pausedUntil"
    changesPending: true
```

Having independent windows allows an administrator to capture semantics like:
- Update my control plane frequently, but don't update my workers until I explicitly unpause a MachineSet during a valid maintenance window.

### Constraints
ServiceDelivery will want to restrict the update policies that can be configured. They will likely want to ensure that:
- There are adequate non-exclusion periods.
- Maintenance windows are sufficiently large (e.g. 4 hours) to make steady progress.

## Standalone / MAPI Maintenance Window

The ClusterVersion resource will be augmented to support a maintenance window (analogous to HostedCluster in HyperShift).

```yaml
kind: ClusterVersion
spec:
  versionManagement:  # Intended to mirror HostedCluster.versionManagement.
    # The maintenanceWindows of a standalone cluster controls the times during which 
    # update related reconciliation is permitted for either control plane or worker nodes.
    # Outside of maintenance windows (or inside any exclusion window), CVO
    # will not reconcile release changes with the associated MachineConfig objects.
    # An administrator who wishes to completely disable reconciliation 
    # regardless of maintenanceWindows may set spec.versionManagement.pausedUntil to prevent
    # further updates.
    # When paused for any reason (maintenance or pausedUntil), HyperShift will
    # set the backing CAPI Cluster object to be paused. This should stop
    # MachineDeployments/Sets from reconciling.
    maintenanceWindows:
      permit: "FREQ=WEEKLY;INTERVAL=1;BYDAY=SU,SA"
      exclude:
      - begin: 2023-12-20
        end: 2024-01-03
          
    enforcing: true
        
status:
  versionManagement:
    nextWindow: "2023-09-01 00:00:00UTC"
    nextWindowDuration: "48h"   
    changesPending: true
```

Analogous to HyperShift NodePool, MachineConfigPool will be updated with a `versionManagement`
specification. To improve the flexibility and reliability of updates, existing NodePool specs
will be adopted into MachineConfigPool. For example, [NodePool.NodePoolManagement](https://hypershift-docs.netlify.app/reference/api/#hypershift.openshift.io/v1beta1.NodePoolManagement)
provides fine-grained control over how a node is drained and replaced during an update.
And [release](https://github.com/openshift/hypershift/blob/15c5cddfb44ca2636deca17fd6c3367336aff287/api/v1beta1/nodepool_types.go#L86)
provides the ability for Machines to run different versions of OCP.

```yaml
kind: MachineConfigPool
spec:
  versionManagement:
    # When outside of a maintenance window, within an exclusion window,
    # or explicitly paused, the MachineConfig controller will prevent any
    # associated Machines from being updated.
    maintenanceWindows:
      exclude:
      - begin: 2023-12-20
        end: 2024-01-03
    
    # Similar to MachineConfigPool.spec.pause, but specific to MachineConfigPool
    # release updates. MachineConfigPool.spec.pause pauses ALL reconciliation.
    # Like other pausedUntil, this value can be a datetime or "true"
    # for an indefinite pause.
    pausedUntil: true
    
    # Setting enforcing=false cannot override an enforcing window 
    # at the HostedCluster level.
    enforcing: true
      
  # Adopted from NodePool to create consistency and further our goal
  # to improve the reliability of worker updates.
  nodeDrainTimeout: 10m
      
  # New policy analog to NodePool.NodePoolManagement.
  machineManagement:
    upgradeType: "Replace"
    replace:
      strategy: "RollingUpdate"
      rollingUpdate:
        maxUnavailable: 0
        maxSurge: 4
      
  # New field to allow Machines associated with this set to run
  # different payloads than the control plane.
  release: 
    # Release payload image
    image: "quay.io....@sha256"
      
status:
  versionManagement:
    # window calculation respects ClusterVersion
    clusterConstraint:
        nextWindow: "2023-09-01 00:00:00UTC" 
        nextWindowDuration: "48h"
    # NodePool window calculation must fall within ClusterVersion constraint.
    nextWindow: "None. Paused with pausedUntil"
    changesPending: true
    
```

### Command Line Integration
`oc adm update` will be enhanced with one or more sub-verbs to manage maintenance windows.

## Details
- Once a control plane update has been initiated, it cannot be paused or terminated.
- If a maintenance window duration is at least 1 hour, no *further* reconciliation will be performed during that time. This is intended to allow nodes to come back online approximately before the maintenance window ends. For durations less than one hour, the contract is simply that no further reconciliation will take place after the maintenance window is over.
- An administrator can reboot nodes, delete machines, scale up, etc. outside of a maintenance window without any functional impact. Rebooted or newly created machines may receive either an old or new version depending on how far along the update is and which resources have been modified. 

# Use Cases

## OCM Forces Control-Plane Update
Either through the ROSA CLI or OCM, the administrator will configure a maintenance window and exclusions.
Both interfaces put in place restrictions on the window / exclusions in order to ensure that there
are at least 4 hours each month.

When a valid reoccurring window and exclusions have been selected, OCM configures the HostedCluster 
and NodePool resources associated with the administrator's cluster.

If OCM must do an emergency rollout of a control plane version, they will set the desired release in
HostedCluster and HostedCluster.spec.versionManagement.enforcing=false . `enforcing=false` will ignore
the administrator's configuration and allow the control-plane update to proceed.

## Assisted Standalone Worker Update
When OCP is ready to deprecate the link between control plane and worker updates, the 
default install will set `MachineConfigPool.spec.versionManagement.pauseUntil: assisted` .

This is interpreted by controllers as "true" - meaning that worker nodes will not be
updated when the control-plane is updated. 

`oc adm update worker-nodes start` can be used by administrators to start the "assisted" update.
When start is triggered, if `pausedUntil=true` then oc will set this field to 
`pausedUntil=false`. It will also annotate the `MachineConfigPool` with something like `assisted-update=true`

When Machines associated with the MachineConfigPool have fully reconciled and updated, if the MachineConfigPool
has the `assisted-update=true` annotation, then `pausedUntil` will be set back to "true"
and the annotation will be removed.

In short, using assisted worker node updates means:
- Worker node updates will not start automatically when the control-plane updates.
- The worker update process will proceed while respecting maintenanceWindows/exclusions, if configured and enforcing.

## Machine by Machine Update in Standalone
An administrator trusts control-plane updates, but wants full control over exactly when and
how workload nodes are replaced.

`ClusterVersion.spec.versionManagement.maintenanceWindow.enforcing=false` allows control-plane
updates to be triggered at any time.

But the administrator configures `MachineConfigPool.spec.machineManagement.replace.strategy=OnDelete` . 
This allows all reconciliation, but Machines will not automatically be replaced. 

The administrator manually drains and reboots/deletes machines as they see fit.

## Replicating the Current Update Process
The 4.14 update process updates the control-plane and then workers begin to be updated. If this 
is the desired experience, then the administrator should set `ClusterVersion.spec.versionManagement.pausedUntil=false` .
This will cause any reconciliation to immediately start the machine roll out (while still respecting
configured and enforcing maintenanceWindows/exlcusions).

## Standalone Worker Node Rollback
After an upgrade of worker nodes, the administrator determines that workloads are malfunctioning.
Red Hat support suggests rolling back `MachineConfigPool.spec.release.image` and starting the assisted 
update process.

## Pausing Worker Updates on Standalone
A MachineConfigPool has taken a long time to drain and the administrator of a standalone cluster
wants to pause the rest of the rollout and resume the next day.

They set `MachineConfigPool.spec.versionManagement.pausedUntil=true`. The assisted update process
will stop. The administrator can resume the update the next day by setting `pausedUpdate=false`.

## Canary Update on Standalone
A canary upgrade allows an administrator to test workloads on a new version of the platform
without committing to it. Canary updates, here, are limited to worker nodes (control-plane
updates affect all masters).

With the features described in this design, canary upgrades can be achieved in several
ways. One straightforward method will be described. 

*Control Plane Setup*
The administrator has recently updated their control-plane. Their control-plane 
updates are controlled using a maintenance window configured in `ClusterVersion`. The cluster
is outside of a maintenance window when the administrator decides to perform the canary 
testing.

*Main MachineConfigPool Setup*
The cluster has one MachineConfigPool for worker nodes (mcp-a), machines of which
are currently one version behind the control-plane. To ensure they have control over when
a new release is rolled out to workers, the administrator has the `MachineConfigPool` configured
with `versionManagement.pausedUntil=true`.

*Canary MachineConfigPool Setup*
The administrator creates a new `MachineConfigPool` (mcp-b). They configure the 
`release` field to point to the release payload currently associated with the control-plane.

*Process*
The administrator scales up `mcp-b`. The `release` value in this `MachineConfigPool` match
the current release of the control-plane, so the version of software running on the newly
created nodes matches the control-plane.

The administrator then cordons nodes associated with `machine-set-a` so that they cannot 
accept new workloads.

Next, the administrator drains one or more of the existing nodes from `machine-set-a`. 
This should force workloads off of `machine-set-a` and on to `machine-set-b`. 

As drained workloads are rescheduled onto the canary nodes, the functionality of the 
workloads running on the canary nodes should be tested.

*Rollback*
If testing fails, the administrator can uncordon nodes from `machine-set-a` and
delete `machine-set-b`. This will drain the canary nodes and return the workloads
to workers running the previous version of the platform.

*Adoption*
The administrator may choose to delete `machine-set-a` and keep workloads on 
the new nodes of `machine-set-b`. However, if they prefer, the administrator can change 
the release field on `machine-set-a` to match the new release. They can then trigger 
an assisted update of `machine-set-a`.  The already cordoned/drained nodes should
quickly be replaced. The administrator may then delete `machine-set-b`. `machine-set-b`
nodes will be drained and the workloads will migrated back to the now up-to-date 
`machine-set-a` nodes.
