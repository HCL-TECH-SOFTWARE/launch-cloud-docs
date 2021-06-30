# HCL Launch Agent Relay - Helm Chart

## Introduction
[HCL Launch](https://www.hcltechsw.com/launch/) is a tool for automating application deployments through your environments. It is designed to facilitate rapid feedback and continuous delivery in agile development while providing the audit trails, versioning and approvals needed in production.

## Chart Details
* This chart deploys a single instance of the HCL Launch agent relay that may be scaled to multiple instances.
* Includes a StatefulSet workload object

## Prerequisites

1. Kubernetes 1.16.0+; kubectl CLI; Helm 3;
  * [Install and setup kubectl CLI](https://kubernetes.io/docs/tasks/tools/)
  * [Install and setup the Helm 3 CLI](https://helm.sh/docs/intro/install/)

2. Image and Helm Chart - The HCL Launch agent image and helm chart can be accessed via the HCL Container Registry (hclcr.io) and public Helm repository.
  * The public Helm chart repository can be accessed at https://hclcr.io/chartrepo/launch-helm and directions for accessing the HCL Launch server chart will be discussed later in this README.
  * Get login credentials to the HCL Container Registry.
    * An imagePullSecret must be created to be able to authenticate and pull images from the HCL Container Registry.  Once this secret has been created you will specify the secret name as the value for the image.secret parameter in the values.yaml you provide to 'helm install ...'  Note: Secrets are namespace scoped, so they must be created in every namespace you plan to install Launch into.  Following is an example command to create an imagePullSecret named 'entitledregistry-secret'.

```
kubectl create secret docker-registry entitledregistry-secret --docker-username=<username> --docker-password=<cli-secret> --docker-server=hclcr.io
```

3. The HCL Launch agent relay must have a HCL Launch server to connect to.

4. Secret - A Kubernetes Secret object must be created to store the HCL Launch server's Codestation authentication token and the password for all keystores used by the product.  The name of the secret you create must be specified in the property 'secret.name' in your values.yaml.

* Through the kubectl CLI, create a Secret object in the target namespace.  Generate the base64 encoded values for the Codestation token and password for all keystores used by the product.

```
echo -n 255b21b7-ca48-4f2e-95c0-048fdbff4197 | base64
MjU1YjIxYjctY2E0OC00ZjJlLTk1YzAtMDQ4ZmRiZmY0MTk3
echo -n 'MyKeystorePassword' | base64
TXlLZXlzdG9yZVBhc3N3b3Jk
```

  * Create a file named secret.yaml with the following contents, using your secret name and base64 encoded values.

```
apiVersion: v1
kind: Secret
metadata:
  name: launch-relay-secrets
type: Opaque
data:
  cspassword: MjU1YjIxYjctY2E0OC00ZjJlLTk1YzAtMDQ4ZmRiZmY0MTk3
  keystorepassword: TXlLZXlzdG9yZVBhc3N3b3Jk
```

  * Create the Secret using oc apply

```
kubectl apply -f ./secret.yaml
```

  * Delete or shred the secret.yaml file.

5. A PersistentVolume that will hold the conf directory for the HCL Launch agent relay is required.  If your cluster supports dynamic volume provisioning you will not need to create a PersistentVolume (PV) or PersistentVolumeClaim (PVC) before installing this chart.  If your cluster does not support dynamic volume provisioning, you will need to either ensure a PV is available or you will need to create one before installing this chart.  You can optionally create the PVC to bind it to a specific PV, or you can let the chart create a PVC and bind to any available PV that meets the required size and storage class.  Sample YAML to create the PV and PVC are provided below.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: launch-relay-conf-vol
  labels:
    volume: launch-relay-conf-vol
spec:
  capacity:
    storage: 10Mi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.17
    path: /volume1/k8/launch-relay-conf
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: launch-relay-conf-volc
spec:
  storageClassName: ""
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 10Mi
  selector:
    matchLabels:
      volume: launch-relay-conf-vol
```
* The following storage options have been tested with HCL Launch

  * IBM Block Storage supports the ReadWriteOnce access mode.  ReadWriteMany is not supported.

  * IBM File Storage supports ReadWriteMany which is required for multiple instances of the HCL Launch agent relay.

* HCL Launch requires non-root access to persistent storage. When using IBM File Storage you need to either use the IBM provided “gid” File storage class with default group ID 65531 or create your own customized storage class to specify a different group ID. Please follow the instructions at https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#cs_storage_nonroot for more details.

6.  If a route or ingress is used to access the WSS port of the HCL Launch server from an HCL Launch agent relay, then port 443 should be specified along with the configured URL to access the proper service port defined for the HCL Launch Server.

### PodSecurityPolicy Requirements

The containerized HCL Launch agent relay works well with restricted security requirements.  No root access is required.

### SecurityContextConstraints Requirements

The containerized HCL Launch agent relay works with the default OpenShift SecurityContextConstraint named 'restricted'.  No root access is required.


## Resources Required
* 200MB of RAM
* 100 millicores CPU

## Installing the Chart

Add the HCL Launch helm chart repository to the local client.
```bash
$ helm repo add hcl-charts http://hclcr.io/chartrepo/launch-helm
```

Get a copy of the values.yaml file from the helm chart so you can update it with values used by the install.
```bash
$ helm inspect values hcl-charts/hcl-launch-relay-prod > myvalues.yaml
```

Edit the file myvalues.yaml to specify the parameter values to use when installing the Launch agent relay instance.  The [configuration](#Configuration) section lists the parameter values that can be set.

To install the chart into namespace 'launchtest' with the release name `my-launch-agentrelay-release` and use the values from myvalues.yaml:

```bash
$ helm install my-launch-agentrelay-release hcl-charts/hcl-launch-relay-prod --namespace launchtest --values myvalues.yaml
```

> **Tip**: List all releases using `helm list`.

## Verifying the Chart
Check the Resources->Agent Relays page of the HCL Launch server UI to verify the agent relay has connected successfully.

## Uninstalling the Chart

To uninstall/delete the `my-launch-agentrelay-release` deployment:

```bash
$ helm delete my-launch-agentrelay-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.


## Configuration

### Parameters

The Helm chart has the following values.

##### Common Parameters

| Qualifier | Parameter  | Definition | Allowed Value |
|---|---|---|---|
| version |  | HCL Launch agent relay product vesion |  |
| image | pullPolicy | Image Pull Policy | Always, Never, or IfNotPresent. Defaults to Always |
|       | secret |  An image pull secret used to authenticate with the image registry | Empty (default) if no authentication is required to access the image registry. |
| license | accept | Set to true to indicate you have read and agree to license agreements | false |
| service | type | Specify type of service | Valid options are ClusterIP, NodePort and LoadBalancer (for clusters that support LoadBalancer). Default is ClusterIP |
| persistence | enabled | Determines if persistent storage will be used to hold the HCL Launch agent relay conf directory contents. This should always be true to preserve agent relay data on container restarts. | Default value "true" |
|             | useDynamicProvisioning | Set to "true" if the cluster supports dynamic storage provisoning | Default value "false" |
|             | fsGroup | The group ID to use to access persistent volumes | Default value "1001" |
| confVolume | name | The base name used when the Persistent Volume and/or Persistent Volume Claim for the HCL Launch agent relay conf directory is created by the chart. | Default value is "conf" |
|            | existingClaimName | The name of an existing Persistent Volume Claim that references the Persistent Volume that will be used to hold the HCL Launch agent relay conf directory. |  |
|            | storageClassName | The name of the storage class to use when persistence.useDynamicProvisioning is set to "true". |  |
|            | size | Size of the volume to hold the HCL Launch agent relay conf directory |  |
|              | accessMode | Persistent storage access mode for the ext-lib persistent volume. | ReadWriteOnce |
| serverHostPort |  | HCL Launch server hostname and WSS port in the form hostname:port. If specifying failover info, separate multiple hostname:port with a comma. For example, launch1.example.com:7919,launch2.example.com:7919) |  |
| secret | name | Kubernetes secret which defines required Launch agent relay passwords. | You may leave this blank to use default name of HelmReleaseName-secrets where HelmReleaseName is the name of your Helm Release, otherwise specify the secret name here. |
| codeStationReplication | enabled | Specify true to enable artifact caching on the relay. | false |
|                        | persisted | Specify true to persist the artifact cache when the relay container is restarted. | true |
|                        | serverUrl | The full URL of the central server to connect to, such as https://myserver.example.com:8443. |  |
|                        | maxCacheSize | The size to which to limit the artifact cache, such as 500M for 500 MB or 5G for 5 GB. To not put a limit on the cache, specify none. |  |
|                        | geotags | If you choose to cache files on the relay, you can specify one or more component version statuses here, separated by semicolons. The agent relay automatically caches component versions with any of these statuses so that those versions are ready when they are needed for a deployment. A status can contain a space except in the first or last position. A status can contain commas. The special * status replicates all artifacts, but use this status with caution, because it can make the agent relay store a large amount of data. If no value is specified, no component versions are cached automatically. |  |
| ingress | httpproxyhost | Host name used to access the HCL Launch agent relay http proxy port. Leave blank on OpenShift to create default route. |  |
|               | codestationhost | Host name used to access the HCL Launch agent relay codestation port. Leave blank on OpenShift to create default route. |  |
| resources | constraints.enabled | Specifies whether the resource constraints specified in this helm chart are enabled.   | false (default) or true  |
|           | limits.cpu  | Describes the maximum amount of CPU allowed | Default is 4000m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu)  |
|           | limits.memory | Describes the maximum amount of memory allowed | Default is 4Gi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |
|           | requests.cpu  | Describes the minimum amount of CPU required - if not specified will default to limit (if specified) or otherwise implementation-defined value. | Default is 100m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu) |
|           | requests.memory | Describes the minimum amount of memory required If not specified, the memory amount will default to the limit (if specified) or the implementation-defined value | Default is 200Mi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |

## Scaling
To increase or decrease the number of HCL Launch agent relay instances/replicas issue the following command:

```bash
$ kubectl scale --replicas=2 statefulset/releaseName-hcl-launch-relay-prod
```

## Storage
See the Prerequisites section of this page for storage information.

## Limitations
