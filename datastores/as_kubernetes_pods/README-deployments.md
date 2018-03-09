# Datastores as Kubernetes pod as Deployments

Sysdig Cloud datastores can be deployed as Kubernetes pods. Each pod can be configured to use a local volume (emptyDir) that is only persisted for the lifetime of the pod, or a persistent volume. The provided manifests contain examples for AWS EBS and GCE Disks, for other type of Kubernetes persistent volumes refer to http://kubernetes.io/docs/user-guide/persistent-volumes/#types-of-persistent-volumes

If using persistent volumes, those will need to be created separately before deploying the datastore pods.
If you use AWS you can create a volume using the command line `aws ec2 create-volume` or the AWS console, If you use GCE you can create a volume using the command line `gcloud compute disks create` or the GCE console.

Please notice that, when running a Kubernetes cluster in multiple zones, special care needs to be taken in maintaining the affinity between zones where the persistent volumes are created and zones where the pods are actually scheduled, since most cloud providers impose limitations. At the time of writing, the current solutions are:

- Use persistent volume claims instead of embedding the volumes inside the pod manifest (like in the following examples): http://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims
- Manually force the affinity of the datastore deployments to specific zones using the nodeSelector field: http://kubernetes.io/docs/user-guide/node-selection/

## MySQL

To create a MySQL deployment, the provided manifest under `manifests/mysql.yaml` can be used. By default, it will use a local non-persistent volume (emptyDir), but the manifest contains commented snippets that can be uncommented when using persistent volumes such as AWS EBS or GCE Disks (just replace the volume id from the cloud provider in the snippet).

Create MySQL service:
```
kubectl create -f manifests/services/mysql-service.yaml --namespace sysdigcloud
```

Create MySQL database:
```
kubectl create -f manifests/deployments/mysql-deployment.yaml --namespace sysdigcloud
```

## Redis

Redis doesn't require persistent storage, so it can be simply deployed as:
Create Redis service:
```
kubectl create -f manifests/services/redis-service.yaml --namespace sysdigcloud
```
Create Redis database:
```
kubectl create -f manifests/deployments/redis-deployment.yaml --namespace sysdigcloud
```

## Cassandra

Before deploying the deployment object, the proper Cassandra headless service must be created (the headless service will be used for service discovery when deploying a multi-node Cassandra cluster):

```
kubectl create -f manifests/services/cassandra-service.yaml --namespace sysdigcloud
```

To create a Cassandra deployment, the provided manifest under `manifests/cassandra-deployment.yaml` can be used. By default, it will use a local non-persistent volume (emptyDir), but the manifest contains commented snippets that can be uncommented when using persistent volumes such as AWS EBS or GCE Disks (just replace the volume id from the cloud provider in the snippet):

```
kubectl create -f manifests/deployments/cassandra-deployment.yaml --namespace sysdigcloud
```

This creates a Cassandra cluster of size 1. To expand the Cassandra cluster, a new deployment must be created for each additional Cassandra node in the cluster. You can't just scale the replicas of the existing deployment because each Cassandra pod must get a different persistent volume, so in that sense Cassandra pods are "pets" with unique identities and not "cattles".

In order for the new Cassandra deployment to automatically join the cluster, some conventions must be followed. In particular, the Cassandra node number (1, 2, 3, ...) must be properly put in the manifest `manifests/deployments/cassandra-deployment.yaml` under the entries marked as `# Cassandra node number`.

For example, to scale a Cassandra cluster from 2 to 3 nodes, the manifest can be edited as such:

```
...
metadata:
  name: sysdigcloud-cassandra-3 # Cassandra node number
...
      labels:
        instance: "3" # Cassandra node number
...
```

And then the deployment can be created as usual:

```
kubectl create -f manifests/deployments/cassandra-deployment.yaml --namespace sysdigcloud
```

After each scaling activity, the status of the cluster can be checked by executing `nodetool status` in one of the Cassandra pods. All the Cassandra nodes should be listed as `UN` in order for the cluster to be fully up and running. Immediately after the scaling activity, the new pod will be in joining phase:

```
$ kubectl --namespace sysdigcloud exec -it sysdigcloud-cassandra-1-2987866586-f5kgo -- nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens  Owns (effective)  Host ID                               Rack
UN  10.52.2.4  1.88 MB    256     54.4%             99121365-4543-4e50-ae6f-a9a9cb720b7c  rack1
UJ  10.52.0.4  14.43 KB   256     ?                 4b084d81-21f1-45b6-add9-8fbea7392978  rack1
UN  10.52.1.7  917.91 KB  256     45.6%             9a7437e9-890f-477a-99be-3d8042ddd9d5  rack1
```

After the bootstrapping process terminates, the new pod will terminate the joining phase and the cluster will be fully operational:

```
$ kubectl --namespace sysdigcloud exec -it sysdigcloud-cassandra-1-2987866586-f5kgo -- nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens  Owns (effective)  Host ID                               Rack
UN  10.52.2.4  1.88 MB    256     34.1%             99121365-4543-4e50-ae6f-a9a9cb720b7c  rack1
UN  10.52.0.4  14.43 KB   256     34.0%             4b084d81-21f1-45b6-add9-8fbea7392978  rack1
UN  10.52.1.7  917.91 KB  256     31.9%             9a7437e9-890f-477a-99be-3d8042ddd9d5  rack1
```

It is important for the number of deployments to be increased by no more than 1 at every scaling activity, since Cassandra will refuse the joining of a new node in the Cluster if one joining process is already in progress. 

Maintaining a multi-node production Cassandra cluster requires some simple but mandatory housekeeping procedures, best described in the official documentation.

## Elasticsearch

Before deploying the deployment object, the proper Elasticsearch headless service must be created (the headless service will be used for service discovery when deploying a multi-node Elasticsearch cluster):

```
kubectl create -f manifests/services/elasticsearch-service.yaml --namespace sysdigcloud
```
To create an Elasticsearch deployment, the provided manifest under `manifests/elasticsearch-deployment.yaml` can be used. By default, it will use a local non-persistent volume (emptyDir), but the manifest contains commented snippets that can be uncommented when using persistent volumes such as AWS EBS or GCE Disks (just replace the volume id from the cloud provider in the snippet):

```
kubectl create -f manifests/deployments/elasticsearch-deployment.yaml --namespace sysdigcloud
```
This creates an Elasticsearch cluster of size 1. To expand the Elasticseatch cluster, a new deployment must be created for each additional Elasticsearch node in the cluster. You can't just scale the replicas of the existing deployment because each Elasticsearch pod must get a different persistent volume, so in that sense Elasticsearch pods are "pets" with unique identities and not "cattles".

In order for the new Elasticsearch deployment to automatically join the cluster, some conventions must be followed. In particular, the Elasticsearch node number (1, 2, 3, ...) must be properly put in the manifest `manifests/elasticsearch-deployment.yaml` under the entries marked as `# Elasticsearch node number`.

For example, to scale the Elasticsearch cluster from 2 to 3 nodes, the manifest can be edited as such:

```
...
metadata:
  name: sysdigcloud-elasticsearch-3 # Elasticsearch node number
...
      labels:
        instance: "3" # Elasticsearch node number
...
```

And then the deployment can be created as usual:

```
kubectl create -f manifests/deployments/elasticsearch-deployment.yaml --namespace sysdigcloud
```
After each scaling activity, the status of the cluster can be checked by executing `curl -sS http://$(hostname -i):9200/_cluster/health?pretty=true` in one of the Elasticsearch pods.

```
$ kubectl --namespace sysdigcloud exec -it sysdigcloud-elasticsearch-1-2660816362-tfht5 -- bash -c 'curl -sS http://$(hostname -i):9200/_cluster/health?pretty=true'
{
  "cluster_name" : "sysdigcloud",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 2,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```