# HCL Launch - Helm Chart

## Introduction

[HCL Launch](https://www.hcltechsw.com/launch/) is a tool for automating application deployments through your environments. It is designed to facilitate rapid feedback and continuous delivery in agile development while providing the audit trails, versioning and approvals needed in production.

## Chart Details

* This chart deploys a single server instance of HCL Launch that may be scaled to multiple instances.
* Includes two statefulSet workload objects, one for server instances and one for distributed front end instances, and corresponding services for them.

## Prerequisites

1. Kubernetes 1.16.0+; kubectl CLI; Helm 3;
  * [Install and setup kubectl CLI](https://kubernetes.io/docs/tasks/tools/)
  * [Install and setup the Helm 3 CLI](https://helm.sh/docs/intro/install/)

2. Image and Helm Chart - The HCL Launch server image and helm chart can be accessed via the HCL Container Registry (hclcr.io) and public Helm repository.
  * The public Helm chart repository can be accessed at https://hclcr.io/chartrepo/launch-helm and directions for accessing the HCL Launch server chart will be discussed later in this README.
  * Get login credentials to the HCL Container Registry.
    * An imagePullSecret must be created to be able to authenticate and pull images from the HCL Container Registry.  Once this secret has been created you will specify the secret name as the value for the image.secret parameter in the values.yaml you provide to 'helm install ...'  Note: Secrets are namespace scoped, so they must be created in every namespace you plan to install Launch into.  Following is an example command to create an imagePullSecret named 'entitledregistry-secret'.

```
kubectl create secret docker-registry entitledregistry-secret --docker-username=<username> --docker-password=<cli-secret> --docker-server=hclcr.io
```

3. Database - HCL Launch requires a database.  The database may be running in your cluster or on hardware that resides outside of your cluster.  This database  must be configured as described in [Installing the server database](https://devops.hcldoc.com/launch/7.1.2/#com.udeploy.install.doc/topics/DBinstall/) before installing the containerized HCL Launch server.  The values used to connect to the database are required when installing the HCL Launch server.  The Apache Derby database type is not supported when running the HCL Launch server in a Kubernetes cluster.

4. Secret - A Kubernetes Secret object must be created to store the initial HCL Launch server administrator password, the password used to access the database mentioned above, and the password for all keystores used by the HCL Launch server.  The name of the secret you create must be specified in the property 'secret.name' in your values.yaml when installing via Helm chart.

* Through the kubectl CLI, create a Secret object in the target namespace.
    Generate the base64 encoded values for the initial Launch admin password, database password, and the password for all keystores used by the product.

```
echo -n 'admin' | base64
YWRtaW4=
echo -n 'MyDbpassword' | base64
TXlEYnBhc3N3b3Jk
echo -n 'MyKeystorePassword' | base64
TXlLZXlzdG9yZVBhc3N3b3Jk
```

  * Create a file named secret.yaml with the following contents, using your secret name and base64 encoded values.

```
apiVersion: v1
kind: Secret
metadata:
  name: launch-secrets
type: Opaque
data:
  initpassword: YWRtaW4=
  dbpassword: TXlEYnBhc3N3b3Jk
  keystorepassword: TXlLZXlzdG9yZVBhc3N3b3Jk
```

  * Create the Secret using kubectl apply

```
kubectl apply -f ./secret.yaml
```

  * Delete or shred the secret.yaml file.

5. JDBC drivers - A PersistentVolume (PV) that contains the JDBC driver(s) required to connect to the database configured above must be created.  You must either:

* Create Persistence Storage Volume - Create a PV, copy the JDBC driver(s) to the PV, and create a PersistentVolumeClaim (PVC) that is bound to the PV. For more information on Persistent Volumes and Persistent Volume Claims, see the [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes). Sample YAML to create the PV and PVC are provided below.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: launch-ext-lib
  labels:
    volume: launch-ext-lib-vol
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.17
    path: /volume1/k8/launch-ext-lib
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: launch-ext-lib-volc
spec:
  storageClassName: ""
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 100Mi
  selector:
    matchLabels:
      volume: launch-ext-lib-vol
```
* Dynamic Volume Provisioning - If your cluster supports [dynamic volume provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/), you may use it to create the PV and PVC. However, the JDBC drivers will still need to be copied to the PV. To copy the JDBC driver(s) to your PV during the chart installation process, first write a bash script that copies the JDBC driver(s) from a location accessible from your cluster to `${UCD_HOME}/ext_lib/`. Next, store the script, named `script.sh`, in a yaml file describing a [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).  Finally, create the ConfigMap in your cluster by running a command such as `kubectl create configmap <map-name> <data-source>`.  Below is an example ConfigMap yaml file that copies a MySQL .jar file from a web server using wget.

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: user-script
data:
  script.sh: |
    #!/bin/bash
    echo "Running script.sh..."
    if [ ! -f ${UCD_HOME}/ext_lib/mysql-jdbc.jar ] ; then
      echo "Copying file(s)..."    
      wget -L -O mysql-jdbc.jar http://webserver-example/mysql-jdbc.jar
      mv mysql-jdbc.jar ${UCD_HOME}/ext_lib/
      echo "Done copying."
    else
      echo "File ${UCD_HOME}/ext_lib/mysql-jdbc.jar already exists."
    fi
```
  * Note the script must be named `script.sh`.

6. A PersistentVolume that will hold the appdata directory for the HCL Launch server is required.  If your cluster supports dynamic volume provisioning you will not need to manually create a PersistentVolume (PV) or PersistentVolumeClaim (PVC) before installing this chart.  If your cluster does not support dynamic volume provisioning, you will need to either ensure a PV is available or you will need to create one before installing this chart.  You can optionally create the PVC to bind it to a specific PV, or you can let the chart create a PVC and bind to any available PV that meets the required size and storage class.  Sample YAML to create the PV and PVC are provided below.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: launch-appdata-vol
  labels:
    volume: launch-appdata-vol
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.17
    path: /volume1/k8/launch-appdata
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: launch-appdata-volc
spec:
  storageClassName: ""
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 20Gi
  selector:
    matchLabels:
      volume: launch-appdata-vol
```
  * The following storage options have been tested with HCL Launch

    * IBM Block Storage supports the ReadWriteOnce access mode.  ReadWriteMany is not supported.

    * IBM File Storage supports ReadWriteMany which is required for Distributed Front End(DFE).

  * HCL Launch requires non-root access to persistent storage. When using IBM File Storage you need to either use the IBM provided “gid” File storage class with default group ID 65531 or create your own customized storage class to specify a different group ID. Please follow the instructions at https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#cs_storage_nonroot for more details.

7.  If a route or ingress is used to access the WSS port of the HCL Launch server from an HCL Launch agent, then port 443 should be specified along with the configured URL to access the proper service port defined for the HCL Launch server.

### PodSecurityPolicy Requirements

The containerized HCL Launch server works well with restricted security requirements.  No root access is required.

### SecurityContextConstraints Requirements

The containerized HCL Launch server works with the default OpenShift SecurityContextConstraint named 'restricted'.  No root access is required.

## Resources Required

* 4GB of RAM, plus 4MB of RAM for each agent
* 2 CPU cores, plus 2 cores for each 500 agents

## Installing the Chart

Add the HCL Launch helm chart repository to the local client.
```bash
$ helm repo add hcl-charts http://hclcr.io/chartrepo/launch-helm
```

Get a copy of the values.yaml file from the helm chart so you can update it with values used by the install.
```bash
$ helm inspect values hcl-charts/hcl-launch-server-prod > myvalues.yaml
```

Edit the file myvalues.yaml to specify the parameter values to use when installing the Launch server instance.  The [configuration](#Configuration) section lists the parameter values that can be set.

To install the chart into namespace 'launchtest' with the release name `my-launch-release` and use the values from myvalues.yaml:

```bash
$ helm install my-launch-release hcl-charts/hcl-launch-server-prod --namespace launchtest --values myvalues.yaml
```

> **Tip**: List all releases using `helm list`.

## Verifying the Chart

See the instructions (from NOTES.txt within chart) after the helm installation completes for chart verification. The instruction can also be viewed by running the command: helm status my-launch-release.

## Upgrading the Chart



## Uninstalling the Chart

To uninstall/delete the `my-launch-release` release:

```bash
$ helm delete my-launch-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.


## Configuration

### Parameters

The Helm chart has the following values.

##### Common Parameters

| Qualifier | Parameter  | Definition | Allowed Value |
|---|---|---|---|
| version |  | HCL Launch product version |  |
| replicas | server | Number of HCL Launch server replicas | Non-zero number of replicas.  Defaults to 1 |
|          | dfe | Number of DFE replicas | Number of Distributed Front End replicas.  Defaults to 0 |
| image | pullPolicy | Image Pull Policy | Always, Never, or IfNotPresent. Defaults to Always |
|       | secret |  An image pull secret used to authenticate with the image registry | Empty (default) if no authentication is required to access the image registry. |
| service | type | Specify type of service | Valid options are ClusterIP, NodePort and LoadBalancer (for clusters that support LoadBalancer). Default is ClusterIP |
| database | type | The type of database Launch will connect to | Valid values are db2, mysql, oracle, and sqlserver |
|          | name | The name of the database to use |  |
|          | hostname | The hostname/IP of the database server | |
|          | port | The database port to connect to | |
|          | username | The user to access the database with | |
|          | jdbcConnUrl | The JDBC Connection URL used to connect to the database used by the Launch server. This value is normally constructed using the database type and other database field values, but must be specified here when using Oracle RAC/ORAAS or SQL Server with Integrated Security. | |
| secureConnections  | required | Specify whether Launch server connections are required to be secure | Default value is "true" |
| secret | name | Kubernetes secret which defines required Launch passwords. | You may leave this blank to use default name of HelmReleaseName-secrets where HelmReleaseName is the name of your Helm Release, otherwise specify the secret name here. |
| license | accept | Set to true to indicate you have read and agree to license agreements | false |
|  | serverURL | Information required to connect to the Launch license server. | Empty (default) to begin a 60-day evaluation license period.|
| persistence | enabled | Determines if persistent storage will be used to hold the Launch server appdata directory contents. This should always be true to preserve server data on container restarts. | Default value "true" |
|             | useDynamicProvisioning | Set to "true" if the cluster supports dynamic storage provisioning | Default value "false" |
|             | fsGroup | The group ID to use to access persistent volumes | Default value "1001" |
| extLibVolume | name | The base name used when the Persistent Volume and/or Persistent Volume Claim for the extlib directory is created by the chart. | Default value is "ext-lib" |
|              | storageClassName | The name of the storage class to use when persistence.useDynamicProvisioning is set to "true". |  |
|              | size | Size of the volume used to hold the JDBC driver .jar files |  |
|              | existingClaimName | Persistent volume claim name for the volume that contains the JDBC driver file(s) used to connect to the Launch database. |  |
|              | configMapName | Name of an existing ConfigMap which contains a script named script.sh. This script is run before HCL Launch server installation and is useful for copying database driver .jars to the ext-lib persistent volume. |  |
|              | accessMode | Persistent storage access mode for the ext-lib persistent volume. | ReadWriteOnce |
| appDataVolume | name | The base name used when the Persistent Volume and/or Persistent Volume Claim for the Launch server appdata directory is created by the chart. | Default value is "appdata" |
|               | existingClaimName | The name of an existing Persistent Volume Claim that references the Persistent Volume that will be used to hold the Launch server appdata directory. |  |
|               | storageClassName | The name of the storage class to use when persistence.useDynamicProvisioning is set to "true". |  |
|               | size | Size of the volume to hold the Launch server appdata directory |  |
|              | accessMode | Persistent storage access mode for the appdata persistent volume. | ReadWriteOnce |
| ingress | host | Host name used to access the Launch server UI. Leave blank on OpenShift to create default route. |  |
|               | dfehost | Host name used to access the Launch server distributed front end (DFE) UI. Leave blank on OpenShift to create default route. |  |
|               | wsshost | Host name used to access the Launch server WSS port. Leave blank on OpenShift to create default route. |  |
| resources | constraints.enabled | Specifies whether the resource constraints specified in this helm chart are enabled.   | true (default) or false  |
|           | limits.cpu  | Describes the maximum amount of CPU allowed | Default is 4000m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu)  |
|           | limits.memory | Describes the maximum amount of memory allowed | Default is 8Gi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |
|           | requests.cpu  | Describes the minimum amount of CPU required - if not specified will default to limit (if specified) or otherwise implementation-defined value. | Default is 200m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu) |
|           | requests.memory | Describes the minimum amount of memory required If not specified, the memory amount will default to the limit (if specified) or the implementation-defined value | Default is 600Mi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |
|      readinessProbe     | initialDelaySeconds | Number of seconds after the container has started before the readiness probe is initiated | Default is 30 |
|           | periodSeconds | How often (in seconds) to perform the readiness probe | Default is 30 |
|           | failureThreshold | When a Pod starts and the probe fails, Kubernetes will try this number times before giving up. In the case of the readiness probe, the Pod will be marked Unready. | Default is 10 |
|      livenessProbe     | initialDelaySeconds | Number of seconds after the container has started before the liveness probe is initiated | Default is 300 |
|           | periodSeconds | How often (in seconds) to perform the liveness probe | Default is 300 |
|           | failureThreshold | When a Pod starts and the probe fails, Kubernetes will try this number times before giving up. Giving up in the case of the liveness probe means restarting the Pod. | Default is 3 |

## Storage
See the Prerequisites section of this page for storage information.

## Limitations

The Apache Derby database type is not supported when running the HCL Launch server in a Kubernetes cluster. This is because the containerized version is running in Launch HA mode, which does not support Derby.
