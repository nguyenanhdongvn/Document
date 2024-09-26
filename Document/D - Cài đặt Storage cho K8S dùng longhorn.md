# Giới thiệu
Ý tưởng cài đặt longhorn hiểu đơn giản như sau:
- Cấu hình cho longhorn biết thư mục mặc định lưu data của longhorn `/data/longhorn-storage` trên các Worker Node 
- Longhorn sẽ chạy trên tất cả các Worker Node thỏa mãn điều kiện là có phân vùng lưu trữ như cấu hình bên trên
- Mỗi Worker Node sẽ được coi là một Node của longhorn, trên mỗi Node này có thể có 1 hoặc nhiều Disk mà ta có thể cấu hình thêm sau khi đã cài đặt longhorn

Ta sẽ có 2 phần:
- **Longhorn Storage:** Là storage quản lý thiết bị lưu trữ, nó có vai trò giống như NFS Server vậy
- **Longhorn Storage Class:** Là một object trên K8S đảm nhiệm việc nhận các yêu cầu tạo Volume trên K8S (PV/PVC) sau đó kết nối với Longhorn Storage để tạo ra phân vùng lưu trữ trên thiết bị lưu trữ

Các bước thực hiện trong bài lab này như sau:
- Chuẩn bị phân vùng lưu dữ liệu (đĩa) trên các Worker Node
- Cài đặt Longhorn Storage trên K8S bằng helm chart
- Tạo Longhorn SC trên K8S
- Test thử tạo PV/PVC và tạo Pod dùng Longhorn SC

**NOTE: xin nhắc lại là việc cài đặt đều được thực hiện trên server `cicd` để quản lý tập trung các file cấu hình cài đặt được dễ dàng**

# Cài đặt Longhorn Storage
## Tạo phân vùng lưu trữ
Ta sẽ tạo folder lưu dữ liệu của Longhorn là `/data/longhorn-storage` trên các Worker Node:
```
sudo mkdir -p /data/longhorn-storage
```

## Cài đặt Longhorn Storage
Tạo folder lưu helm chart và các file cấu hình Longhorn:
```bash
mkdir /home/sysadmin/kubernetes_installation/longhorn-storage
```

Add repo và download helm chart của Longhorn về folder đã tạo:
```
cd /home/sysadmin/kubernetes_installation/longhorn-storage
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm search repo longhorn
helm pull longhorn/longhorn --version 1.7.1
tar -xzf longhorn-1.7.1.tgz
```

Ở đây mình đang dùng bản 1.7.1, các bạn nếu cài version khác nên chú ý xem cấu hình có khác thì update tương ứng <br>
Sau bước trên mình đã down về và giải nén thư mục helm chart của longhorn ở thư mục `/home/sysadmin/kubernetes_installation/longhorn-storage` <br>
Copy file value mặc định của helmchart ra ngoài để tùy biến theo môi trường của mình: <br>
```
cp /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn/values.yaml /home/sysadmin/kubernetes_installation/longhorn-storage/values-longhorn.yaml
```
Sửa file values-longhorn.yaml và cập nhật một số tham số như sau:
```
service:
  ui:
    type: NodePort
    nodePort: 30888
  manager:
    type: ClusterIP
defaultSettings:    
  defaultDataPath: /data/longhorn-storage/
  replicaSoftAntiAffinity: true
  storageMinimalAvailablePercentage: 15
  upgradeChecker: false
  defaultReplicaCount: 2
  backupstorePollInterval: 500
  nodeDownPodDeletionPolicy: do-nothing
  guaranteedInstanceManagerCPU: 15

ingress:  
  enabled: true
  ingressClassName: longhorn-storage-ingress
  host: longhorn-ui.dongna.com

namespaceOverride: "storage"
```
**NOTE: <br>
Ở đây mình chưa động gì tới haproxy với nginx-ingress nên để expose service ra bên ngoài thì trước mắt tạm thời sẽ dùng Node Port<br>
Phải cài thêm và start open-iscsi cho các Worker Node để nó có thể mount được phân vùng từ longhorn storage**
```
sudo yum -y install iscsi-initiator-utils
sudo systemctl start iscsid.service
sudo systemctl enable iscsid.service
```

Trở lại `cicd` server, chạy helm install để cài đặt Longhorn:
```
helm install longhorn-storage -f values-longhorn.yaml longhorn --namespace storage
```

Output:
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

- /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-storageclass-delete.yaml
```
cat <<EOF >> /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-storageclass-delete.yaml
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

- /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-storageclass-retain.yaml
```
cat <<EOF >> /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-storageclass-retain.yaml
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
kubectl apply -f /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-storageclass-delete.yaml
kubectl apply -f /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-storageclass-retain.yaml
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
- /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-pvc-delete.yaml
```
cat <<EOF >> /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-pvc-delete.yaml
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

- /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-pvc-delete.yaml
```
cat <<EOF >> /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-pvc-retain.yaml
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
kubectl apply -f /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-pvc-delete.yaml
kubectl apply -f /home/sysadmin/kubernetes_installation/longhorn-storage/longhorn-pvc-retain.yaml
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
- /home/sysadmin/kubernetes_installation/longhorn-storage/test-pod-longhorn-delete.yaml
```
cat <<EOF >> /home/sysadmin/kubernetes_installation/longhorn-storage/test-pod-longhorn-delete.yaml
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

- /home/sysadmin/kubernetes_installation/longhorn-storage/test-pod-longhorn-retain.yaml
```
cat <<EOF >> /home/sysadmin/kubernetes_installation/longhorn-storage/test-pod-longhorn-retain.yaml
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
kubectl apply -f /home/sysadmin/kubernetes_installation/longhorn-storage/test-pod-longhorn-delete.yaml
kubectl apply -f /home/sysadmin/kubernetes_installation/longhorn-storage/test-pod-longhorn-retain.yaml
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
