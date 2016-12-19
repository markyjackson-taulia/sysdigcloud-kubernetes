# Datastores as external services

It is possible to use one or more external datastore services that are not managed by Kubernetes. Simply edit the configuration ConfigMap and set the proper options for:

```
mysql.endpoint: <DNS/IP>
cassandra.endpoint: <DNS/IP>
redis.endpoint: <DNS/IP>
```

#### MySQL notes:
MySQL service requires a pre-existing schema named `draios` and character-set and collation configured to UTF-8
`CREATE SCHEMA draios DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_unicode_ci;
`
If you are planning to use `Google Cloud SQL second generation` please see 
https://github.com/draios/sysdigcloud-kubernetes/tree/master/datastores/as_kubernetes_pods
