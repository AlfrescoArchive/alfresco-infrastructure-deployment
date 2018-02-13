# Alfresco Infrastructure Deployment

## Prerequisites

The Alfresco Infrastructre Deployment requires:

| Component        | Recommended version |
| ------------- |:-------------:|
| Docker     | 17.0.9.1 |
| Kubernetes | 1.8.0    |
| Helm       | 2.7.0    |
| Minikube   | 0.24.1   |

Any variation from these technologies and versions may affect the end result. If you do experience any issues please let us know through our [Gitter channel](https://gitter.im/Alfresco/platform-services?utm_source=share-link&utm_medium=link&utm_campaign=share-link).

After you have installed the prerequisites please run the following commands:

```bash
git clone https://github.com/Alfresco/alfresco-infrastructure-deployment.git
cd alfresco-infrastructure-deployment
```

### Kubernetes Cluster

You can choose to deploy the infrastructure to a local kubernetes cluster (illustrated using minikube) or you can choose to deploy to the cloud (illustrated using AWS).
Please check the Anaxes Shipyard documentation on [running a cluster](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/SECRETS.md).

Note the resource requirements:
* Minikube: At least 12 gigs of memory, i.e.:
```bash
minikube start --memory 12000
```
* AWS: A VPC and cluster with 5 nodes. Each node should be a m4.xlarge EC2 instance.

### Helm Tiller

Initialize the Helm Tiller:
```bash
helm init
```

### K8s Cluster Namespace

As mentioned as part of the Anaxes Shipyard guidelines, you should deploy into a separate namespace in the cluster to avoid conflicts (create the namespace only if it does not already exist):
```bash
export DESIREDNAMESPACE=example
kubectl create namespace $DESIREDNAMESPACE
```

This environment variable will be used in the deployment steps.

## Deployment

### 1. Install the nginx-ingress-controller

Install the nginx-ingress-controller into your cluster
```bash
helm install stable/nginx-ingress \
--version 0.8.11 \
--namespace $DESIREDNAMESPACE
```
### 2. Get the nginx-ingress-controller release name from the previous command and set it as a varible:
```bash
export INGRESSRELEASE=knobby-wolf
```

### 3. Wait for the nginx-ingress-controller release to get deployed.  When checking status your pod should be READY 1/1):
```bash
helm status $INGRESSRELEASE
```

### 4. Get the nginx-ingress-controller port for the infrastructure (**NOTE! ONLY FOR MINIKUBE**):
```bash
export INFRAPORT=$(kubectl get service $INGRESSRELEASE-nginx-ingress-controller --namespace $DESIREDNAMESPACE -o jsonpath={.spec.ports[0].nodePort})
```

### 5. Get Minikube or ELB IP and set it as a variable for future use:

```bash
#ON MINIKUBE
export ELBADDRESS=$(minikube ip)

#ON AWS
export ELBADDRESS=$(kubectl get services $INGRESSRELEASE-nginx-ingress-controller --namespace=$DESIREDNAMESPACE -o jsonpath={.status.loadBalancer.ingress[0].hostname})
```

### 6. EFS Storage (**NOTE! ONLY FOR AWS!**)

Create a EFS storage on AWS and make sure it is in the same VPC as your cluster. Make sure you open inbound traffic in the security group to allow NFS traffic. Save the name of the server ex:
```bash
export NFSSERVER=fs-d660549f.efs.us-east-1.amazonaws.com
```

### 7. Deploy the infrastructure charts:
```bash

helm repo add alfresco-incubator https://alfresco.github.io/charts/incubator

helm dependency update alfresco-infrastructure

#ON MINIKUBE
helm install alfresco-infrastructure \
--set alfresco-api-gateway.keycloakURL="http://$ELBADDRESS:$INFRAPORT/auth/" \
--namespace $DESIREDNAMESPACE

#ON AWS
helm install alfresco-infrastructure \
--set alfresco-api-gateway.keycloakURL="http://$ELBADDRESS:$INFRAPORT/auth/" \
--set persistence.volumeEnv=aws \
--set persistence.nfs.server="$NFSSERVER"
--namespace $DESIREDNAMESPACE
```

### 8. Get the infrastructure release name from the previous command and set it as a variable:
```bash
export INFRARELEASE=enervated-deer
```

### 9. Wait for the infrastructure release to get deployed.  When checking status all your pods should be READY 1/1):
```bash
helm status $INFRARELEASE
```

### 10. Teardown:

```bash
helm delete $INGRESSRELEASE
helm delete $INFRARELEASE
kubectl delete namespace $DESIREDNAMESPACE
```
Depending on your cluster type you should be able to also delete it if you want.
For minikube you can just run
```bash
minikube delete
```
For more information on running and tearing down k8s environemnts, follow this [guide](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/docs/running-a-cluster.md).
