# DevOps Deploy Agent - Helm Chart

## Introduction
[DevOps Deploy](https://www.hcltechsw.com/launch/) is a tool for automating application deployments through your environments. It is designed to facilitate rapid feedback and continuous delivery in agile development while providing the audit trails, versioning and approvals needed in production.


## Chart Details
* This chart deploys a single instance of the DevOps Deploy agent that may be scaled to multiple instances.
* Includes a StatefulSet workload object
* **NOTE:**  Helm Chart versions are independent from DevOps Deploy Product Versions.  One of the values that you will specify in the values.yaml file used during deployment of the product will be the version of DevOps Deploy Agent that that you wish to install.  The best practice is to use the latest available Helm chart and then select the product version that you wish to deploy.

## Prerequisites

1. Kubernetes 1.19.0+; kubectl CLI; Helm 3;
  * [Install and setup kubectl CLI](https://kubernetes.io/docs/tasks/tools/)
  * [Install and setup the Helm 3 CLI](https://helm.sh/docs/intro/install/)

2. Image and Helm Chart - The DevOps Deploy agent image and helm chart can be accessed via the HCL Container Registry (hclcr.io) and public Helm repository.
  * The public Helm chart repository can be accessed at https://hclcr.io/harbor/projects/23/repositories/hcl-launch-agent-prod and directions for accessing the DevOps Deploy server chart will be discussed later in this README.
  * Get login credentials to the HCL Container Registry.
    * An imagePullSecret must be created to be able to authenticate and pull images from the HCL Container Registry.  Once this secret has been created you will specify the secret name as the value for the image.secret parameter in the values.yaml you provide to 'helm install ...'  Note: Secrets are namespace scoped, so they must be created in every namespace you plan to install DevOps Deploy agent into.  Following is an example command to create an imagePullSecret named 'entitledregistry-secret'.

```
kubectl create secret docker-registry entitledregistry-secret --docker-username=<username> --docker-password=<cli-secret> --docker-server=hclcr.io
```

3. The agent must have an DevOps Deploy server or relay to connect to.

4. Secret - A Kubernetes Secret object must be created to store the password for all keystores used by the product.  The name of the secret you create must be specified in the property 'secret.name' in your values.yaml.

  * Through the kubectl CLI, create a Secret object in the target namespace.  Generate the base64 encoded value for the password for all keystores used by the product.

```
echo -n 'MyKeystorePassword' | base64
TXlLZXlzdG9yZVBhc3N3b3Jk
```

  * Create a file named secret.yaml with the following contents, using your Helm Release name and base64 encoded values.

```
apiVersion: v1
kind: Secret
metadata:
  name: launch-agent-secrets
type: Opaque
data:
  keystorepassword: TXlLZXlzdG9yZVBhc3N3b3Jk
```

  * Create the Secret using oc apply

```
kubectl apply -f ./secret.yaml
```

  * Delete or shred the secret.yaml file.

5. A PersistentVolume that will hold the conf directory for the DevOps Deploy agent is required.  If your cluster supports dynamic volume provisioning you will not need to create a PersistentVolume (PV) or PersistentVolumeClaim (PVC) before installing this chart.  If your cluster does not support dynamic volume provisioning, you will need to either ensure a PV is available or you will need to create one before installing this chart.  You can optionally create the PVC to bind it to a specific PV, or you can let the chart create a PVC and bind to any available PV that meets the required size and storage class.  Sample YAML to create the PV and PVC are provided below.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: devopsdeploy-agent-conf-vol
  labels:
    volume: devopsdeploy-agent-conf-vol
spec:
  capacity:
    storage: 10Mi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.17
    path: /volume1/k8/devopsdeploy-agent-conf
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: devopsdeploy-agent-conf-volc
spec:
  storageClassName: ""
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 10Mi
  selector:
    matchLabels:
      volume: devopsdeploy-agent-conf-vol
```
  * The following storage options have been tested with DevOps Deploy

    * IBM Block Storage supports the ReadWriteOnce access mode.  ReadWriteMany is not supported.

    * IBM File Storage supports ReadWriteMany which is required for multiple instances of the DevOps Deploy agent.

  * DevOps Deploy requires non-root access to persistent storage. When using IBM File Storage you need to either use the IBM provided “gid” File storage class with default group ID 65531 or create your own customized storage class to specify a different group ID. Please follow the instructions at https://cloud.ibm.com/docs/containers?topic=containers-cs_troubleshoot_storage#cs_storage_nonroot for more details.

  6.  If a route or ingress is used to access the WSS port of the DevOps Deploy server from an DevOps Deploy agent, then port 443 should be specified along with the configured URL to access the proper service port defined for the DevOps Deploy Server.

### PodSecurityPolicy Requirements

The containerized DevOps Deploy agent works well with restricted security requirements.  No root access is required.

### SecurityContextConstraints Requirements

The containerized DevOps Deploy agent works with the default OpenShift SecurityContextConstraint named 'restricted'.  No root access is required.

## Resources Required
* 200MB of RAM
* 50 millicores CPU

## Installing the Chart

Get a copy of the values.yaml file from the helm chart so you can update it with values used by the install.

```bash
$ helm inspect values oci://hclcr.io/launch-helm/hcl-launch-agent-prod > myvalues.yaml
```

Edit the file myvalues.yaml to specify the parameter values to use when installing the DevOps Deploy agent instance.  The [configuration](#Configuration) section lists the parameter values that can be set.

To install the chart into namespace 'devopsdeploytest' with the release name `my-devops-deploy-agent-release` and use the values from myvalues.yaml:

```bash
$ helm install my-devops-deploy-agent-release oci://hclcr.io/launch-helm/hcl-launch-agent-prod --namespace devopsdeploytest --values myvalues.yaml
```

> **Tip**: List all releases using `helm list`.

## Verifying the Chart
Check the Resources->Agents page of the DevOps Deploy server UI to verify the agent has connected successfully.

## Uninstalling the Chart

To uninstall/delete the `my-devops-deploy-agent-release` deployment:

```bash
$ helm delete my-devops-deploy-agent-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.


## Configuration

### Parameters

The Helm chart has the following values that can be overriden using the --set parameter or specified via -f my_values.yaml.

##### Common Parameters

| Qualifier | Parameter  | Definition | Allowed Value |
|---|---|---|---|
| version |  | DevOps Deploy agent product version |  |
| replicas | agent | Number of DevOps Deploy agent replicas | Non-zero number of replicas.  Defaults to 1 |
| image | pullPolicy | Image Pull Policy | Always, Never, or IfNotPresent. Defaults to Always |
|       | secret |  An image pull secret used to authenticate with the image registry | Empty (default) if no authentication is required to access the image registry. |
| license | accept | Set to true to indicate you have read and agree to license agreements | false |
| persistence | enabled | Determines if persistent storage will be used to hold the DevOps Deploy agent conf directory contents. This should always be true to preserve agent data on container restarts. | Default value "true" |
|             | useDynamicProvisioning | Set to "true" if the cluster supports dynamic storage provisoning | Default value "true" |
|             | fsGroup | The group ID to use to access persistent volumes | Default value "1001" |
| confVolume | name | The base name used when the Persistent Volume and/or Persistent Volume Claim for the DevOps Deploy agent conf directory is created by the chart. | Default value is "conf" |
|               | existingClaimName | The name of an existing Persistent Volume Claim that references the Persistent Volume that will be used to hold the DevOps Deploy agent conf directory. |  |
|               | storageClassName | The name of the storage class to use when persistence.useDynamicProvisioning is set to "true". |  |
|               | size | Size of the volume to hold the DevOps Deploy agent conf directory |  |
|              | accessMode | Persistent storage access mode for the conf directory persistent volume. | ReadWriteOnce |
| relayUri |  | Agent Relay Proxy URI if the agent is connecting to a relay. If multiple relays are specified, separate them with commas. For example, random:(http://relay1:20080,http://relay2:20080) |  |
| codestationUrl |  | Agent Relay Codestation URL. If multiple relays are specified, separate them with commas. For example, random:(https://relay1:20081,https://relay2:20081) |  |
| serverUri |  | DevOps Deploy server URI. If multiple servers are specified, separate them with commas. For example, random:(wss://devopsdeploy1.example.com:7919,wss://devopsdeploy2.example.com:7919) |  |
| secret | name | Kubernetes secret which defines password to use when creating keystores. | |
| agentTeams |  | Teams to add this agent to when it connects to the DevOps Deploy server.Format is <team>:<type>. Multiple team specifications are separated with a comma. |  |
| userUtils | existingClaimName | Name of existing Persistent Volume Claim that refers to Persistent Volume that contains executables for the agent process to execute as part of deployment processes. | |
|  | executablesPath | Relative pathname to the directory containing the user provided executable(s).  Comma separate multiple directory paths. | Default is '.', the top-level directory of the PV. |
| resources | constraints.enabled | Specifies whether the resource constraints specified in this helm chart are enabled.   | false (default) or true  |
|           | limits.cpu  | Describes the maximum amount of CPU allowed | Default is 2000m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu)  |
|           | limits.memory | Describes the maximum amount of memory allowed | Default is 2Gi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |
|           | requests.cpu  | Describes the minimum amount of CPU required - if not specified will default to limit (if specified) or otherwise implementation-defined value. | Default is 50m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu) |
|           | requests.memory | Describes the minimum amount of memory required If not specified, the memory amount will default to the limit (if specified) or the implementation-defined value | Default is 200Mi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |

## Scaling
To increase or decrease the number of DevOps Deploy Agent instances issue the following command:

```bash
$ kubectl scale --replicas=2 statefulset/releaseName-hcl-launch-agent-prod
```

## User defined utilities to run in DevOps Deploy Agent container
Users can extend the tools the agent can execute without having to modify the image. The user can provide a Persistent Volume Claim(PVC) in the values.yaml file. This PVC would refer to a Persistent Volume(PV) the user has created and load the executables they want the agent to run. See the userUtils.existingClaimName and userUtils.executablesPath in the "Configuration" on how to provide user defined utilities.  

## Storage
See the Prerequisites section of this page for storage information.

## Limitations
