# Swift-Kitura Helm Chart

Encapsulates Helm chart and other cloud pak artifacts for Swift running a Kitura server.

## Introduction

This chart deploys a Swift application running a Kitura server.

## Chart Details

This chart will install the following:

* One [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) to rollout a [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) to create [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) with Swift based containers.
* A [NodePort Service](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) to enable secure connection from outside the cluster.

## Prerequisites

If you prefer to install from the command prompt, you will need:

* The `cloudctl`, `kubectl` and `helm` commands available.
* Your environment configured to connect to the target cluster.

The installation environment has the following prerequisites:

* Kubernetes version `1.11.1`.
* [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) support in the underlying infrastructure if `logs.persistLogs` (See "[Create Persistent Volumes](#create-persistent-volumes)" below).

### PodSecurityPolicy Requirements

This chart requires a PodSecurityPolicy to be bound to the target namespace prior to installation. Choose either a predefined PodSecurityPolicy or have your cluster administrator create a custom PodSecurityPolicy for you:

* Predefined PodSecurityPolicy name: [`ibm-restricted-psp`](https://ibm.biz/cpkspec-psp)
* Custom PodSecurityPolicy definition:

```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: kitura-psp
spec:
  allowPrivilegeEscalation: false
  forbiddenSysctls:
  - '*'
  fsGroup:
    ranges:
    - max: 65535
      min: 1
    rule: MustRunAs
  requiredDropCapabilities:
  - ALL
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    ranges:
    - max: 65535
      min: 1
    rule: MustRunAs
  volumes:
  - configMap
  - emptyDir
  - projected
  - secret
  - downwardAPI
  - persistentVolumeClaim
```

* Custom ClusterRole for the custom PodSecurityPolicy:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kitura-clusterrole
rules:
- apiGroups:
  - extensions
  resourceNames:
  - kitura-psp
  resources:
  - podsecuritypolicies
  verbs:
  - use
```

#### Configuration scripts can be used to create the required resources

Download the following scripts located at [/ibm_cloud_pak/pak_extensions/pre-install](https://github.com/IBM/charts/tree/master/stable/kitura/ibm_cloud_pak/pak_extensions/pre-install) directory.

* The pre-install instructions are located at `clusterAdministration/createSecurityClusterPrereqs.sh` for cluster admins to create the PodSecurityPolicy and ClusterRole for all releases of this chart.

* The namespace scoped instructions are located at `namespaceAdministration/createSecurityNamespacePrereqs.sh` for team admin/operator to create the RoleBinding for the namespace. This script takes one argument; the name of a pre-existing namespace where the chart will be installed.
  * Example usage: `./createSecurityNamespacePrereqs.sh myNamespace`

#### Configuration scripts can be used to clean up resources created

Download the following scripts located at [/ibm_cloud_pak/pak_extensions/post-delete](https://github.com/IBM/charts/tree/master/stable/kitura/ibm_cloud_pak/pak_extensions/post-delete) directory.

* The post-delete instructions are located at `clusterAdministration/deleteSecurityClusterPrereqs.sh` for cluster admins to delete the PodSecurityPolicy and ClusterRole for all releases of this chart.

* The namespace scoped instructions are located at `namespaceAdministration/deleteSecurityNamespacePrereqs.sh` for team admin/operator to delete the RoleBinding for the namespace. This script takes one argument; the name of the namespace where the chart was installed.
  * Example usage: `./deleteSecurityNamespacePrereqs.sh myNamespace`

### Red Hat OpenShift SecurityContextConstraints Requirements

This chart requires a `SecurityContextConstraints` to be bound to the target namespace prior to installation. To meet this requirement there may be cluster scoped as well as namespace scoped pre and post actions that need to occur.

The predefined `SecurityContextConstraints` name: [`ibm-restricted-scc`](https://ibm.biz/cpkspec-scc) has been verified for this chart, if your target namespace is bound to this `SecurityContextConstraints` resource you can proceed to install the chart.

#### Creating the required resources

This chart defines a custom `SecurityContextConstraints` which can be used to finely control the permissions/capabilities needed to deploy this chart. You can enable this custom `SecurityContextConstraints` resource using the supplied instructions or scripts in the `pak_extensions/pre-install` directory.

* From the user interface, you can copy and paste the following snippets to enable the custom `SecurityContextConstraints`
  * Custom `SecurityContextConstraints` definition:

  ```yaml
  apiVersion: security.openshift.io/v1
  kind: SecurityContextConstraints
  metadata:
    annotations:
    name: kitura-scc
  allowHostDirVolumePlugin: false
  allowHostIPC: false
  allowHostNetwork: false
  allowHostPID: false
  allowHostPorts: false
  allowPrivilegedContainer: false
  allowedCapabilities: []
  allowedFlexVolumes: []
  defaultAddCapabilities: []
  fsGroup:
    type: MustRunAs
    ranges:
    - max: 65535
      min: 1
  readOnlyRootFilesystem: false
  requiredDropCapabilities:
  - ALL
  runAsUser:
    type: MustRunAsNonRoot
  seccompProfiles:
  - docker/default
  seLinuxContext:
    type: RunAsAny
  supplementalGroups:
    type: MustRunAs
    ranges:
    - max: 65535
      min: 1
  volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
  priority: 0
  ```

* From the command line, you can run the setup scripts included under `pak_extensions/pre-install`
  As a cluster admin the pre-install instructions are located at:
  * `pre-install/clusterAdministration/createSecurityClusterPrereqs.sh`

  As team admin the namespace scoped instructions are located at:
  * `pre-install/namespaceAdministration/createSecurityNamespacePrereqs.sh`

## Resources Required

See [Requirements for Kitura on macOS](https://www.kitura.io/docs/getting-started/installation-mac.html) and [Requirements for Kitura on Linux](https://www.kitura.io/docs/getting-started/installation-linux.html).

## Installing the Chart

There are three steps to run your Kitura application:

* Create an Kitura application image
* Create persistent volumes (Optional)
* Install the Helm chart

### Create Persistent Volumes

Persistence is not enabled by default so no persistent volumes are required. If you are not using persistence, you can skip this section.

Enable persistence if you want logs generated by the server to be retained in the event of a restart. If persistence is enabled, one physical volume will be required for each instance of the server.

To create physical volumes, you must have the Cluster administrator role.

You can find more information about storage requirements below.

For volumes that support ownership management, specify the group ID of the group owning the persistent volumes' file systems using the `persistence.fsGroupGid` parameter.

### Install the Helm chart

Add the IBM Cloud Private internal Helm repository called `local-charts` to the Helm CLI as an external repository, as described [here](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/app_center/add_int_helm_repo_to_cli.html).

Install the chart, specifying the release name and namespace with the following command:

```bash
helm install --name <release_name> --namespace=<namespace_name> kitura
```

NOTE: The release name should consist of lower-case alphanumeric characters and not start with a digit or contain a space.

The command deploys a Swift application with Kitura included on the Kubernetes cluster with the default configuration.

The [Configuration](#configuration) lists the parameters that can be overridden during installation by adding them to the Helm install command as follows:

```bash
--set key=value[,key=value]
```

### Verifying the Chart

See NOTES.txt associated with this chart for verification instructions

### Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```bash
helm delete my-release --purge
```

This command removes all the Kubernetes components associated with the chart, except any persistent volume claims (PVCs) which is created when `logs.persistLogs`. This is the default behavior of Kubernetes, and ensures that valuable data is not deleted. In order to delete the server data, you can delete the PVC using the following command:

```bash
kubectl delete pvc my-pvc
```

Note: You can use `kubectl get pvc` to see the list of available PVCs.

### Cleanup any pre-requirement that were created

If cleanup scripts where included in the [/ibm_cloud_pak/pak_extensions/prereqs](./ibm_cloud_pak/pak_extensions/prereqs) directory; run them to cleanup namespace and cluster scoped resources when appropriate.

## Configuration

The following tables lists the configurable parameters of the Swift Kitura chart and their default values.

| Parameter                  | Description                                     | Default                                                    |
| -----------------------    | ---------------------------------------------   | ---------------------------------------------------------- |
| `replicaCount`             | The number of desired replica pods that run simultaneously                   | `1`                                                        |
| `image.repository`         | Docker image repository                         | `ibmcom/swift-kitura`                             |
| `image.pullPolicy`         | Docker image pull policy. Defaults to `Always` when the latest tag is specified.                             | `IfNotPresent`                                             |
| `image.tag`                | Docker image tag                                | `9.0.5.0`                                          |
| `image.pullSecrets`        | Image pull secrets, if using Docker registries that require credentials. | `[]`                                |
| `image.extraEnvs`          | Additional Environment Variables                | `[]`                                                       |
| `image.extraVolumeMounts`  | Extra Volume Mounts                             | `[]`                                                       |
| `image.security`           | Configure the security attributes of the image  | `{}`
| `deployment.annotations`   | Custom deployment annotations                   | `{}`                                                       |
| `deployment.labels`        | Custom deployment labels                        | `{}`                                                       |
| `pod.annotations`          | Custom pod annotations                          | `{}`                                                       |
| `pod.labels`               | Custom pod labels                               | `{}`                                                       |
| `pod.extraVolumes`         | Additional Volumes for server pods.             | `{}`                                                       |
| `pod.security`             | Configure the security attributes of the pod    | `{}`
| `service.type`             | Kubernetes service type exposing ports| `NodePort`                                                 |
| `service.name`             | Kubernetes service name for HTTP                                | `https-was`                                                |
| `service.port`             | The abstracted service port for HTTP, which other pods use to access this service                     | `8080`                                                    |
| `service.targetPort`       | Secured HTTP port the container accepts traffic on. Ensure that it matches the port exposed by the container       | `8080`                                                    |
| `service.annotations`      | Kubernetes service custom annotations"|        `{}`                                                 |
| `service.labels`           | Kubernetes service custom labels"|        `{}`                                                      |
| `ingress.enabled`          | Specifies whether to enable [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)                                  | `false`                                                    |
| `ingress.rewriteTarget`    | Specifies target URI where traffic must be redirected | `/`                                                  |
| `ingress.path`             | Specifies path for the Ingress HTTP rule        | `/`                                                        |
| `ingress.annotations`      | Kubernetes ingress custom annotations |        `{}`                                                 |
| `ingress.labels`           | Kubernetes ingress custom labels      |        `{}`                                                      |
| `ingress.hosts`             | Specifies an array of fully qualified domain names of Ingress, as defined by RFC 3986. | `[]` |
| `ingress.secretName`       | Specifies the name of the Kubernetes secret that contains [Ingress'](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls) TLS certificate and key.   | `""` |
| `configProperties.configMapName`      | Name of the [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-a-configmap) that contains one or more configuration properties files to configure your Swift Kitura environment | `""`         |
| `readinessProbe.initialDelaySeconds`| Number of seconds after the container has started before readiness probe is initiated | `30`        |
| `readinessProbe.periodSeconds`| How often (in seconds) to perform the readiness probe. Minimum value is 1  | `5`                                                        |
| `readinessProbe.httpGet.enabled`| Specifies whether to determine readiness by sending an HTTP GET request to the specified path on the server (`readinessProbe.httpGet.path`). Otherwise, uses connection to a TCP socket on the specified target port  | `false` |
| `readinessProbe.httpGet.path`| Path to access on the server | `/`                                                                         |
| `livenessProbe.initialDelaySeconds`| Number of seconds after the container has started before liveness probe is initiated | `180`         |
| `livenessProbe.periodSeconds`| How often (in seconds) to perform the liveness probe. Minimum value is 1 | `20`                            |
| `livenessProbe.httpGet.enabled`| Specifies whether to determine livenessProbe by sending an HTTP GET request to the specified path on the server (`livenessProbe.httpGet.path`). Otherwise, uses connection to a TCP socket on the specified target port  | `false` |
| `livenessProbe.httpGet.path`| Path to access on the server | `/`                                                                          |
| `autoscaling.enabled`      | Enables a Horizontal Pod Autoscaler. Enabling this field disables the `replicaCount` field | `false`         |
| `autoscaling.minReplicas`  | Lower limit for the number of pods that can be set by the autoscaler              | `1`                                                        |
| `autoscaling.maxReplicas`  | Upper limit for the number of pods that can be set by the autoscaler. It cannot be lower than `minReplicas`| `10`                                 |
| `autoscaling.targetCPUUtilizationPercentage` | Target average CPU utilization (represented as a percentage of requested CPU) over all the pods | `50` |
| `resources.constraints.enabled`| Specifies whether the resource constraints are enabled               | `false`                                                    |
| `resources.requests.memory`| Describes the minimum amount of memory required. Corresponds to `requests.memory` in Kubernetes.                        | `2Gi`                                                      |
| `resources.requests.cpu`   | Describes the minimum amount of CPU required. Corresponds to `requests.cpu` in Kubernetes                           | `500m`                                                     |
| `resources.limits.memory`  | Describes the maximum amount of memory allowed                          | `10Gi`                                                     |
| `resources.limits.cpu`     | Describes the maximum amount of CPU allowed                             | `500m`                                                     |
| `arch.amd64`               | Architecture preference for amd64 worker node   | `2 - No preference`                                        |
| `arch.ppc64le`             | Architecture preference for ppc64le worker node | `0 - Do not use`                                           |
| `arch.s390x`               | Architecture preference for s390x worker node   | `0 - Do not use`                                           |
| `persistence.name`         | A prefix for the name of the persistence volume claim (PVC). A PVC will not be created unless `logs.persistLogs` is set to `true` | `pvc` |
| `persistence.size`         | Size of the volume to hold all the persisted data | `1Gi`                                                    |
| `persistence.fsGroupGid`             | The group ID added to the containers with persistent storage to allow access. Volumes that support ownership management must be owned and writable by this group ID | `nil`                        |
| `persistence.useDynamicProvisioning` | If `true`, the persistent volume claim will use the `storageClassName` to bind the volume. Otherwise, the selector will be used for the binding process | `true` |
| `persistence.storageClassName`       | Specifies a StorageClass pre-created by the Kubernetes sysadmin. When set to `""`, then the PVC is bound to the default StorageClass setup by kube Administrator | `""` |
| `persistence.selector.label`         | When matching a PV, the label is used to find a match on the key. See Kubernetes - [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) | `""` |
| `persistence.selector.value`         | When matching a PV, the value is used to find a match on the values. See Kubernetes - [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) | `""` |
| `logs.persistLogs`         | When `true`, the server logs will be persisted to the volume bound according to the persistence parameters | `false` |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install ...`.

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. The following commands creates a YAML file and installs a chart:

```bash
$ cat << EOF > my-values.yaml
logs:
  persistLogs: false
image:
  repository: mycluster.icp:8500/my-namespace/my-app
  tag: v1
  pullPolicy: Always
EOF

$ helm install --name my-release --namespace=my-namespace kitura --values=my-values.yaml
```

### Configure Environment using Configuration Properties

Your Swift Kitura environment can be configured using a ConfigMap that contains one or more configuration properties files. The configuration values will be automatically applied to the application. Specify the name of your ConfigMap using `configProperties.configMapName` parameter.

* You can find more information here :
  * [Creating a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-a-configmap)

### Configure Liveness and Readiness Probes

With the default configuration, readiness and liveness probes are determined by attempting to establish a connection by opening a TCP socket to your container on the specified target port. If connection can be established then the container is considered healthy, otherwise the container is considered a failure.

You can use HTTP probes instead to determine readiness and liveness. For readiness, configure `readinessProbe.httpGet.enabled` and `readinessProbe.httpGet.path` parameters. For liveness, configure `livenessProbe.httpGet.enabled` and `livenessProbe.httpGet.path` parameters.

The `readinessProbe.initialDelaySeconds` and `livenessProbe.initialDelaySeconds` parameters define how long to wait before performing the first probe. You should set appropriate values for your container to ensure that the readiness and liveness probes don’t interfere with each other. Otherwise, the liveness probe might continiously restart the pod and the pod will never be marked as ready.

More information about configuring liveness and readiness probes can be found [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)

## Storage

If persistence is enabled, each server Pod requires one Physical Volume. You either need to create a
[persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#static) for each server Pod, or specify a
storage class that supports [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#dynamic).

If these persistent volumes are to be created manually, this must be done by the system administrator who will add these to a central pool before the Helm chart can be installed. The installation will then claim the required number of persistent volumes from this pool. For manual creation, `persistence.useDynamicProvisioning` must be disabled in the Helm chart when it is installed. It is up to the administrator to provide appropriate storage to back these physical volumes.

If these persistent volumes are to be created automatically at the time of installation, the system administrator must enable support for this prior to installing the Helm chart. For automatic creation `persistence.useDynamicProvisioning` should be enabled in the Helm chart when it is installed and storage class names provided to define which types of Persistent Volume get allocated to the deployment. If `persistence.storageClassName` is not specified, the default StorageClass setup by kube Administrator would be used.

More information about persistent volumes and the system administration steps required can be found [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

## Limitations

See release notes (RELEASENOTES.md) for the list of limitations.

## Documentation

For more information about Kitura, visit the [Kitura Website](https://www.kitura.io/).