# Giới thiệu
Ý tưởng cài đặt Longhorn hiểu đơn giản như sau:
- Cấu hình cho Longhorn trỏ đến folder mặc định lưu data của longhorn `/data/longhorn-storage` trên các `Worker Node`
- Longhorn sẽ chạy trên tất cả các `Worker Node` mà có phân vùng lưu trữ như cấu hình bên trên
- Mỗi `Worker Node` được xem là 1 `Longhorn Node`, trên mỗi `Longhorn Node` có thể có 1 hoặc nhiều Disk mà ta có thể cấu hình thêm sau khi đã cài đặt Longhorn

Ta sẽ có 2 phần:
- `Longhorn Storage:` Là storage quản lý thiết bị lưu trữ, nó có vai trò giống như NFS Server vậy
- `Longhorn Storage Class:` Là một object trên K8S đảm nhiệm việc nhận các yêu cầu tạo Volume trên K8S (PV/PVC) sau đó kết nối với Longhorn Storage để tạo ra phân vùng lưu trữ trên thiết bị lưu trữ

Các bước thực hiện trong bài lab này như sau:
- Chuẩn bị partition lưu data (disk) trên các Worker Node
- Cài đặt Longhorn Storage trên K8S bằng Helm Chart
- Tạo Longhorn SC trên K8S
- Test thử tạo PV/PVC và tạo Pod dùng Longhorn SC

# Cài đặt Longhorn Storage
## Tạo folder lưu data
- Trên các `Worker Node`, tạo folder lưu data của Longhorn là `/data/longhorn-storage`
```
sudo mkdir -p /data/longhorn-storage
```

## Cài đặt Longhorn Storage
- Trên `local machine`, tạo folder lưu Helm Chart và các file cấu hình Longhorn:
```
mkdir $HOME/k8s/longhorn-storage && cd $HOME/k8s/longhorn-storage
```

- Pull Helm Chart của Longhorn
```
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm search repo longhorn
helm pull longhorn/longhorn --version 1.7.1
tar -xzf longhorn-1.7.1.tgz
```

- Copy file value default của Longhorn Helm Chart ra 1 file value khác
```
cp $HOME/k8s/longhorn-storage/longhorn/values.yaml $HOME/k8s/longhorn-storage/values-longhorn.yaml
```
Sửa file values-longhorn.yaml và cập nhật một số tham số như sau:
```
# Default values for longhorn.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
global:
  # -- Toleration for nodes allowed to run user-deployed components such as Longhorn Manager, Longhorn UI, and Longhorn Driver Deployer.
  tolerations: []
  # -- Node selector for nodes allowed to run user-deployed components such as Longhorn Manager, Longhorn UI, and Longhorn Driver Deployer.
  nodeSelector: {}
  cattle:
    # -- Default system registry.
    systemDefaultRegistry: ""
    windowsCluster:
      # -- Setting that allows Longhorn to run on a Rancher Windows cluster.
      enabled: false
      # -- Toleration for Linux nodes that can run user-deployed Longhorn components.
      tolerations:
      - key: "cattle.io/os"
        value: "linux"
        effect: "NoSchedule"
        operator: "Equal"
      # -- Node selector for Linux nodes that can run user-deployed Longhorn components.
      nodeSelector:
        kubernetes.io/os: "linux"
      defaultSetting:
        # -- Toleration for system-managed Longhorn components.
        taintToleration: cattle.io/os=linux:NoSchedule
        # -- Node selector for system-managed Longhorn components.
        systemManagedComponentsNodeSelector: kubernetes.io/os:linux

networkPolicies:
  # -- Setting that allows you to enable network policies that control access to Longhorn pods.
  enabled: false
  # -- Distribution that determines the policy for allowing access for an ingress. (Options: "k3s", "rke2", "rke1")
  type: "k3s"

image:
  longhorn:
    engine:
      # -- Repository for the Longhorn Engine image.
      repository: longhornio/longhorn-engine
      # -- Tag for the Longhorn Engine image.
      tag: v1.7.1
    manager:
      # -- Repository for the Longhorn Manager image.
      repository: longhornio/longhorn-manager
      # -- Tag for the Longhorn Manager image.
      tag: v1.7.1
    ui:
      # -- Repository for the Longhorn UI image.
      repository: longhornio/longhorn-ui
      # -- Tag for the Longhorn UI image.
      tag: v1.7.1
    instanceManager:
      # -- Repository for the Longhorn Instance Manager image.
      repository: longhornio/longhorn-instance-manager
      # -- Tag for the Longhorn Instance Manager image.
      tag: v1.7.1
    shareManager:
      # -- Repository for the Longhorn Share Manager image.
      repository: longhornio/longhorn-share-manager
      # -- Tag for the Longhorn Share Manager image.
      tag: v1.7.1
    backingImageManager:
      # -- Repository for the Backing Image Manager image. When unspecified, Longhorn uses the default value.
      repository: longhornio/backing-image-manager
      # -- Tag for the Backing Image Manager image. When unspecified, Longhorn uses the default value.
      tag: v1.7.1
    supportBundleKit:
      # -- Repository for the Longhorn Support Bundle Manager image.
      repository: longhornio/support-bundle-kit
      # -- Tag for the Longhorn Support Bundle Manager image.
      tag: v0.0.42
  csi:
    attacher:
      # -- Repository for the CSI attacher image. When unspecified, Longhorn uses the default value.
      repository: longhornio/csi-attacher
      # -- Tag for the CSI attacher image. When unspecified, Longhorn uses the default value.
      tag: v4.6.1
    provisioner:
      # -- Repository for the CSI Provisioner image. When unspecified, Longhorn uses the default value.
      repository: longhornio/csi-provisioner
      # -- Tag for the CSI Provisioner image. When unspecified, Longhorn uses the default value.
      tag: v4.0.1
    nodeDriverRegistrar:
      # -- Repository for the CSI Node Driver Registrar image. When unspecified, Longhorn uses the default value.
      repository: longhornio/csi-node-driver-registrar
      # -- Tag for the CSI Node Driver Registrar image. When unspecified, Longhorn uses the default value.
      tag: v2.12.0
    resizer:
      # -- Repository for the CSI Resizer image. When unspecified, Longhorn uses the default value.
      repository: longhornio/csi-resizer
      # -- Tag for the CSI Resizer image. When unspecified, Longhorn uses the default value.
      tag: v1.11.1
    snapshotter:
      # -- Repository for the CSI Snapshotter image. When unspecified, Longhorn uses the default value.
      repository: longhornio/csi-snapshotter
      # -- Tag for the CSI Snapshotter image. When unspecified, Longhorn uses the default value.
      tag: v7.0.2
    livenessProbe:
      # -- Repository for the CSI liveness probe image. When unspecified, Longhorn uses the default value.
      repository: longhornio/livenessprobe
      # -- Tag for the CSI liveness probe image. When unspecified, Longhorn uses the default value.
      tag: v2.14.0
  openshift:
    oauthProxy:
      # -- Repository for the OAuth Proxy image. This setting applies only to OpenShift users.
      repository: longhornio/openshift-origin-oauth-proxy
      # -- Tag for the OAuth Proxy image. This setting applies only to OpenShift users. Specify OCP/OKD version 4.1 or later. The latest stable version is 4.15.
      tag: 4.15
  # -- Image pull policy that applies to all user-deployed Longhorn components, such as Longhorn Manager, Longhorn driver, and Longhorn UI.
  pullPolicy: IfNotPresent

service:
  ui:
    # -- Service type for Longhorn UI. (Options: "ClusterIP", "NodePort", "LoadBalancer", "Rancher-Proxy")
    type: NodePort
    # -- NodePort port number for Longhorn UI. When unspecified, Longhorn selects a free port between 30000 and 32767.
    nodePort: 30888
  manager:
    # -- Service type for Longhorn Manager.
    type: ClusterIP
    # -- NodePort port number for Longhorn Manager. When unspecified, Longhorn selects a free port between 30000 and 32767.
    nodePort: ""

persistence:
  # -- Setting that allows you to specify the default Longhorn StorageClass.
  defaultClass: true
  # -- Filesystem type of the default Longhorn StorageClass.
  defaultFsType: ext4
  # -- mkfs parameters of the default Longhorn StorageClass.
  defaultMkfsParams: ""
  # -- Replica count of the default Longhorn StorageClass.
  defaultClassReplicaCount: 3
  # -- Data locality of the default Longhorn StorageClass. (Options: "disabled", "best-effort")
  defaultDataLocality: disabled
  # -- Reclaim policy that provides instructions for handling of a volume after its claim is released. (Options: "Retain", "Delete")
  reclaimPolicy: Delete
  # -- Setting that allows you to enable live migration of a Longhorn volume from one node to another.
  migratable: false
  # -- Setting that disables the revision counter and thereby prevents Longhorn from tracking all write operations to a volume. When salvaging a volume, Longhorn uses properties of the volume-head-xxx.img file (the last file size and the last time the file was modified) to select the replica to be used for volume recovery.
  disableRevisionCounter: "true"
  # -- Set NFS mount options for Longhorn StorageClass for RWX volumes
  nfsOptions: ""
  recurringJobSelector:
    # -- Setting that allows you to enable the recurring job selector for a Longhorn StorageClass.
    enable: false
    # -- Recurring job selector for a Longhorn StorageClass. Ensure that quotes are used correctly when specifying job parameters. (Example: `[{"name":"backup", "isGroup":true}]`)
    jobList: []
  backingImage:
    # -- Setting that allows you to use a backing image in a Longhorn StorageClass.
    enable: false
    # -- Backing image to be used for creating and restoring volumes in a Longhorn StorageClass. When no backing images are available, specify the data source type and parameters that Longhorn can use to create a backing image.
    name: ~
    # -- Data source type of a backing image used in a Longhorn StorageClass.
    # If the backing image exists in the cluster, Longhorn uses this setting to verify the image.
    # If the backing image does not exist, Longhorn creates one using the specified data source type.
    dataSourceType: ~
    # -- Data source parameters of a backing image used in a Longhorn StorageClass.
    # You can specify a JSON string of a map. (Example: `'{\"url\":\"https://backing-image-example.s3-region.amazonaws.com/test-backing-image\"}'`)
    dataSourceParameters: ~
    # -- Expected SHA-512 checksum of a backing image used in a Longhorn StorageClass.
    expectedChecksum: ~
  defaultDiskSelector:
    # -- Setting that allows you to enable the disk selector for the default Longhorn StorageClass.
    enable: false
    # -- Disk selector for the default Longhorn StorageClass. Longhorn uses only disks with the specified tags for storing volume data. (Examples: "nvme,sata")
    selector: ""
  defaultNodeSelector:
    # -- Setting that allows you to enable the node selector for the default Longhorn StorageClass.
    enable: false
    # -- Node selector for the default Longhorn StorageClass. Longhorn uses only nodes with the specified tags for storing volume data. (Examples: "storage,fast")
    selector: ""
  # -- Setting that allows you to enable automatic snapshot removal during filesystem trim for a Longhorn StorageClass. (Options: "ignored", "enabled", "disabled")
  removeSnapshotsDuringFilesystemTrim: ignored

preUpgradeChecker:
  # -- Setting that allows Longhorn to perform pre-upgrade checks. Disable this setting when installing Longhorn using Argo CD or other GitOps solutions.
  jobEnabled: true
  # -- Setting that allows Longhorn to perform upgrade version checks after starting the Longhorn Manager DaemonSet Pods. Disabling this setting also disables `preUpgradeChecker.jobEnabled`. Longhorn recommends keeping this setting enabled.
  upgradeVersionCheck: true

csi:
  # -- kubelet root directory. When unspecified, Longhorn uses the default value.
  kubeletRootDir: ~
  # -- Replica count of the CSI Attacher. When unspecified, Longhorn uses the default value ("3").
  attacherReplicaCount: ~
  # -- Replica count of the CSI Provisioner. When unspecified, Longhorn uses the default value ("3").
  provisionerReplicaCount: ~
  # -- Replica count of the CSI Resizer. When unspecified, Longhorn uses the default value ("3").
  resizerReplicaCount: ~
  # -- Replica count of the CSI Snapshotter. When unspecified, Longhorn uses the default value ("3").
  snapshotterReplicaCount: ~

defaultSettings:
  # -- Endpoint used to access the backupstore. (Options: "NFS", "CIFS", "AWS", "GCP", "AZURE")
  backupTarget: ~
  # -- Name of the Kubernetes secret associated with the backup target.
  backupTargetCredentialSecret: ~
  # -- Setting that allows Longhorn to automatically attach a volume and create snapshots or backups when recurring jobs are run.
  allowRecurringJobWhileVolumeDetached: ~
  # -- Setting that allows Longhorn to automatically create a default disk only on nodes with the label "node.longhorn.io/create-default-disk=true" (if no other disks exist). When this setting is disabled, Longhorn creates a default disk on each node that is added to the cluster.
  createDefaultDiskLabeledNodes: ~
  # -- Default path for storing data on a host. The default value is "/var/lib/longhorn/".
  defaultDataPath: /data/longhorn-storage/
  # -- Default data locality. A Longhorn volume has data locality if a local replica of the volume exists on the same node as the pod that is using the volume.
  defaultDataLocality: ~
  # -- Setting that allows scheduling on nodes with healthy replicas of the same volume. This setting is disabled by default.
  replicaSoftAntiAffinity: true
  # -- Setting that automatically rebalances replicas when an available node is discovered.
  replicaAutoBalance: ~
  # -- Percentage of storage that can be allocated relative to hard drive capacity. The default value is "100".
  storageOverProvisioningPercentage: ~
  # -- Percentage of minimum available disk capacity. When the minimum available capacity exceeds the total available capacity, the disk becomes unschedulable until more space is made available for use. The default value is "25".
  storageMinimalAvailablePercentage: 15
  # -- Percentage of disk space that is not allocated to the default disk on each new Longhorn node.
  storageReservedPercentageForDefaultDisk: ~
  # -- Upgrade Checker that periodically checks for new Longhorn versions. When a new version is available, a notification appears on the Longhorn UI. This setting is enabled by default
  upgradeChecker: false
  # -- Default number of replicas for volumes created using the Longhorn UI. For Kubernetes configuration, modify the `numberOfReplicas` field in the StorageClass. The default value is "3".
  defaultReplicaCount: 2
  # -- Default Longhorn StorageClass. "storageClassName" is assigned to PVs and PVCs that are created for an existing Longhorn volume. "storageClassName" can also be used as a label, so it is possible to use a Longhorn StorageClass to bind a workload to an existing PV without creating a Kubernetes StorageClass object. The default value is "longhorn-static".
  defaultLonghornStaticStorageClass: ~
  # -- Number of seconds that Longhorn waits before checking the backupstore for new backups. The default value is "300". When the value is "0", polling is disabled.
  backupstorePollInterval: 500
  # -- Number of minutes that Longhorn keeps a failed backup resource. When the value is "0", automatic deletion is disabled.
  failedBackupTTL: ~
  # -- Setting that restores recurring jobs from a backup volume on a backup target and creates recurring jobs if none exist during backup restoration.
  restoreVolumeRecurringJobs: ~
  # -- Maximum number of successful recurring backup and snapshot jobs to be retained. When the value is "0", a history of successful recurring jobs is not retained.
  recurringSuccessfulJobsHistoryLimit: ~
  # -- Maximum number of failed recurring backup and snapshot jobs to be retained. When the value is "0", a history of failed recurring jobs is not retained.
  recurringFailedJobsHistoryLimit: ~
  # -- Maximum number of snapshots or backups to be retained.
  recurringJobMaxRetention: ~
  # -- Maximum number of failed support bundles that can exist in the cluster. When the value is "0", Longhorn automatically purges all failed support bundles.
  supportBundleFailedHistoryLimit: ~
  # -- Taint or toleration for system-managed Longhorn components.
  # Specify values using a semicolon-separated list in `kubectl taint` syntax (Example: key1=value1:effect; key2=value2:effect).
  taintToleration: ~
  # -- Node selector for system-managed Longhorn components.
  systemManagedComponentsNodeSelector: ~
  # -- PriorityClass for system-managed Longhorn components.
  # This setting can help prevent Longhorn components from being evicted under Node Pressure.
  # Notice that this will be applied to Longhorn user-deployed components by default if there are no priority class values set yet, such as `longhornManager.priorityClass`.
  priorityClass: &defaultPriorityClassNameRef "longhorn-critical"
  # -- Setting that allows Longhorn to automatically salvage volumes when all replicas become faulty (for example, when the network connection is interrupted). Longhorn determines which replicas are usable and then uses these replicas for the volume. This setting is enabled by default.
  autoSalvage: ~
  # -- Setting that allows Longhorn to automatically delete a workload pod that is managed by a controller (for example, daemonset) whenever a Longhorn volume is detached unexpectedly (for example, during Kubernetes upgrades). After deletion, the controller restarts the pod and then Kubernetes handles volume reattachment and remounting.
  autoDeletePodWhenVolumeDetachedUnexpectedly: ~
  # -- Setting that prevents Longhorn Manager from scheduling replicas on a cordoned Kubernetes node. This setting is enabled by default.
  disableSchedulingOnCordonedNode: ~
  # -- Setting that allows Longhorn to schedule new replicas of a volume to nodes in the same zone as existing healthy replicas. Nodes that do not belong to any zone are treated as existing in the zone that contains healthy replicas. When identifying zones, Longhorn relies on the label "topology.kubernetes.io/zone=<Zone name of the node>" in the Kubernetes node object.
  replicaZoneSoftAntiAffinity: ~
  # -- Setting that allows scheduling on disks with existing healthy replicas of the same volume. This setting is enabled by default.
  replicaDiskSoftAntiAffinity: ~
  # -- Policy that defines the action Longhorn takes when a volume is stuck with a StatefulSet or Deployment pod on a node that failed.
  nodeDownPodDeletionPolicy: do-nothing
  # -- Policy that defines the action Longhorn takes when a node with the last healthy replica of a volume is drained.
  nodeDrainPolicy: ~
  # -- Setting that allows automatic detaching of manually-attached volumes when a node is cordoned.
  detachManuallyAttachedVolumesWhenCordoned: ~
  # -- Number of seconds that Longhorn waits before reusing existing data on a failed replica instead of creating a new replica of a degraded volume.
  replicaReplenishmentWaitInterval: ~
  # -- Maximum number of replicas that can be concurrently rebuilt on each node.
  concurrentReplicaRebuildPerNodeLimit: ~
  # -- Maximum number of volumes that can be concurrently restored on each node using a backup. When the value is "0", restoration of volumes using a backup is disabled.
  concurrentVolumeBackupRestorePerNodeLimit: ~
  # -- Setting that disables the revision counter and thereby prevents Longhorn from tracking all write operations to a volume. When salvaging a volume, Longhorn uses properties of the "volume-head-xxx.img" file (the last file size and the last time the file was modified) to select the replica to be used for volume recovery. This setting applies only to volumes created using the Longhorn UI.
  disableRevisionCounter: "true"
  # -- Image pull policy for system-managed pods, such as Instance Manager, engine images, and CSI Driver. Changes to the image pull policy are applied only after the system-managed pods restart.
  systemManagedPodsImagePullPolicy: ~
  # -- Setting that allows you to create and attach a volume without having all replicas scheduled at the time of creation.
  allowVolumeCreationWithDegradedAvailability: ~
  # -- Setting that allows Longhorn to automatically clean up the system-generated snapshot after replica rebuilding is completed.
  autoCleanupSystemGeneratedSnapshot: ~
  # -- Setting that allows Longhorn to automatically clean up the snapshot generated by a recurring backup job.
  autoCleanupRecurringJobBackupSnapshot: ~
  # -- Maximum number of engines that are allowed to concurrently upgrade on each node after Longhorn Manager is upgraded. When the value is "0", Longhorn does not automatically upgrade volume engines to the new default engine image version.
  concurrentAutomaticEngineUpgradePerNodeLimit: ~
  # -- Number of minutes that Longhorn waits before cleaning up the backing image file when no replicas in the disk are using it.
  backingImageCleanupWaitInterval: ~
  # -- Number of seconds that Longhorn waits before downloading a backing image file again when the status of all image disk files changes to "failed" or "unknown".
  backingImageRecoveryWaitInterval: ~
  # -- Percentage of the total allocatable CPU resources on each node to be reserved for each instance manager pod when the V1 Data Engine is enabled. The default value is "12".
  guaranteedInstanceManagerCPU: 15
  # -- Setting that notifies Longhorn that the cluster is using the Kubernetes Cluster Autoscaler.
  kubernetesClusterAutoscalerEnabled: ~
  # -- Setting that allows Longhorn to automatically delete an orphaned resource and the corresponding data (for example, stale replicas). Orphaned resources on failed or unknown nodes are not automatically cleaned up.
  orphanAutoDeletion: ~
  # -- Storage network for in-cluster traffic. When unspecified, Longhorn uses the Kubernetes cluster network.
  storageNetwork: ~
  # -- Flag that prevents accidental uninstallation of Longhorn.
  deletingConfirmationFlag: ~
  # -- Timeout between the Longhorn Engine and replicas. Specify a value between "8" and "30" seconds. The default value is "8".
  engineReplicaTimeout: ~
  # -- Setting that allows you to enable and disable snapshot hashing and data integrity checks.
  snapshotDataIntegrity: ~
  # -- Setting that allows disabling of snapshot hashing after snapshot creation to minimize impact on system performance.
  snapshotDataIntegrityImmediateCheckAfterSnapshotCreation: ~
  # -- Setting that defines when Longhorn checks the integrity of data in snapshot disk files. You must use the Unix cron expression format.
  snapshotDataIntegrityCronjob: ~
  # -- Setting that allows Longhorn to automatically mark the latest snapshot and its parent files as removed during a filesystem trim. Longhorn does not remove snapshots containing multiple child files.
  removeSnapshotsDuringFilesystemTrim: ~
  # -- Setting that allows fast rebuilding of replicas using the checksum of snapshot disk files. Before enabling this setting, you must set the snapshot-data-integrity value to "enable" or "fast-check".
  fastReplicaRebuildEnabled: ~
  # -- Number of seconds that an HTTP client waits for a response from a File Sync server before considering the connection to have failed.
  replicaFileSyncHttpClientTimeout: ~
  # -- Number of seconds that Longhorn allows for the completion of replica rebuilding and snapshot cloning operations.
  longGRPCTimeOut: ~
  # -- Log levels that indicate the type and severity of logs in Longhorn Manager. The default value is "Info". (Options: "Panic", "Fatal", "Error", "Warn", "Info", "Debug", "Trace")
  logLevel: ~
  # -- Setting that allows you to specify a backup compression method.
  backupCompressionMethod: ~
  # -- Maximum number of worker threads that can concurrently run for each backup.
  backupConcurrentLimit: ~
  # -- Maximum number of worker threads that can concurrently run for each restore operation.
  restoreConcurrentLimit: ~
  # -- Setting that allows you to enable the V1 Data Engine.
  v1DataEngine: ~
  # -- Setting that allows you to enable the V2 Data Engine, which is based on the Storage Performance Development Kit (SPDK). The V2 Data Engine is a preview feature and should not be used in production environments.
  v2DataEngine: ~
  # -- Setting that allows you to configure maximum huge page size (in MiB) for the V2 Data Engine.
  v2DataEngineHugepageLimit: ~
  # -- Number of millicpus on each node to be reserved for each Instance Manager pod when the V2 Data Engine is enabled. The default value is "1250".
  v2DataEngineGuaranteedInstanceManagerCPU: ~
  # -- Setting that allows scheduling of empty node selector volumes to any node.
  allowEmptyNodeSelectorVolume: ~
  # -- Setting that allows scheduling of empty disk selector volumes to any disk.
  allowEmptyDiskSelectorVolume: ~
  # -- Setting that allows Longhorn to periodically collect anonymous usage data for product improvement purposes. Longhorn sends collected data to the [Upgrade Responder](https://github.com/longhorn/upgrade-responder) server, which is the data source of the Longhorn Public Metrics Dashboard (https://metrics.longhorn.io). The Upgrade Responder server does not store data that can be used to identify clients, including IP addresses.
  allowCollectingLonghornUsageMetrics: ~
  # -- Setting that temporarily prevents all attempts to purge volume snapshots.
  disableSnapshotPurge: ~
  # -- Maximum snapshot count for a volume. The value should be between 2 to 250
  snapshotMaxCount: ~
  # -- Setting that allows you to configure the log level of the SPDK target daemon (spdk_tgt) of the V2 Data Engine.
  v2DataEngineLogLevel: ~
  # -- Setting that allows you to configure the log flags of the SPDK target daemon (spdk_tgt) of the V2 Data Engine.
  v2DataEngineLogFlags: ~
  # -- Setting that freezes the filesystem on the root partition before a snapshot is created.
  freezeFilesystemForSnapshot: ~
  # -- Setting that automatically cleans up the snapshot when the backup is deleted.
  autoCleanupSnapshotWhenDeleteBackup: ~
  # -- Turn on logic to detect and move RWX volumes quickly on node failure.
  rwxVolumeFastFailover: ~

privateRegistry:
  # -- Setting that allows you to create a private registry secret.
  createSecret: ~
  # -- URL of a private registry. When unspecified, Longhorn uses the default system registry.
  registryUrl: ~
  # -- User account used for authenticating with a private registry.
  registryUser: ~
  # -- Password for authenticating with a private registry.
  registryPasswd: ~
  # -- Kubernetes secret that allows you to pull images from a private registry. This setting applies only when creation of private registry secrets is enabled. You must include the private registry name in the secret name.
  registrySecret: ~

longhornManager:
  log:
    # -- Format of Longhorn Manager logs. (Options: "plain", "json")
    format: plain
  # -- PriorityClass for Longhorn Manager.
  priorityClass: *defaultPriorityClassNameRef
  # -- Toleration for Longhorn Manager on nodes allowed to run Longhorn components.
  tolerations: []
  ## If you want to set tolerations for Longhorn Manager DaemonSet, delete the `[]` in the line above
  ## and uncomment this example block
  # - key: "key"
  #   operator: "Equal"
  #   value: "value"
  #   effect: "NoSchedule"
  # -- Node selector for Longhorn Manager. Specify the nodes allowed to run Longhorn Manager.
  nodeSelector: {}
  ## If you want to set node selector for Longhorn Manager DaemonSet, delete the `{}` in the line above
  ## and uncomment this example block
  #  label-key1: "label-value1"
  #  label-key2: "label-value2"
  # -- Annotation for the Longhorn Manager service.
  serviceAnnotations: {}
  ## If you want to set annotations for the Longhorn Manager service, delete the `{}` in the line above
  ## and uncomment this example block
  #  annotation-key1: "annotation-value1"
  #  annotation-key2: "annotation-value2"

longhornDriver:
  # -- PriorityClass for Longhorn Driver.
  priorityClass: *defaultPriorityClassNameRef
  # -- Toleration for Longhorn Driver on nodes allowed to run Longhorn components.
  tolerations: []
  ## If you want to set tolerations for Longhorn Driver Deployer Deployment, delete the `[]` in the line above
  ## and uncomment this example block
  # - key: "key"
  #   operator: "Equal"
  #   value: "value"
  #   effect: "NoSchedule"
  # -- Node selector for Longhorn Driver. Specify the nodes allowed to run Longhorn Driver.
  nodeSelector: {}
  ## If you want to set node selector for Longhorn Driver Deployer Deployment, delete the `{}` in the line above
  ## and uncomment this example block
  #  label-key1: "label-value1"
  #  label-key2: "label-value2"

longhornUI:
  # -- Replica count for Longhorn UI.
  replicas: 2
  # -- PriorityClass for Longhorn UI.
  priorityClass: *defaultPriorityClassNameRef
  # -- Toleration for Longhorn UI on nodes allowed to run Longhorn components.
  tolerations: []
  ## If you want to set tolerations for Longhorn UI Deployment, delete the `[]` in the line above
  ## and uncomment this example block
  # - key: "key"
  #   operator: "Equal"
  #   value: "value"
  #   effect: "NoSchedule"
  # -- Node selector for Longhorn UI. Specify the nodes allowed to run Longhorn UI.
  nodeSelector: {}
  ## If you want to set node selector for Longhorn UI Deployment, delete the `{}` in the line above
  ## and uncomment this example block
  #  label-key1: "label-value1"
  #  label-key2: "label-value2"

ingress:
  # -- Setting that allows Longhorn to generate ingress records for the Longhorn UI service.
  enabled: true

  # -- IngressClass resource that contains ingress configuration, including the name of the Ingress controller.
  # ingressClassName can replace the kubernetes.io/ingress.class annotation used in earlier Kubernetes releases.
  ingressClassName: longhorn-storage-ingress

  # -- Hostname of the Layer 7 load balancer.
  host: longhorn.dongna.com

  # -- Setting that allows you to enable TLS on ingress records.
  tls: false

  # -- Setting that allows you to enable secure connections to the Longhorn UI service via port 443.
  secureBackends: false

  # -- TLS secret that contains the private key and certificate to be used for TLS. This setting applies only when TLS is enabled on ingress records.
  tlsSecret: longhorn.local-tls

  # -- Default ingress path. You can access the Longhorn UI by following the full ingress path {{host}}+{{path}}.
  path: /

  # -- Ingress path type. To maintain backward compatibility, the default value is "ImplementationSpecific".
  pathType: ImplementationSpecific

  ## If you're using kube-lego, you will want to add:
  ## kubernetes.io/tls-acme: true
  ##
  ## For a full list of possible ingress annotations, please see
  ## ref: https://github.com/kubernetes/ingress-nginx/blob/master/docs/annotations.md
  ##
  ## If tls is set to true, annotation ingress.kubernetes.io/secure-backends: "true" will automatically be set
  # -- Ingress annotations in the form of key-value pairs.
  annotations:
  #  kubernetes.io/ingress.class: nginx
  #  kubernetes.io/tls-acme: true

  # -- Secret that contains a TLS private key and certificate. Use secrets if you want to use your own certificates to secure ingresses.
  secrets:
  ## If you're providing your own certificates, please use this to add the certificates as secrets
  ## key and certificate should start with -----BEGIN CERTIFICATE----- or
  ## -----BEGIN RSA PRIVATE KEY-----
  ##
  ## name should line up with a tlsSecret set further up
  ## If you're using kube-lego, this is unneeded, as it will create the secret for you if it is not set
  ##
  ## It is also possible to create and manage the certificates outside of this helm chart
  ## Please see README.md for more information
  # - name: longhorn.local-tls
  #   key:
  #   certificate:

# -- Setting that allows you to enable pod security policies (PSPs) that allow privileged Longhorn pods to start. This setting applies only to clusters running Kubernetes 1.25 and earlier, and with the built-in Pod Security admission controller enabled.
enablePSP: false

# -- Specify override namespace, specifically this is useful for using longhorn as sub-chart and its release namespace is not the `longhorn-system`.
namespaceOverride: "storage"

# -- Annotation for the Longhorn Manager DaemonSet pods. This setting is optional.
annotations: {}

serviceAccount:
  # -- Annotations to add to the service account
  annotations: {}

metrics:
  serviceMonitor:
    # -- Setting that allows the creation of a Prometheus ServiceMonitor resource for Longhorn Manager components.
    enabled: false
    # -- Additional labels for the Prometheus ServiceMonitor resource.
    additionalLabels: {}
    # -- Annotations for the Prometheus ServiceMonitor resource.
    annotations: {}
    # -- Interval at which Prometheus scrapes the metrics from the target.
    interval: ""
    # -- Timeout after which Prometheus considers the scrape to be failed.
    scrapeTimeout: ""
    # -- Configures the relabeling rules to apply the target’s metadata labels. See the [Prometheus Operator
    # documentation](https://prometheus-operator.dev/docs/api-reference/api/#monitoring.coreos.com/v1.Endpoint) for
    # formatting details.
    relabelings: []
    # -- Configures the relabeling rules to apply to the samples before ingestion. See the [Prometheus Operator
    # documentation](https://prometheus-operator.dev/docs/api-reference/api/#monitoring.coreos.com/v1.Endpoint) for
    # formatting details.
    metricRelabelings: []

## openshift settings
openshift:
  # -- Setting that allows Longhorn to integrate with OpenShift.
  enabled: false
  ui:
    # -- Route for connections between Longhorn and the OpenShift web console.
    route: "longhorn-ui"
    # -- Port for accessing the OpenShift web console.
    port: 443
    # -- Port for proxy that provides access to the OpenShift web console.
    proxy: 8443

# -- Setting that allows Longhorn to generate code coverage profiles.
enableGoCoverDir: false

```
**NOTE: <br>
** - Do ta chưa cài đặt `HAproxy` và `nginx-ingress` nên tạm thời ta expose service bằng cách dùng Node Port** <br>

- Trên các `Worker Node `, cài đặt và start `open-iscsi` để `Worker Node` có thể mount được phân vùng từ Longhorn Storage
```
sudo yum -y install iscsi-initiator-utils
sudo systemctl start iscsid.service
sudo systemctl enable iscsid.service
```

- Trên `local machine`, cài đặt Longhorn Storage
```
helm install longhorn-storage -f values-longhorn.yaml longhorn --namespace storage
```

- Output:
```
[sysadmin@cicd ~]$ kubectl -n storage get pods
NAME                                                         READY   STATUS    RESTARTS   AGE
csi-attacher-75588bff58-5sbpj                                1/1     Running   5          11h
csi-attacher-75588bff58-7qjnf                                1/1     Running   0          11h
csi-attacher-75588bff58-d4j2z                                1/1     Running   0          11h
csi-provisioner-669c8cc698-7xgn2                             1/1     Running   4          11h
csi-provisioner-669c8cc698-fnvh7                             1/1     Running   0          11h
csi-provisioner-669c8cc698-tp9dd                             1/1     Running   0          11h
csi-resizer-5c88bfd4cf-bhbl7                                 1/1     Running   4          11h
csi-resizer-5c88bfd4cf-hlntt                                 1/1     Running   0          11h
csi-resizer-5c88bfd4cf-t27rg                                 1/1     Running   0          11h
csi-snapshotter-69f8bc8dcf-5zpd4                             1/1     Running   0          11h
csi-snapshotter-69f8bc8dcf-pp5zg                             1/1     Running   1          11h
csi-snapshotter-69f8bc8dcf-tqtwp                             1/1     Running   3          11h
engine-image-ei-d4c780c6-k4wvn                               1/1     Running   0          11h
engine-image-ei-d4c780c6-pf8qt                               1/1     Running   0          11h
engine-image-ei-d4c780c6-xn6s7                               1/1     Running   0          11h
instance-manager-e-35ef3652                                  1/1     Running   0          11h
instance-manager-e-4ae5bd44                                  1/1     Running   0          <invalid>
instance-manager-e-96f5b164                                  1/1     Running   0          <invalid>
instance-manager-r-20ce7ce0                                  1/1     Running   0          <invalid>
instance-manager-r-313a0032                                  1/1     Running   0          11h
instance-manager-r-33c7edc7                                  1/1     Running   0          <invalid>
longhorn-csi-plugin-48z2j                                    2/2     Running   0          11h
longhorn-csi-plugin-d2gj5                                    2/2     Running   0          11h
longhorn-csi-plugin-rljhz                                    2/2     Running   0          11h
longhorn-driver-deployer-6d567db9f-fp4b6                     1/1     Running   0          11h
longhorn-manager-6vvg8                                       1/1     Running   0          11h
longhorn-manager-7qk6v                                       1/1     Running   0          11h
longhorn-manager-8fb8x                                       1/1     Running   1          11h
longhorn-ui-5dd96c9699-22hqq                                 1/1     Running   0          11h
nfs-storage-delete-nfs-client-provisioner-745bd8cb9b-fvbsl   1/1     Running   0          18h
nfs-storage-delete-nfs-client-provisioner-745bd8cb9b-wwcln   1/1     Running   1          18h
nfs-storage-delete-nfs-client-provisioner-745bd8cb9b-zzn2n   1/1     Running   1          18h
nfs-storage-retain-nfs-client-provisioner-7bcb4c965b-cd5nq   1/1     Running   1          18h
nfs-storage-retain-nfs-client-provisioner-7bcb4c965b-m97mb   1/1     Running   0          18h
nfs-storage-retain-nfs-client-provisioner-7bcb4c965b-r4fp9   1/1     Running   0          18h
share-manager-pvc-706cc6b1-4840-45f1-bd54-13a3053ccb08       1/1     Running   0          <invalid>
```

Longhorn UI có thể access qua IP của các K8s Nodes: http://192.168.10.11:30888/
![image](https://github.com/nguyenanhdongvn/Document/assets/90097692/d7c10722-b27b-46ef-b277-f5732d80089e)
![image](https://github.com/nguyenanhdongvn/Document/assets/90097692/6226287b-b2c2-4a2b-b9fe-dd86a9570d5a)

# Tạo Longhorn SC trên K8s
Tạo 2 yaml file `longhorn-storageclass-delete.yaml` và `longhorn-storageclass-retain.yaml` tương ứng với 2 loại reclaim policy là `delete` và `retain`

- $HOME/k8s/longhorn-storage/longhorn-storageclass-delete.yaml
```
cat <<EOF >> $HOME/k8s/longhorn-storage/longhorn-storageclass-delete.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn-storage-delete
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  fsType: "ext4"
EOF
```

- $HOME/k8s/longhorn-storage/longhorn-storageclass-retain.yaml
```
cat <<EOF >> $HOME/k8s/longhorn-storage/longhorn-storageclass-retain.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn-storage-retain
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  fsType: "ext4"
EOF
```

Tạo 2 SC từ 2 yaml file bên trên
```
kubectl apply -f $HOME/k8s/longhorn-storage/longhorn-storageclass-delete.yaml
kubectl apply -f $HOME/k8s/longhorn-storage/longhorn-storageclass-retain.yaml
```

Output
```
kubectl get sc
NAME                                PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
dongna-nfs-delete                   dongna-nfs-storage-delete-provisioner   Delete          Immediate           true                   18h
dongna-nfs-retain                   dongna-nfs-storage-retain-provisioner   Retain          Immediate           true                   18h
longhorn (default)                  driver.longhorn.io                      Delete          Immediate           true                   11h
longhorn-storage-delete (default)   driver.longhorn.io                      Delete          Immediate           true                   11h
longhorn-storage-retain             driver.longhorn.io                      Retain          Immediate           true                   11h

```

# Tạo PV/PVC và tạo POD để sử dụng 2 longhorn SC vừa tạo
Tạo 2 yaml file cho 2 PVC
- $HOME/k8s/longhorn-storage/longhorn-pvc-delete.yaml
```
cat <<EOF >> $HOME/k8s/longhorn-storage/longhorn-pvc-delete.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-pvc-delete
spec:
  accessModes:
    #- ReadWriteOnce
    - ReadWriteMany
  storageClassName: longhorn-storage-delete
  resources:
    requests:
      storage: 2Gi
EOF
```

- $HOME/k8s/longhorn-storage/longhorn-pvc-delete.yaml
```
cat <<EOF >> $HOME/k8s/longhorn-storage/longhorn-pvc-retain.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-pvc-retain
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-storage-retain
  resources:
    requests:
      storage: 2Gi
EOF
```

Tạo 2 PVC bằng yaml file bên trên
```
kubectl apply -f $HOME/k8s/longhorn-storage/longhorn-pvc-delete.yaml
kubectl apply -f $HOME/k8s/longhorn-storage/longhorn-pvc-retain.yaml
```

Output
```
kubectl get pvc
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
longhorn-pvc-delete   Bound    pvc-71b628cc-8492-44b6-a857-2fa6a2f0531a   2Gi        RWX            longhorn-storage-delete   30s
longhorn-pvc-retain   Bound    pvc-b5beccd7-e6be-40b7-972b-44ab8f7400de   2Gi        RWO            longhorn-storage-retain   8s
```

Như vậy 2 PVC đều đã được longhorn storage class cấp PV cho rồi (STATUS là Bound)<br>
Tạo 2 yaml file cho 2 POD sử dụng 2 PVC bên trên 
- $HOME/k8s/longhorn-storage/test-pod-longhorn-delete.yaml
```
cat <<EOF >> $HOME/k8s/longhorn-storage/test-pod-longhorn-delete.yaml
kind: Pod
apiVersion: v1
metadata:
  name: pod-longhorn-delete
spec:
  volumes:
        - name: longhorn-pvc-delete
          persistentVolumeClaim:
            claimName: longhorn-pvc-delete
  containers:
    - name: my-container
      volumeMounts:
        - name: longhorn-pvc-delete # This is the name of the volume we set at the pod level
          mountPath: /var/simple # Where to mount this directory in our container

      # Now that we have a directory mounted at /var/simple, let's
      # write to a file inside it!
      image: alpine
      command: ["/bin/sh"]
      args: ["-c", "while true; do date >> /var/simple/file.txt; sleep 5; done"]
EOF
```

- $HOME/k8s/longhorn-storage/test-pod-longhorn-retain.yaml
```
cat <<EOF >> $HOME/k8s/longhorn-storage/test-pod-longhorn-retain.yaml
kind: Pod
apiVersion: v1
metadata:
  name: pod-longhorn-retain
spec:
  volumes:
        - name: longhorn-pvc-retain
          persistentVolumeClaim:
            claimName: longhorn-pvc-retain
  containers:
    - name: my-container
      volumeMounts:
        - name: longhorn-pvc-retain # This is the name of the volume we set at the pod level
          mountPath: /var/simple # Where to mount this directory in our container

      # Now that we have a directory mounted at /var/simple, let's
      # write to a file inside it!
      image: alpine
      command: ["/bin/sh"]
      args: ["-c", "while true; do date >> /var/simple/file.txt; sleep 5; done"]
EOF
```

Tạo 2 POD từ 2 yaml file bên trên để sử dụng 2 PVC đã tạo
```
kubectl apply -f $HOME/k8s/longhorn-storage/test-pod-longhorn-delete.yaml
kubectl apply -f $HOME/k8s/longhorn-storage/test-pod-longhorn-retain.yaml
```

Output
```
kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
pod-longhorn-delete            1/1     Running   0          85s
pod-longhorn-retain            1/1     Running   0          81s
```

Check trên Longhorn UI để thấy phân vùng được tạo và replicas theo cấu hình đã tạo (2 replicas):

![image](https://github.com/user-attachments/assets/c8b408f9-e72e-4188-89a1-315e61190231)

**NOTE:**<br>
Để pod trên các worker node có thể mount được longhorn thì các worker node phải cài nfs-client



Longhorn best practice<br>
https://www.reddit.com/r/kubernetes/comments/1cojq13/longhorn_best_practices/<br>
https://longhorn.io/docs/1.7.1/best-practices/
