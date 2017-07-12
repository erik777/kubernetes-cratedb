# kubernetes-cratedb
YAML for deploying a [CrateDB](https://crate.io/) cluster in [Kubernetes](https://kubernetes.io/) (K8S). 

Presumption is you have a working K8S cluster and are comfortable deploying pods with kubectl and using related tools for monitoring.  

This is currently for Kubernetes 1.6 and has been tested on [Google Container Engine (GKE)](https://cloud.google.com/container-engine/). 

## Terminology

Kubernetes calls each VM or machine your pods run on a "node".  In a CrateDB cluster, each instance running is a Crate node.  In general, this documentation refers to the former as a "K8S node".  Otherwise, context is everything.   

## CrateDB for Elastic Scaling

[CrateDB](https://crate.io/) is a masterless horizontally scalable database that supports SQL while also supporting JSON documents.  Masterless means that "all nodes in a CrateDB cluster are identical, allowing simple and elastic scaling with high availability and replication." [source: crate.io](https://crate.io/overview/crate-vs-other-databases/)  

The elastic scaling capability makes it a good back-end database for microservices in a highly elastic platform like Kubernetes.  You can also do rolling updates of your CrateDB cluster with K8S. 

CrateDB was originally a fork of Elasticsearch, and uses its libraries today within it.  Yet, CrateDB added [interesting new capabilities](https://crate.io/a/how-is-crate-data-different-than-elasticsearch/).

## Kubernetes Deployment Options

The StatefulSet approach is probably the one you want to use.  It is much simpler to deploy, and puts the data of each CrateDB node on its own physical volume (PV).  

The Deployment option puts the data on an NFS server.  This biggest down side to this is a single point of failure if the nfs-server becomes unavailable or your only PV becomes corrupt, and can potentially limit performance depending on physical I/O limitations of the PV.

It is preserved to demonstrate how to use a Kubernetes Deployment.  Plus, there are other reasons you might want to include an NFS sever in your solution, such as to host your conf, export backups of your data, or as a dev/test environment where you don't necessarily need a PV for each node.

[StatefulSet](statefulset)

[Deployment](deployment)

## Data Volumes

Each node in the CrateDB cluster has its own data volume, which, simply put, is a folder where CrateDB creates the physical database.  It is then up to the Crate nodes in the cluster to coordinate replication and other cluster behaviors, providing fail-over and performance capabilities.     

There are two approaches to defining the volumes for the nodes.  Either you create each node with its own volume (StatefulSet), or you point the nodes to a shared folder (Deployment), where it will create a node folder underneath for each DB node, a feature that CrateDB supports.  

With both methods you can scale.  This means that you can start with 3 nodes.  Then, with a single kubectl command, scale to 5 nodes, and CrateDB will handle it.  Kubernetes provides the ultimate platform for leveraging the scaling capabilities of CrateDB. 

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

## Possible next steps

* Add health checks.

* Try with a newer cluster file systems such as GlusterFS instead of NFS and see what the performance impact is, and how it behaves when a FS node goes down.  

* Ensure rolling upgrade meets [CrateDB requirements](https://crate.io/docs/reference/best_practice/rolling_upgrade.html#rolling-upgrade) for a graceful stop.  [K8S Life-cycle hooks starter](https://pracucci.com/graceful-shutdown-of-kubernetes-pods.html). Refer to [K8S Volume Examples](https://github.com/kubernetes/kubernetes/tree/master/examples/volumes).  

