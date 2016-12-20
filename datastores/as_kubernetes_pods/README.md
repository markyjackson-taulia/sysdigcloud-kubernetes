# Datastores as Kubernetes pod

Sysdig Cloud datastores can be deployed as Kubernetes pods. Each pod can be configured to use a local volume (emptyDir) that is only persisted for the lifetime of the pod, or a persistent volume. The provided manifests contain examples for AWS EBS and GCE Disks, for other type of Kubernetes persistent volumes refer to http://kubernetes.io/docs/user-guide/persistent-volumes/#types-of-persistent-volumes

If using persistent volumes, those will need to be created separately before deploying the datastore pods.
If you use AWS you can create a volume using the command line `aws ec2 create-volume` or the AWS console, If you use GCE you can create a volume using the command line `gcloud compute disks create` or the GCE console.

Please notice that, when running a Kubernetes cluster in multiple zones, special care needs to be taken in maintaining the affinity between zones where the persistent volumes are created and zones where the pods are actually scheduled, since most cloud providers impose limitations. At the time of writing, the current solutions are:

- Use persistent volume claims instead of embedding the volumes inside the pod manifest (like in the following examples): http://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims
- Manually force the affinity of the datastore deployments to specific zones using the nodeSelector field: http://kubernetes.io/docs/user-guide/node-selection/

## MySQL

To create a MySQL deployment, the provided manifest under `manifests/mysql.yaml` can be used. By default, it will use a local non-persistent volume (emptyDir), but the manifest contains commented snippets that can be uncommented when using persistent volumes such as AWS EBS or GCE Disks (just replace the volume id from the cloud provider in the snippet):

```
kubectl create -f manifests/mysql.yaml --namespace sysdigcloud
```
## Google Cloud SQL with CloudSQL proxy (Second Generation instances only)

You'll need to create several Secret resources to allow the SQL proxy to connect with your SQL instance. First, you'll need to create a secret resource containing the Service Account credentials to allow the proxy to communicate with the Cloud SQL API.

You would need to enable CloudSQL API in two places
1) Via the API Manager and enabling the "Cloud SQL API”
2) When creating the Container Cluster and under project access and enabling "Cloud SQL"

Run this command, making sure to replace <PATH_TO_CREDENTIAL_FILE> with the correct location of the JSON file of your service account:
```
kubectl create secret generic cloudsql-oauth-credentials --from-file=credentials.json=<PATH_TO_CREDENTIAL_FILE>
```
Change `[INSTANCE_CONNECTION_NAME]` in `manifests/mysql-proxy.yaml`  to include your GCP
project, the region of your Cloud SQL instance and the name
of your Cloud SQL instance. The format is
` -instances=$PROJECT:$REGION:INSTANCE=tcp:0.0.0.0:3306`

Then, run:

kubectl create -f manifests/mysql-proxy.yaml 


## Redis

Redis doesn't require persistent storage, so it can be simply deployed as:

```
kubectl create -f manifests/redis.yaml --namespace sysdigcloud
```

## Cassandra

Before deploying the deployment object, the proper Cassandra headless service must be created (the headless service will be used for service discovery when deploying a multi-node Cassandra cluster):

```
kubectl create -f manifests/cassandra-service.yaml --namespace sysdigcloud
```

To create a Cassandra deployment, the provided manifest under `manifests/cassandra-deployment.yaml` can be used. By default, it will use a local non-persistent volume (emptyDir), but the manifest contains commented snippets that can be uncommented when using persistent volumes such as AWS EBS or GCE Disks (just replace the volume id from the cloud provider in the snippet):

```
kubectl create -f manifests/cassandra-deployment.yaml --namespace sysdigcloud
```

This creates a Cassandra cluster of size 1. To expand the Cassandra cluster, a new deployment must be created for each additional Cassandra node in the cluster. You can't just scale the replicas of the existing deployment because each Cassandra pod must get a different persistent volume, so in that sense Cassandra pods are "pets" with unique identities and not "cattles".

In order for the new Cassandra deployment to automatically join the cluster, some conventions must be followed. In particular, the Cassandra node number (1, 2, 3, ...) must be properly put in the manifest `manifests/cassandra-deployment.yaml` under the entries marked as `# Cassandra node number`.

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
kubectl create -f manifests/cassandra-deployment.yaml --namespace sysdigcloud
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
