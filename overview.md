# Goals

## Change perception of control-plane/worker-node updates
We want to decouple control-plane updates from worker-node updates to help customers
understand that the control-plane is a reliable and easy component to update, while 
worker-nodes and their workloads are typically the source
of long / problematic upgrades.

## Provide an extremely clear update status
Customers report difficulty in interpreting why updates get stuck or fail.
Provide detailed, update specific, insights during the process along
with supporting documentation links to cover 90% of common issues.

## Provide pre-flight update checks
Before initiating an update, allow the customer to check for issues which
may interfere with a success update. Customers are informed of risks by
this "pre-checks" before an update is initiated.

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
