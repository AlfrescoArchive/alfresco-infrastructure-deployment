# Alfresco Infrastructure

The Alfresco Infrastructure chart aims at bringing in components that will commonly be used by the majority of applications within the Digital Business Platforms.

# Introduction

This chart bootstraps the creation of a persistent volume and persistent volume claim on a [Kubernetes](http://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

## Prerequisites


| Component        | Recommended version |
| ------------- |:-------------:|
| Docker     | 17.0.9.1 |
| Kubernetes | 1.8.4    |
| Helm       | 2.8.2    |

Any variation from these technologies and versions may affect the end result. If you do experience any issues please let us know through our [Gitter channel](https://gitter.im/Alfresco/platform-services?utm_source=share-link&utm_medium=link&utm_campaign=share-link).

### Kubernetes Cluster

Please check the Anaxes Shipyard documentation on [running a cluster](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/SECRETS.md).


### K8s Cluster Namespace

As mentioned as part of the Anaxes Shipyard guidelines, you should deploy into a separate namespace in the cluster to avoid conflicts (create the namespace only if it does not already exist):
```bash
export DESIREDNAMESPACE=example
kubectl create namespace $DESIREDNAMESPACE
```

This environment variable will be used in the deployment steps.

### Amazon EFS storage (Optional)

Create a EFS storage on AWS and make sure it is in the same VPC as your cluster. Make sure you open inbound traffic in the security group to allow NFS traffic. Save the name of the server ex:
```bash
export NFSSERVER=fs-d660549f.efs.us-east-1.amazonaws.com
```

***Note!***
The Persistent volume created with NFS to store the data on the created EFS has the ReclaimPolicy set to Recycle.
This means that by default, when you delete the release the saved data is deleted automatically.

To change this behaviour and keep the data you can set the persistence.reclaimPolicy value to Retain.

For more Information on Reclaim Policies checkout the official K8S documentation here -> https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaim-policy


## Installing the chart

### 1. Deploy the infrastructure charts:
```bash

helm repo add alfresco-incubator http://kubernetes-charts.alfresco.com/incubator


helm install alfresco-incubator/alfresco-infrastructure \
--set persistence.efs.enabled=true \
--set persistence.efs.dns="$NFSSERVER" \
--namespace $DESIREDNAMESPACE
```

### 2. Get the infrastructure release name from the previous command and set it as a variable:
```bash
export INFRARELEASE=enervated-deer
```

### 3. Wait for the infrastructure release to get deployed. (When checking status all your pods should be READY 1/1):
```bash
helm status $INFRARELEASE
```

### 4. Teardown:

```bash
helm delete --purge $INGRESSRELEASE
helm delete --purge $INFRARELEASE
kubectl delete namespace $DESIREDNAMESPACE
```

For more information on running and tearing down k8s environments, follow this [guide](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/docs/running-a-cluster.md).


## Configuration
The following table lists the configurable parameters of the infrastructure chart and their default values.


| Parameter                  | Description                                     | Default                                                    |
| -----------------------    | ---------------------------------------------   | ---------------------------------------------------------- |
| `persistence.enabled`      | Persistence is enabled for this chart           | `true`                                                     |
| `persistence.baseSize`     | Size of the persistent volume.                  | `20Gi`                                                     |
| `persistence.reclaimPolicy`| Policy for keeping or removing the data after helm delete. Use Retain to keep the data. | `Recycle`          |
| `persistence.efs.enabled`  | Use efs persistence.                            | `false`                                                    |
| `persistence.efs.dns`      | Elastic File System DNS address                 | `none`                                                     |
| `persistence.efs.path`     | Path into the EFS mount to be used.             | `/`                                                         |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,

```bash
$ helm install --name my-release \
  --set persistence.efs.enabled=true \
    alfresco-incubator/alfresco-infrastructure
```

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```console
$ helm install alfresco-incubator/alfresco-infrastructure --name my-release -f values.yaml
```
