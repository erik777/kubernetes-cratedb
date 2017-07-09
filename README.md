# kubernetes-cratedb
YAML for deploying a [CrateDB](https://crate.io/) cluster in [Kubernetes](https://kubernetes.io/) (K8S). 

Presumption is you have a working K8S cluster are comfortable deploying pods in K8S with kubectl and using related tools for monitoring.  

This is currently for Kubernetes 1.6 and has been tested on [Google Container Engine (GKE)](https://cloud.google.com/container-engine/). 

## Terminology

Kubernetes calls each VM or machine your pods run on a "node".  In a CrateDB cluster, each instance running is a Crate node.  In general, this documentation refers to the former as a "K8S node".  Otherwise, context is everything.   

## CrateDB for Elastic Scaling

[CrateDB](https://crate.io/) is a masterless horizontally scalable database that supports SQL while also supporting JSON documents.  Masterless means that "all nodes in a CrateDB cluster are identical, allowing simple and elastic scaling with high availability and replication." [source: crate.io](https://crate.io/overview/crate-vs-other-databases/)  

The elastic scaling capability makes it a good back-end database for microservices in a highly elastic environment like Kubernetes.  You can also do rolling updates of CrateDB with K8S. 

## Data Volumes

Each node in the CrateDB cluster has its own data volume, which, simply put, is a folder where CrateDB creates the physical database.  It is then up to the Crate nodes in the cluster to coordinate replication and other cluster behaviors, providing fail-over and performance capabilities.     

There are two approaches to defining the volumes for the nodes.  Either you create each node with its own volume, or you point the nodes to a shared folder, where it will create a node folder underneath for each DB node, a feature that CrateDB supports.  Note this latter feature was the default until 2.x.  Our configuration re-enables this feature by setting 

    node.max_local_storage_nodes=50

While Kubernetes 1.6 supports dynamic provisioning of persistent volumes, it currently does not have a way to dynamically provision a distinct volume for each replica in a deployment.   On top of that, in GKE and other cloud providers, multiple pods cannot write to the same volume.  

Fortunately, there are options such as NFS where your pods can all write to the same volume.  Combined with CrateDB's ability to support multiple Crate nodes on a single volume, you can achieve dynamic elasticity.  

This means that you can start with 3 nodes.  Then, with a single kubectl command, scale to 5 nodes, and CrateDB will handle it.  Kubernetes provides the ultimately platform for leveraging the scaling capabilities of CrateDB. 

## Dependencies

To use this configuration provided, you need to setup an [nfs-server](https://github.com/erik777/kubernetes-nfs-server) in your cluster, which is very easy to do. 

You can remove the NFS mount from the crate-deployment.yaml, and the cluster will still run.  But, then each node will be using the root volume of the pod to store its data, meaning that when the pod is deleted, that node's data will be gone.  Of course, since this is a clustered solution that replicates the data across Crate nodes, as long as you maintain [the minimum # of nodes for a Crate cluster to function](https://crate.io/docs/reference/architecture/shared_nothing.html#components-of-a-crate-node), to your DB client, the data will still be there.  But, when you delete your crate deployment, the data will be completely gone.   

You will also need to know the ClusterIP address of your nfs-server, as K8S currently does not support discovery or name resolution in the declaration of your pod used by "kubectl create".  Somewhere in the documentation they said they plan to change this so you won't need to put the IP in your YAML in the future.  Until then...

    kubectl describe services nfs-server

Your initial folders you mount in your pod must also exist on your NFS volume before you create the Crate pods.  Otherwise, K8S will not deploy the pods, and you will see something like an rcpbind error in your monitoring.

Presuming your NFS server exports the root of a volume that is mounted to /exports, you will need to create the folder /exports/crate/data in your nfs-server, or just /crate/data in the root of the volume, so it can mount /crate/data in your crate pods. 

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

## Confirming the node count

If the CrateDB pods form a cluster, the node count will be 3.  There are many ways you can verify it.  

If you are using GKE or another provider that supports the LoadBalancer service type, uncomment that line from the crate-service.yaml and re-create it.  This will put it on the public Internet.  While not a great idea for a real database you plan to use, it will allow you to view the UI at port 4200 with your web browser.  

If you terminal to a K8S node containing a Crate pod, you can use docker exec to /bin/sh into the crate node.  From there, if you ls /data/data/nodes, you should see 3 folders for the 3 nodes: 0, 1 and 2.  

Another way to confirm is to use Crate's SQL client tool called "crash" to query the number of nodes.  If you use **docker exec** to **/bin/sh** into one of the crate pods, you can do the following:

   
	# /crate/bin/crash
	cr> \c crate:4200
	+-------------------+---------------+---------+-----------+---------+
	| server_url        | node_name     | version | connected | message |
	+-------------------+---------------+---------+-----------+---------+
	| http://crate:4200 | Penne Blanche | 2.0.2   | TRUE      | OK      |
	+-------------------+---------------+---------+-----------+---------+
	CONNECT OK
	CLUSTER CHECK OK
	TYPES OF NODE CHECK OK
	cr> select count(*) from sys.nodes;
	+----------+
	| count(*) |
	+----------+
	|        3 |
	+----------+
	SELECT 1 row in set (0.072 sec)
 
The host name you connect to, "crate", resolves to the ClusterIP of the service.  Unlike when you created the deployment, inside a pod, you can just use the host name.  

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

## Possible next steps

Try with a newer cluster file systems such as GlusterFS instead of NFS and see what the performance impact is, and how it behaves when a FS node goes down.  

