# sdc-kubernetes: Sysdig Pro Software deployment on Kubernetes

[Sysdig](https://sysdig.com/) is the the first unified approach to container security, monitoring and forensics.

This project contains the tools you need to deploy the **Pro Software** (AKA onprem) version of Sysdig to your infrastructure as Kubernetes deployment.


## Recent updates <a id="Recent-updates"></a>

SDC-Kubernetes is an on-prem version of [Sysdig Monitor](https://sysdig.com/product/monitor/), a SAAS offering by Sysdig 
Inc for monitoring containerized and non-containerized environments. 
The official on-prem Kubernetes guide can be found [here](https://github.com/draios/sysdigcloud-kubernetes). 

Here are the most recent updates:

- **Introduction of Statefulsets**
    
    **NOTE**: Kubernetes statefulsets are stable (GA) in version 1.9. Using an earlier version may have adverse affects.

- **Introduction of persistence to datastores**

    Persistent volumes can utilize block disks from the various cloud provider dynamically. The disks can be encrypted, 
    adjusted for IOPS specific performance and can utilize snapshots for backups.
  
## Infrastructure Overview <a id="Infrastructure-Overview"></a>

![sdc-kubernetes](https://github.com/draios/sysdigcloud-kubernetes/raw/master/images/sysdig_cloud_infrastructure.png?raw=true)

###### Backend Components
* api-servers: Provides a web and API interface to the sysdig application
* collectors: Agents connect to the backend via sysdig collectors
* workers: Process data aggregations and alerts

###### Cache Layer
* redis: intra-service cache

###### DataStores
* mysql: stores user data and environmental data
* elasticsearch: stores event and metadata
* cassandra: stores sysdig metrics

Backend components (worker, api and collector) are stateless deployed in deploymentsets.
Datastores (redis, mysql, elasticsearch and cassandra) are stateful.

## Requirements <a id="Requirements"></a>

- Access to a running Kubernetes cluster on AWS or GKE.
- Sysdig Cloud quay.io pull secret
- Sysdig Cloud license
- kubectl installed on your machine and communicating with the Kubernetes cluster
 
## Installation Guide <a id="installation-guide"></a>


### Step 1: Namespace Creation

1. Create a namespace called *sysdigcloud* where all components are deployed.

    ```
    kubectl create namespace sysdigcloud
    ```

### Step 2: Create Config

2. Create Kubernetes secrets and configMaps populated with information about usernames, passwords, ssl certs, 
quay.io pull secret and various application specific parameters.

    ```
    kubectl -n sysdigcloud create -f sysdigcloud/config.yaml
    ```

### Step 3: Quay Pull Secret

To download Sysdig Cloud Docker images it is mandatory to create a Kubernetes pull secret. 
Edit the file `sysdigcloud/pull-secret.yaml` and change the place holder `<PULL_SECRET>` with the provided pull secret.
Create the pull secret object using kubectl:

```
kubectl -n sysdigcloud create -f sysdigcloud/pull-secret.yaml
```

### Step 4: SSL Certificates

Sysdig Cloud api and collector services use SSL to secure the communication between the customer browser 
and sysdigcloud agents.

If you want to use a custom SSL secrets, make sure to obtain the respective `server.crt` 
and `server.key` files, otherwise you can also create a self-signed certificate with:

```
openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 -subj "/C=US/ST=CA/L=SanFrancisco/O=ICT/CN=onprem.sysdigcloud.com" -keyout server.key -out server.crt
```

Once done, create a Kubernetes secret:

```
kubectl -n sysdigcloud create secret tls sysdigcloud-ssl-secret --cert=server.crt --key=server.key
```

##### Optional: Custom SSL Certificates

If you want to use services that implement SSL self-signed certificates you can import those certificates 
and their chains, storing them in PEM format and injecting them as a generic kubernets secret.
For each certificate you want to import create a file, for example: certs1.crt, cert2.crt, ... and then 
the kubernetes secret using the following command line:

```
kubectl -n sysdigcloud create secret generic sysdigcloud-java-certs --from-file=certs1.crt --from-file=certs2.crt
```

### Step 4: Install Components

1. Verify that there is a storage class created already.

    ```
    kubectl get storageclass
    ```
    
    If there is a storage class that is suitable for this install note the name and fill in the `storageClassName` in the following yamls
    
    ```
    datastores/as_kubernetes_pods/manifests/cassandra/cassandra-statefulset.yaml
    datastores/as_kubernetes_pods/manifests/elasticsearch/elasticsearch-statefulset.yaml
    ```
    
    If there is no storage class defined, create one. Below is an example of a manifest for a storage class using GP2 volumes in AWS. Assuming you've saved it as `sysdigcloud-storageclass.yaml`, running `kubectl create -f sysdigcloud-storageclass.yaml` will then create the storage class.

    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: gp2
      annotations:
        storageclass.beta.kubernetes.io/is-default-class: "true"
      labels:
        kubernetes.io/cluster-service: "true"
        addonmanager.kubernetes.io/mode: EnsureExists
    provisioner: kubernetes.io/aws-ebs
    parameters:
      type: gp2
    ```

2. Create the datastore statefulsets (elasticsearch and cassandra). Elasticsearch and Cassandra are 
automatically setup with --replica=3 generating full clusters.  

    ```
    kubectl -n sysdigcloud create -f datastores/as_kubernetes_pods/manifests/cassandra/cassandra-service.yaml
    kubectl -n sysdigcloud create -f datastores/as_kubernetes_pods/manifests/cassandra/cassandra-statefulset.yaml
    kubectl -n sysdigcloud create -f datastores/as_kubernetes_pods/manifests/elasticsearch/elasticsearch-service.yaml
    kubectl -n sysdigcloud create -f datastores/as_kubernetes_pods/manifests/elasticsearch/elasticsearch-statefulset.yaml
    ```

3. Create the datastore deployments (mysql)

    ```
    kubectl -n sysdigcloud create -f datastores/as_kubernetes_pods/manifests/mysql.yaml
    kubectl -n sysdigcloud create -f datastores/as_kubernetes_pods/manifests/redis/redis-deployment.yaml
    ```

4. Deploy the backend Deployment sets (worker, collector and api)
 
    ```
    kubectl -n sysdigcloud create -f sysdigcloud/sdc-api.yaml
    kubectl -n sysdigcloud create -f sysdigcloud/sdc-collector.yaml
    kubectl -n sysdigcloud create -f sysdigcloud/sdc-worker.yaml
    ```

### Step 5: Expose Sysdig Cloud services

To expose the Sysdig Cloud api and collector deployments you can create a Kubernetes NodePort or 
LoadBalacer service, depending on the specific needs.

#### NodePort

Using a NodePort service the Kubernetes master will allocate a port on each node and will proxy that 
port (the same port number on every Node) towards the service.
After this step, it should be possible to correctly fill all the parameters in the ConfigMap, 
such as `collector.endpoint`, `collector.port` and `api.url`.

It is possible to create a NodePort service for Sysdig Cloud api and collector using kubectl and the 
templates in the sysdigcloud directory:

```
kubectl -n sysdigcloud create -f sysdigcloud/api-nodeport-service.yaml
kubectl -n sysdigcloud create -f sysdigcloud/collector-nodeport-service.yaml
```

#### LoadBalancer

On cloud providers which support external load balancers, using a LoadBalancer service will provision a 
load balancer for the service. The actual creation of the load balancer happens asynchronously. Traffic 
from the external load balancer will be directed at the backend pods, though exactly how that works depends 
on the cloud provider.

It is possible to create a LoadBalancer Service for Sysdig Cloud api and collector using kubectl and the 
templates in the sysdigcloud folder:

```
kubectl -n sysdigcloud create -f sysdigcloud/api-loadbalancer-service.yaml
kubectl -n sysdigcloud create -f sysdigcloud/collector-loadbalancer-service.yaml
```

## Confirm Installation  <a id="Confirm-Installation"></a>

Once the installation has been completed, your output should look similar (please note that the below output is an example):
    
    $ kubectl -n sysdigcloud get pods     
    sdc-api-2039094698-11rtd         1/1       Running   0          13m
    sdc-cassandra-0                  1/1       Running   0          12m
    sdc-cassandra-1                  1/1       Running   0          11m
    sdc-cassandra-2                  1/1       Running   0          11m
    sdc-collector-1001165270-chrz0   1/1       Running   0          13m
    sdc-elasticsearch-0              1/1       Running   0          14m
    sdc-elasticsearch-1              1/1       Running   0          14m
    sdc-elasticsearch-2              1/1       Running   0          14m
    sdc-mysql                        1/1       Running   0          14m
    sdc-redis                        1/1       Running   0          14m
    sdc-worker-1937471472-hfp25      1/1       Running   0          13m

    $ kubectl -n sysdigcloud get services
    NAME                CLUSTER-IP   EXTERNAL-IP        PORT(S)                               AGE
    sdc-api             10.3.0.36    ad0d03112c706...   443:32253/TCP                         32m
    sdc-cassandra       None         <none>             9042/TCP,7000/TCP,7001/TCP,7199/TCP   34m
    sdc-collector       10.3.0.203   ad0e5cf87c706...   6443:31063/TCP                        32m
    sdc-elasticsearch   None         <none>             9200/TCP,9300/TCP                     34m
    sdc-mysql           None         <none>             3306/TCP                              34m
    sdc-redis           None         <none>             6379/TCP,16379/TCP                    34m

Describe the sdc-api service to get the full API endpoint URL.
It will be `ad0d03112c70611e79d6006e5a830746-1802392156.us-west-1.elb.amazonaws.com` in this case. Use this URL to 
access the SDC Monitor interface. This URL can be given a sensible URL via Route53 or similar.
(please note that the below output is an example, including the loadBalancer Ingress annotation)

    $ kubectl -n sysdigcloud describe service sdc-api
    Name:            sdc-api
    Namespace:       sysdigcloud
    Labels:          app=sysdigcloud
                     role=api
    Annotations:     <none>
    Selector:        app=sysdigcloud,role=api
    Type:            LoadBalancer
    IP:              10.3.0.36
    LoadBalancer Ingress:    ad0d03112c70611e79d6006e5a830746-1802392156.us-west-1.elb.amazonaws.com
    Port:            secure-api    443/TCP
    NodePort:        secure-api    32253/TCP
    Endpoints:        10.2.79.173:443
    Session Affinity:    None
    Events:
      FirstSeen    LastSeen    Count    From            SubObjectPath    Type        Reason            Message
      ---------    --------    -----    ----            -------------    --------    ------            -------
      33m        33m        1    service-controller            Normal        CreatingLoadBalancer    Creating load balancer
      33m        33m        1    service-controller            Normal        CreatedLoadBalancer     Created load balancer


Describe the sdc-collector service to see the full collector endpoint URL. It will be `ad0e5cf87c70611e79d6006e5a830746-257288196.us-west-1.elb.amazonaws.com`
(please note that the below output is an example, including the loadBalancer Ingress annotation)

    $ kubectl -n sysdigcloud describe service sdc-collector
    Name:            sdc-collector
    Namespace:       sysdigcloud
    Labels:          app=sysdigcloud
                     role=collector
    Annotations:     <none>
    Selector:        app=sysdigcloud,role=collector
    Type:            LoadBalancer
    IP:              10.3.0.203
    LoadBalancer Ingress:    ad0e5cf87c70611e79d6006e5a830746-257288196.us-west-1.elb.amazonaws.com
    Port:            secure-collector    6443/TCP
    NodePort:        secure-collector    31063/TCP
    Endpoints:        10.2.23.211:6443
    Session Affinity:    None
    Events:
      FirstSeen    LastSeen    Count    From            SubObjectPath    Type        Reason            Message
      ---------    --------    -----    ----            -------------    --------    ------            -------
      34m        34m        1    service-controller            Normal        CreatingLoadBalancer    Creating load balancer
      33m        33m        1    service-controller            Normal        CreatedLoadBalancer     Created load balancer


In the above example, go to `https://ad0d03112c70611e79d6006e5a830746-1802392156.us-west-1.elb.amazonaws.com:<port#>` to 
access the main Monitor GUI.
Point your collectors to `ad0e5cf87c70611e79d6006e5a830746-257288196.us-west-1.elb.amazonaws.com`.

#### Version updates <a id="Version-updates"></a>

Sysdig Cloud releases are listed [here](https://github.com/draios/sysdigcloud-kubernetes/releases). Each release has a 
version number (e.g. 925) and specific release notes. 

```
image: quay.io/sysdig/sysdigcloud-backend:658
```
In this case, we are running version 658 of the backend. 

To upgrade to version 925 (the latest), there are two options:

1. Edit the sdc-api, sdc-collector and sdc-worker yaml definitions and add the new image tag `sysdigcloud-backend`
```
image: quay.io/sysdig/sysdigcloud-backend:925
```
Finally, you will need to delete the sdc-api, sdc-collector and sdc-worker pods with the command  
```
kubectl -n sysdigcloud delete pod <pod name>
```

2. You can do a rolling update if downtimes are sensitive.
```
kubectl -n sysdigcloud set image deployment/sdc-api api=quay.io/sysdig/sysdigcloud-backend:893 
kubectl -n sysdigcloud set image deployment/sdc-collector collector=quay.io/sysdig/sysdigcloud-backend:893 
kubectl -n sysdigcloud set image deployment/sdc-worker worker=quay.io/sysdig/sysdigcloud-backend:893 
```
