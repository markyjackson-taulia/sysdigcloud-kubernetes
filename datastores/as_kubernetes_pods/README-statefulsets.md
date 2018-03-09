# Datastores as Kubernetes pod as StatefulSets
This part of the installation guide has been tested on kubernetes v1.8.7. 

Sysdig Cloud datastores can be deployed as Kubernetes pods. The provided manifests contain examples for AWS EBS and GCE Disks, for other type of Kubernetes persistent volumes refer to the official documentation [here](http://kubernetes.io/docs/user-guide/persistent-volumes/#types-of-persistent-volumes).

This guide assumes that your cluster is configured to dynamically provision PersistentVolumes. If your cluster is not configured to do so, you will have to manually provision the volumes.

Statefulsets in this guide will use VolumeClaims with an associated storage class, we are prodiving example of storage classes for AWS and GCE please refer to the official documentation for more info and examples.

#### Create a custom storage class:
###### AWS
```
kubectl create -f manifests/statefulsets/aws-storage-class.yaml --namespace sysdigcloud
```
###### GCE
```
kubectl create -f manifests/statefulsets/gce-storage-class.yaml --namespace sysdigcloud
```

## MySQL

To create a MySQL statefulset, the provided manifest under `manifests/statefulset/mysql-statefulset.yaml` can be used.

**Before creating the statefulset please customized the manifest choosing the volume provider to use.**

Create MySQL service:
```
kubectl create -f manifests/services/mysql-service.yaml --namespace sysdigcloud
```

Create MySQL database:
```
kubectl create -f manifests/statefulsets/mysql-statefulset.yaml --namespace sysdigcloud
```

## Redis

Redis doesn't require persistent storage, so it can be simply deployed with:
```
kubectl create -f manifests/services/redis-service.yaml --namespace sysdigcloud
kubectl create -f manifests/statefulsets/redis-statefulset.yaml --namespace sysdigcloud
```

## Cassandra

Before deploying the statefulset object, the proper Cassandra headless service must be created (the headless service will be used for service discovery when deploying a multi-node Cassandra cluster):

```
kubectl create -f manifests/services/cassandra-service.yaml --namespace sysdigcloud
```

To create a Cassandra statefulset, the provided manifest under `manifests/statefulset/cassandra-statefulset.yaml` can be used.

**Before creating the statefulset please customized the manifest choosing the volume provider to use.**

```
kubectl create -f manifests/statefulsets/cassandra-statefulset.yaml --namespace sysdigcloud
```

This creates a Cassandra cluster of size 1.

To expand the Cassandra cluster, it is possible use kubectl:
```
kubectl --namespace sysdigcloud scale statefulset sysdigcloud-cassandra --replicas=N 	#where N is the number od desiderate cassandra nodes.
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

Maintaining a multi-node production Cassandra cluster requires some simple but mandatory housekeeping procedures, best described in the official documentation.

## Elasticsearch

Before deploying the statefulset object, the proper Elasticsearch headless service must be created (the headless service will be used for service discovery when deploying a multi-node Elasticsearch cluster):

```
kubectl create -f manifests/services/elasticsearch-service.yaml --namespace sysdigcloud
```
To create an Elasticsearch statefulset, the provided manifest under `manifests/statefulsets/elasticsearch-statefulset.yaml` can be used.

**Before creating the statefulset please customized the manifest choosing the volume provider to use.**

```
kubectl create -f manifests/statefulsets/elasticsearch-statefulset.yaml --namespace sysdigcloud
```
This creates an Elasticsearch cluster of size 1.

To expand the Elasticsearch cluster, it is possible use kubectl:
```
kubectl --namespace sysdigcloud scale statefulset sysdigcloud-elasticsearch --replicas=N 	#where N is the number od desiderate elasticsearch nodes.
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