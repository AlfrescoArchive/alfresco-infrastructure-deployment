# Alfresco Infrastructure

The Alfresco Infrastructure chart aims at bringing in components that will commonly be used by the majority of applications within the Alfresco Digital Business Platform.

## Introduction

This chart bootstraps the creation of a persistent volume and persistent volume claim on a [Kubernetes](https://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

Beside this it will bring in other shared, common components like the Identity Service. See the [Helm chart requirements](helm/alfresco-infrastructure/requirements.yaml) for the list of additional dependencies brought in.

Check [prerequisites section](https://github.com/Alfresco/alfresco-dbp-deployment/blob/master/README-prerequisite.md) before you start.

### Kubernetes Cluster

Please check the Anaxes Shipyard documentation on [running a cluster](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/SECRETS.md).

### K8s Cluster Namespace

As mentioned as part of the Anaxes Shipyard guidelines, you should deploy into a separate namespace in the cluster to avoid conflicts (create the namespace only if it does not already exist):

```bash
export DESIREDNAMESPACE=example
kubectl create namespace $DESIREDNAMESPACE
```

This environment variable will be used in the deployment steps.

### Amazon EFS Storage (**NOTE! ONLY FOR AWS!**)

Create an EFS storage on AWS and make sure it is in the same VPC as your cluster. Make sure you open inbound traffic in the security group to allow NFS traffic. Save the name of the server as in this example:

```bash
export NFSSERVER=fs-d660549f.efs.us-east-1.amazonaws.com
```

Then install a nfs client service to create a dynamic storage class in kubernetes.
This can be used by multiple deployments.

```bash
helm install stable/nfs-client-provisioner \
--name $DESIREDNAMESPACE \
--set nfs.server="$NFSSERVER" \
--set nfs.path="/" \
--set storageClass.reclaimPolicy="Retain" \
--set storageClass.name="$DESIREDNAMESPACE-sc" \
--namespace $DESIREDNAMESPACE
```

***Note!***
The Persistent volume created with NFS to store the data on the created EFS has the [ReclaimPolicy](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaim-policy) set to Recycle.
This means that by default, when you delete the release the saved data is deleted automatically.

To change this behaviour and keep the data you can set the persistence.reclaimPolicy value to Retain.

## Installing the chart

### 1. Deploy the infrastructure charts:
```bash

helm repo add alfresco-incubator https://kubernetes-charts.alfresco.com/incubator
helm repo add alfresco-stable https://kubernetes-charts.alfresco.com/stable

helm install alfresco-incubator/alfresco-infrastructure \
--set persistence.storageClass.enabled=true \
--set persistence.storageClass.name="$DESIREDNAMESPACE-sc" \
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

## Nginx-ingress Custom Configuration

By default, this chart deploys the [nginx-ingress chart](https://github.com/kubernetes/charts/tree/master/stable/nginx-ingress) with the following configuration that will create an ELB when using AWS and will set a dummy certificate on it:

```yaml
nginx-ingress:
  rbac:
    create: true
  config:
    ssl-redirect: "false"
    server-tokens: "false"
  controller:
    scope:
      enabled: true
```
If you want to customize the certificate type on the ingress level, you can choose one of the options below:

<details>
<summary>Using a self-signed certificate</summary>
<p>

If you want your own certificate set on the ELB created through AWS you should create a secret from your cert files

```bash
kubectl create secret tls certsecret --key /tmp/tls.key --cert /tmp/tls.crt \
  --namespace $DESIREDNAMESPACE
```

Then deploy the infrastructure chart with the following:

```bash
cat <<EOF > infravalues.yaml
#Persistence options
persistence:
  #Enables the creation of a persistent volume
  enabled: true
  efs:
    #Enables EFS ussage
    enabled: false
    #DNS address of EFS
    dns: fs-example.efs.us-east-1.amazonaws.com
    #Base path to use within the EFS that is mounted as a volume
    path: "/"
  #Size allocated to the volume in K8S
  baseSize: 20Gi

nginx-ingress:
  rbac:
    create: true
  controller:
    config:
      ssl-redirect: "false"
      server-tokens: "false"
    scope:
      enabled: true
    publishService:
      enabled: true
    extraArgs:
      default-ssl-certificate: $DESIREDNAMESPACE/certsecret
EOF

helm install alfresco-incubator/alfresco-infrastructure \
-f infravalues.yaml \
--namespace $DESIREDNAMESPACE
```

</p>
</details>

<details>
<summary>Using an AWS generated certificate and Amazon Route 53 zone</summary>
<p>

If you

* created the cluster in AWS using [kops](https://github.com/kubernetes/kops/)
* have a matching SSL/TLS certificate stored in [AWS Certificate Manager](https://aws.amazon.com/certificate-manager/)
* are using a zone in [Amazon Route 53](https://aws.amazon.com/route53/)

Kubernetes' [External DNS](https://github.com/kubernetes-incubator/external-dns)
can autogenerate a DNS entry for you (a CNAME of the generated ELB) and apply
the SSL/TLS certificate to the ELB.

_Note: External DNS is currenty in Alpha Version - June 2018_

_Note: AWS Certificate Manager ARNs are of the form `arn:aws:acm:REGION:ACCOUNT:certificate/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`._

Set `DOMAIN` to the DNS Zone you used when [creating the cluster](https://github.com/kubernetes/kops/blob/master/docs/aws.md#scenario-1b-a-subdomain-under-a-domain-purchasedhosted-via-aws).

```bash
ELB_CNAME="${DESIREDNAMESPACE}.${DOMAIN}"
ELB_CERTIFICATE_ARN=$(aws acm list-certificates | \
  jq '.CertificateSummaryList[] | select (.DomainName == "'${DOMAIN}'") | .CertificateArn')

cat <<EOF > infravalues.yaml
#Persistence options
persistence:
  #Enables the creation of a persistent volume
  enabled: true
  efs:
    #Enables EFS ussage
    enabled: false
    #DNS address of EFS
    dns: fs-example.efs.us-east-1.amazonaws.com
    #Base path to use within the EFS that is mounted as a volume
    path: "/"
  #Size allocated to the volume in K8S
  baseSize: 20Gi

nginx-ingress:
  rbac:
    create: true
  controller:
    config:
      ssl-redirect: "false"
      server-tokens: "false"
    scope:
      enabled: true
    publishService:
      enabled: true
    service:
      targetPorts:
        http: http
        https: http
      annotations:
        external-dns.alpha.kubernetes.io/hostname: ${ELB_CNAME}
        service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
        service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: '3600'
        service.beta.kubernetes.io/aws-load-balancer-ssl-cert: ${ELB_CERTIFICATE_ARN}
        service.beta.kubernetes.io/aws-load-balancer-ssl-ports: https
EOF

helm install alfresco-incubator/alfresco-infrastructure \
-f infravalues.yaml \
--namespace $DESIREDNAMESPACE
```

</p>
</details>
<br/>

For additional information on customizing the nginx-ingress chart please refer to the [nginx-ingress chart Readme](https://github.com/kubernetes/charts/tree/master/stable/nginx-ingress)

***Note!*** Terminating SSL at LB level currently causes invalid redirect issues on identity service level.

## Configuration
The following table lists the configurable parameters of the infrastructure chart and their default values.


| Parameter                                                   | Description                                          | Default                                             |
| --------------------------------------------------------    | --------------------------------------------------   | --------------------------------------------------- |
| `persistence.enabled`                                       | Persistence is enabled for this chart                | `true`                                              |
| `persistence.baseSize`                                      | Size of the persistent volume.                       | `20Gi`                                              |
| `persistence.storageClass.enabled`                          | Use custom Storage Class persistence                 | `false`                                             |
| `persistence.storageClass.name`                             | Storage Class Name                                   | `nfs`                                               |
| `persistence.storageClass.accessModes`                      | Access modes for the volume                          | `[ReadWriteMany]`                                   |
| `alfresco-infrastructure.activemq.enabled`                  | Activemq is enabled for this chart                   | `true`                                              |
| `alfresco-infrastructure.alfresco-identity-service.enabled` | Alfresco Identity Service is enabled for this chart  | `true`                                              |
| `alfresco-infrastructure.nginx-ingress.enabled`             | Nginx-ingress is enabled for this chart              | `true`                                              |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,

```bash
$ helm install --name my-release \
  --set persistence.enabled=true \
    alfresco-incubator/alfresco-infrastructure
```

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```console
$ helm install alfresco-incubator/alfresco-infrastructure --name my-release -f values.yaml
```

## Troubleshooting

**Error: "realm-secret" already exists**
When installing the Infrastructure chart, with the Identity Service enabled, if you recieve the message *Error: release \<release-name\> failed: secrets "realm-secret" already exists* there is an existing realm secret in the namespace you are installing.  This could mean that you are either installing into a namespace with an existing Identity Service or there is a realm secret leftover from a previous installation of the Identity Service.

If the realm secret is leftover from a previous installation it can be removed with the following

```bash
$ kubectl delete secret realm-secret --namespace $DESIREDNAMESPACE
```
