# CrateDB on Kubernetes with Deployment 

The Deployment option puts the data on an NFS server.  This biggest down side to this is a single point of failure if the nfs-server becomes unavailable or your only PV becomes corrupt, and can potentially limit performance depending on physical I/O limitations of the PV.

It is preserved to demonstrate how to use a Kubernetes Deployment.  Plus, there are other reasons you might want to include an NFS sever in your solution, such as to host your conf, export backups of your data, or as a dev/test environment where you don't necessarily need a PV for each node.

## Data Volumes

Our Deployment configuration enables the ability to have the pods share a data volume by setting 

    node.max_local_storage_nodes=50

While Kubernetes 1.6 supports dynamic provisioning of persistent volumes, it currently does not have a way to dynamically provision a distinct volume for each replica in a Deployment.   On top of that, in GKE and other cloud providers, multiple pods cannot write to the same provisioned persistent volume.  

Fortunately, there are options such as NFS where your pods can all write to the same volume.  Combined with CrateDB's ability to support multiple Crate nodes on a single volume, you can achieve dynamic elasticity.  

This means that you can start with 3 nodes.  Then, with a single kubectl command, scale to 5 nodes, and CrateDB will handle it.  Kubernetes provides the ultimate platform for leveraging the scaling capabilities of CrateDB. 

## Dependencies

### Persistent Volume

With CrateDB, you will want your data to be on a persistent volume that survives the life-cycle of your Crate pods.  To use the CrateDB configuration provided, you need to setup an [nfs-server](https://github.com/erik777/kubernetes-nfs-server) in your cluster, which is very easy to do. 

You can remove the NFS mount from the crate-deployment.yaml, and the cluster will run just fine.  But, then each node will be using the root volume of the pod to store its data, meaning that when the pod is deleted, that node's data will be gone.  Of course, since this is a clustered solution that replicates the data across Crate nodes, it is possible to [lose a node without losing data](https://crate.io/docs/reference/best_practice/rolling_upgrade.html#rolling-upgrade).  
 But, when you delete your crate deployment, the data will be completely gone.   

You will need to know the ClusterIP address of your nfs-server, as K8S currently does not support discovery or name resolution in the declaration of your pod used by "kubectl create".  Somewhere in the documentation they said they plan to change this so you won't need to put the IP in your YAML in the future.  Until then...

    kubectl describe services nfs-server

Your initial folders you mount in your pod must also exist on your NFS volume before you create the Crate pods.  Otherwise, K8S will not deploy the pods, and you will see something like an rcpbind error in your monitoring.

Presuming your NFS server exports the root of a volume that is mounted to /exports, you will need to create the folder /exports/crate/data in your nfs-server, or just /crate/data in the root of the volume mounted to /exports, so it can mount /crate/data in your crate pods. 

### vm.max\_map\_count

CrateDB requires [vm.max_map_count to be at least 262166](https://crate.io/docs/reference/en/latest/configuration.html#virtual-memory).  You will get an error on startup if it is less.  Your K8S nodes could have this set to a lower value.  Options include changing the image used to create nodes to use 262166, manually changing a node after it is created, or, you can run a DaemonSet that automatically sets it when a new K8S node is created.  

To do the latter, use the provided init-daemonset.yaml:

    kubectl create -f init-daemonset.yaml

## Creating your CrateDB cluster

You will need to change the crate-deployment.yaml to replace YOUR_NFS_CLUSTER_IP with the ClusterIP.  

      volumes:
      - name: nfs-data
        nfs:
          # use "kubectl describe services nfs-server" to discover your NFS ClusterIP
          server: 252.12.25.34
          path: /crate/data

Then, you simply 

    kubectl create -f crate-deployment
    kubectl create -f crate-service
    
That's it!  If all went well, you should now have a CrateDB cluster.  You can use curl to test it.  

## Clearing the data

You can always clear the database by deleting your crate deployment from K8S, then clearing the contents of /crate/data on the NFS volume.  If you just created the nfs-server for this, you can delete the nfs server, including its PVC and PV.  

To do a complete removal, delete the services, too.  However, if are just deleting to recreate with a different configuration, and not toggling the LoadBalancer on or off, you can leave the services alone to preserve their IP assignments.  

## Scaling your cluster

Use kubectl to scale and watch your pods grow:
    
    kubectl scale deployments crate --replicas=4 
    kubectl get pods --watch

In GKE, if you have your K8S node pool configured to autoscale, you might see new VMs pop up to provide the needed capacity.  Likewise, if it created VMs to provide capacity for Crate, then deleting your Crate deployment should remove VMs.    

## Rolling update

For a rolling update, you can configure the max # of pods that can be down at one time and the max # that can be temporarily added.  For 3 replicas, this defaults to 1, meaning that during a rolling update, you will have 2-4 running pods at any given time.    

To update, you simply change the docker image to the next minor release available.  

    kubectl set image deployment/crate crate=crate:2.0.3 
    
    # Query pod images (versions)
    kubectl get pods --selector=app=crate -o jsonpath='{.items[*].spec.containers[*].image}'
    
Of course, be sure the image exists, and that the update will not break your application or the database.  To do an update that requires a restart, such as from 1.x to 2.x, you would simply change your YAML to the new image, delete your current deployment, backup your data, then recreate your deployment.  

Always check Crate release notes before updating a live database.  

If you just created a play deployment, have some fun and try an update.  If you are on the latest, you can delete your crate deployment, clear your /crate/data contents, change your YAML to the prior version, re-create, then do an update when that is up and running.  The latest version available is in [Docker Hub](https://hub.docker.com/_/crate/).

