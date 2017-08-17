# CrateDB on Kubernetes with StatefulSet 

## Dependencies

### Persistent Volume

With CrateDB, you will want your data to be on a persistent volume that survives the life-cycle of your Crate pods.  StatefulSet provides this capability.  

Each node created will have its own PersistentVolumeClaim (PVC) and corresponding PersistentVolume (PV) created.  Because the PV is created at the request of the PVC, it will continue to live so long as the PVC lives.  

While the PVC is created for each new node in your cluster, it does not get removed when the pods are removed.  So, if you start with 3 nodes, you will have 3 PVC which will be used to create 3 PVs (unless existing PVs meet the matching PVC criteria -- in most use cases, the PVs are created on-the-fly). 

When you scale to 5 nodes, 2 more PVC/PVs are created.  However, when you scale back to 3, or for any reason delete nodes, you will still have PVC/PVs, two no longer used.  But, the data is still on them.  Then, if you scale back to 5, no new PVs will be created.  It will simply re-attached to the same PVC/PVs it was using before.  This is because the pods and the PVC/PVs are all numbered, in this case from 0 to 4.  StatefulSet maintains order.  4 cannot run unless 3 is running first.  This provides predictability to which PVs your nodes will use as you scale.  

### vm.max\_map\_count

CrateDB requires [vm.max_map_count to be at least 262166](https://crate.io/docs/reference/en/latest/configuration.html#virtual-memory).  You will get an error on startup if it is less.  This is set using a container init in the pod definition.

## Clearing the data

You can always clear the database by deleting your crate deployment from K8S, deleting each of the PVCs.  If the PVCs created the PVs, this should delete them as well.  

To do a complete removal, delete the services, too.  However, if are just deleting to recreate with a different configuration, and not toggling the LoadBalancer on or off, you can leave the services alone.  

## Scaling your cluster

[Scale a StatefulSet](https://kubernetes.io/docs/tasks/run-application/scale-stateful-set/)

	kubectl scale statefulsets <stateful-set-name> --replicas=<new-replicas>

## Discovery

This is configured primarily through these two parameters for Kubernetes:

	- -Cdiscovery.type=srv
	- -Cdiscovery.srv.query=_cluster._tcp.crate-service.default.svc.cluster.local

In this, `crate-service` is the service name, as defined in `crate-service.yml`.

## Rolling update

TODO

# Credit

Thank you, aleks of pulseid.com, for converting the original Deployment example to a StatefulSet in [this Github project](https://github.com/pulse-id/pulse-crate-k8s).  
