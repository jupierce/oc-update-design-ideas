# Background
Currently, when an OpenShift cluster is updated, an updated control-plane is rolled
out and, subsequently, worker-nodes are updated by the MachineConfigOperator
which will continuously attempt to migrate worker-nodes to match the version of the
control-plane (unless the process is explicitly paused with a change to the 
node's MachineConfigPool).

The MCO will drain nodes based on a `maxUnavailable` in each MachineConfigPool which represents to the MCO
the maximum number of nodes it can try to update at any given time (either as an
integer or percentage).

# Goals
## Change perception of control-plane/worker-node updates
We want to decouple control-plane updates from worker-node updates to help customers
understand that the control-plane is a reliable and easy component to update, while 
worker-nodes and their workloads are typically the source
of long / problematic upgrades.

## Introduce maintenance windows
We want to marry this idea with the concept of maintenance windows. Customers who may
feel comfortable updating their control-plane (since it does not cause disruption)
may want to have strict controls around when their worker-nodes are updated (which is
prone to disruption based on workload configurations).

To provide those administrators with more control around their worker-node updates,
they will be able to specify maintenance windows and exclusion periods. Ultimately,
this should focus worker-node update activities only in windows the administrator
believes are low risk.

## HyperShift / Standalone
HyperShift has a very different topology compare to the classic standalone 
cluster. It uses NodePools and CAPI on the management cluster and 
MachineSets MAPI on the hosted cluster. There are no MachineConfigPools /
MachineConfig CRDS in the hosted cluster. NodePool contains serialized
MachineConfig objects for the hosted cluster.

Everything we develop should be consistent with and support this new topology.

# UpdatePolicies

A new CRD will be created called `UpdatePolicy` which will expose durable administrator
preferences for how a cluster should be updated. 

UpdatePolicy resources will be non-namespaced resources. In HyperShift deployments,
they will be instantiated on the management cluster and managed via OCM/ROSA cli. 

## Specification

```yaml
kind: UpdatePolicy
metadata:
  name: workers
spec:
  
  # Limits this policy to one or more machine roles.
  scopes: 
  - worker-nodes

  # An UpdatePolicy can apply to 0 or more sets of machines.
  # Depending on the context, it may apply to a MachineSet
  # on a standalone cluster or a NodePool on a management
  # cluster. 
  # By default, an UpdatePolicy applies to all machines
  # and all scopes.
  # This allows administrators to configure different
  # policies for different sets of machines (e.g. if
  # one set has disruption sensitive workloads and
  # another does not).
  selector:
    myPoolLabeledForPolicy: "easy-drain"

  # Disruption policy informs how nodes will be drained.
  # Does not apply to control-plane scope.
  disruption: 
    # "prevent" (default) means only graceful termination of workloads will 
    # be permitted. If pods cannot be terminated gracefully (e.g. due to 
    # pod disruption budgets), worker node updates may fail to progress 
    # indefinitely.
    # 
    # "permit" means that a configurable grace period will be provided 
    # for nodes to terminate and evict workloads after which the assisted 
    # update will force pods to be terminated to allow the update to progress. 
    # Services running on the cluster may be disrupted if they are misbehaving 
    # or misconfigured in such as way as to prevent draining.
    # 
    # "ignore" means assisted update will terminate pods without concern for 
    # grace periods or pod disruption budgets to progress through the 
    # update as quickly as possible. Services running on the cluster will 
    # likely be disrupted.
    policy: permit
    gracePeriod: 60

  replacement:
    policy: surge
    surge:
      # How many additional nodes to bring online during an update.
      maxSurge: 2
      # The number of nodes we can drain at a given time.
      maxUnavailable: 2

  # Define maintenance windows for this UpdatePolicy. 
  maintenance:
    # "window" is the default. It indicates the platform is allowed to roll out
    # updates across nodes during configured maintenance windows (exclusions override
    # windows). If there is no UpdatePolicy which applies to a group of machines,
    # then the window is considered open. This means the default behavior of OCP 
    # can remain the same. 
    #
    # "assisted" is a mode which will progress on an update the update is 
    # configured in its stanza. Eventually, we could ship a worker scoped
    # update policy in assisted mode. This means that the control-plane
    # could be updated without triggering worker-node updates UNTIL
    # `oc adm update worker-nodes start` configured the target version
    # in maintenance.assisted.forTarget: <version>.
    # 
    # "manual" pauses all update assistance. 
    policy: window
    # If enabled==false, any policy is paused indefinitely.
    enabled: true
    window:
        permit:
        - "FREQ=WEEKLY;INTERVAL=1;BYDAY=SU,SA"
        exclude:
        - begin: 2023-12-20
          end: 2024-01-03 
status:
  nextMaintenanceWindow: "2023-09-01 00:00:00UTC"
```

## Processing

To determine whether a set of machines can be updated, a controller must:
1. Sort all UpdatePolicies by resource `name` to ensure its decisions are deterministic even in the case of conflicting policies.
2. Find all UpdatePolicies which are applicable to the machineset (matched using labels) and its scope (control-plane, worker-nodes, *).
3. Iterate through the policies in order. Fields like `updateMode`, `disruption`, and `scaling` are overridden during the iteration. Maintenance windows and exclusions are aggregated into lists. 
4. After the applicable policies have been processed, if `updateMode` is manual, then updates are not permitted. Do not proceed.
5. If any of the list of exclusions periods contains the current datetime, then updates are not permitted. Do not proceed.
6. If none of the list of maintenance windows contains the current datetime, then updates are not permitted. Do not proceed.
7. Updates are permitted for the manchineset being analyzed.

## Constraints
ServiceDelivery will want to restrict the update policies that can be created. They will likely want to ensure that:
- That maintenance.policy == 'window'.
- There are adequate non-exclusion periods.
- Maintenance windows are sufficiently large (e.g. 4 hours) to make steady progress.

## Command Line Integration
Advanced administrators are welcome to craft UpdatePolicies themselves. If they do not, `oc adm update` will manage
one for them. If a user defined UpdatePolicy exists, the oc and update verbs which attempt to update the policy
will fail and advise the administrator to edit their policy directly.