# Giới thiệu
Như đã nói trong các phần trước, Ta sẽ cài NFS-Server lên server `rancher` (vì server `rancher` load khá thấp và đỡ phải tạo một server riêng cho NFS). Ta cũng đã tạo sẵn phân vùng /data2 để dành cho NFS-Server.<br>
Reclaim policy có 2 loại là delete và retain, hiểu đơn giản đó là cấu hình chính sách xử lý các phân vùng lưu trữ khi sau khi xóa PVC.
- `delete`: Khi xóa PVC trên K8s thì hệ thống cũng tự động xóa PV tương ứng và đồng thời hỗ trợ xóa luôn phân vùng lưu trên thiết bị lưu trữ mà gán với PV đó
- `retain`: Khi xóa PVC trên K8s thì phân vùng lữu trữ trên thiết bị lưu trữ sẽ không tự động bị xóa đi.

# Các bước cài đặt trong bài lab này có tóm tắt lại như sau:
- Cài đặt NFS Server (trên server `rancher`)
- Cài đặt NFS Client (trên các Worker Node và các Master Node)
- Cài đặt kubect và helm3 (trên server `rancher`)
- Cài đặt Storage Class
- Tạo PVC để test

# Cài đặt NFS storage cho K8S
Trong môi trường Production thì một số loại storage đã hỗ trợ sẵn NFS Server, nghĩa là có thể output ra cho bạn một phân vùng share để sử dụng. Việc quản lý lỗi, quản lý tính sẵn sàng sẽ được thực hiện trên thiết bị Storage này. Nhưng nếu bạn cài đặt NFS Server để share cho K8s sử dụng thì lúc đó nó sẽ là một NFS Server chạy standalone và được coi là single point of failure vì khi NFS Server này down thì gây ảnh hưởng dịch vụ. Trong phạm vi bài lab này do không có thiết bị Storage chuyên dụng do đó ta sẽ cài một NFS Server để sử dụng.

# Cài đặt NFS-Server trên server `rancher`
Việc cài đặt này khá đơn giản, mình sẽ cài trên OS là Centos. Đầu tiên cần tạo thư mục để share và cài NFS Server:
```
sudo su -

#NFS Server installation
yum install nfs-utils -y

#Create shared folder
mkdir -p /data2/delete
mkdir -p /data2/retain

#Change folder
chmod -R 755 /data2
 
systemctl enable rpcbind
systemctl enable nfs-server

systemctl start rpcbind
systemctl start nfs-server
```

Cấu hình file /etc/exports để share quyền cho các node theo format sau mục đích là để cho phép các server trong dải ip 192.168.10.0/24 có quyền Read/Write vào 2 thư mục /data2/delete và /data2/retain:
```
cat <<EOF >> /etc/exports
/data2/retain    192.168.10.0/24(rw,sync,no_root_squash,no_all_squash)
/data2/delete    192.168.10.0/24(rw,sync,no_root_squash,no_all_squash)
EOF
```

Restart NFS Server để update cấu hình mới:
```
systemctl restart nfs-server
```

Check xem 2 thư mục trên đã được share chưa bằng lệnh sau:
```
showmount -e 192.168.10.19
```
Output:
```
Export list for 192.168.10.19:
/data2/delete 192.168.10.0/24
/data2/retain 192.168.10.0/24
```

# Cài đặt NFS Client trên các Worker Node
Cài đặt NFS Client trên tất cả các Worker Node để khi POD được tạo trên Worker Node có sử dụng NFS Storage Class thì Worker Node đó có thể mount được phân vùng NFS đã được share bởi NFS Server.
```
sudo yum install nfs-utils -y
```

Check xem từ các Worker Node và Master Node đã thấy được các folder được share chưa:
```
showmount -e 192.168.10.19
```
Output:
```
Export list for 192.168.10.19:
/data2/delete 192.168.10.0/24
/data2/retain 192.168.10.0/24
```

# Cài đặt NFS Storage Class trên K8S
**_NOTE_: ban đầu ta cài đặt Kubernetes Cluster bằng Kubespray từ server `cicd`. Toàn bộ phần cài đặt khác cho cụm K8s ta cũng sẽ cài từ server `cicd` để quản lý tập trung cấu hình cài đặt dễ dàng hơn. Mình sẽ cài đặt NFS Storage Class qua helm chart, do đó cũng cần phải cài helm lên server `cicd`**

## Cài đặt kubectl và helm trên server `cicd`
Do phiên bản Kubernetes đang cài trong series LAB này là v1.30.4 nên ta cũng sẽ cài kubectl cùng phiên bản:

```
curl -LO https://dl.k8s.io/release/v1.30.4/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

Cấu hình kubectl để kết nối tới cụm K8S của mình:
```
mkdir -p $HOME/.kube
scp master1:~/.kube/config  $HOME/.kube/
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Sửa file config, tham số "server: https://127.0.0.1:6443" thành "server: https://192.168.10.11:6443"
```
sed -i 's/127.0.0.1/192.168.10.11/g' $HOME/.kube/config
```

Giờ thử kiếm tra kết nối bằng lệnh kubect get node xem đã kết nối ok hay chưa:

```
kubectl get node
```
output
```
NAME             STATUS   ROLES                  AGE   VERSION
master1   Ready    control-plane,master   19h   v1.20.7
master2   Ready    control-plane,master   19h   v1.20.7
master3   Ready    control-plane,master   19h   v1.20.7
worker1   Ready    <none>                 19h   v1.20.7
worker2   Ready    <none>                 19h   v1.20.7
worker3   Ready    <none>                 19h   v1.20.7
```

Tiếp tục cài đặt helm chart
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
sudo chmod 700 get_helm.sh
./get_helm.sh
```

Do helm sẽ mặc định dùng chung config của kubectl, nên ở bước này không cần cấu hình gì thêm
```
helm list
```
Output
```
NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART   APP VERSION
```

Kết quả như trên là helm kết nối K8S ok

## Cài đặt NFS Storage Class
Tạo thư mục cài đặt để lưu helm-chart và các file config:
```
mkdir /home/sysadmin/kubernetes_installation/nfs-storage
cd /home/sysadmin/kubernetes_installation/nfs-storage
```

Download helm chart nfs-client-provisioner về để cài offline:
```
helm repo add stable https://charts.helm.sh/stable
helm search repo nfs-client-provisioner
helm pull stable/nfs-client-provisioner --version 1.2.11
tar -xzf nfs-client-provisioner-1.2.11.tgz
```

Trước khi cài đặt cần thay đổi tham số mặc định của helm chart này. Mình sẽ tạo 2 storage class khác nhau tương ứng với `reclaim policy` là `delete` và `retain`--> Cần 2 file value tương ứng để cài đặt 2 storage class này. Tạo file value cho storage class có `reclaim policy` là `delete` và `retain`:
```
cp /home/sysadmin/kubernetes_installation/nfs-storage/nfs-client-provisioner/values.yaml /home/sysadmin/kubernetes_installation/nfs-storage/values-nfs-delete.yaml
cp /home/sysadmin/kubernetes_installation/nfs-storage/nfs-client-provisioner/values.yaml /home/sysadmin/kubernetes_installation/nfs-storage/values-nfs-retain.yaml
```

Thay đổi các tham số trong file `values-nfs-delete.yaml` như sau:
```
# Default values for nfs-client-provisioner.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 3
strategyType: Recreate

image:
  #repository: quay.io/external_storage/nfs-client-provisioner
  #tag: v3.1.0-k8s1.11
  repository: rkevin/nfs-subdir-external-provisioner
  tag: fix-k8s-1.20
  pullPolicy: IfNotPresent

nfs:
  server: 192.168.10.19
  path: /data2/delete
  mountOptions:

# For creating the StorageClass automatically:
storageClass:
  create: true

  # Set a provisioner name. If unset, a name will be generated.
  provisionerName: dongna-nfs-storage-delete-provisioner

  # Set StorageClass as the default StorageClass
  # Ignored if storageClass.create is false
  defaultClass: false

  # Set a StorageClass name
  # Ignored if storageClass.create is false
  name: dongna-nfs-delete

  # Allow volume to be expanded dynamically
  allowVolumeExpansion: true

  # Method used to reclaim an obsoleted volume
  reclaimPolicy: Delete

  # When set to false your PVs will not be archived by the provisioner upon deletion of the PVC.
  archiveOnDelete: false

  # Set access mode - ReadWriteOnce, ReadOnlyMany or ReadWriteMany
  accessModes: ReadWriteOnce

## For RBAC support:
rbac:
  # Specifies whether RBAC resources should be created
  create: true

# If true, create & use Pod Security Policy resources
# https://kubernetes.io/docs/concepts/policy/pod-security-policy/
podSecurityPolicy:
  enabled: false

## Set pod priorityClassName
# priorityClassName: ""

serviceAccount:
  # Specifies whether a ServiceAccount should be created
  create: true

  # The name of the ServiceAccount to use.
  # If not set and create is true, a name is generated using the fullname template
  name:

resources: {}
  # limits:
  #  cpu: 100m
  #  memory: 128Mi
  # requests:
  #  cpu: 100m
  #  memory: 128Mi

nodeSelector: {}

tolerations: []

affinity: {}
```

Thay đổi các tham số trong file `values-nfs-retain.yaml` như sau:
```
# Default values for nfs-client-provisioner.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 3
strategyType: Recreate

image:
  #repository: quay.io/external_storage/nfs-client-provisioner
  #tag: v3.1.0-k8s1.11
  repository: rkevin/nfs-subdir-external-provisioner
  tag: fix-k8s-1.20
  pullPolicy: IfNotPresent

nfs:
  server: 192.168.10.19
  path: /data2/retain
  mountOptions:

# For creating the StorageClass automatically:
storageClass:
  create: true

  # Set a provisioner name. If unset, a name will be generated.
  provisionerName: dongna-nfs-storage-retain-provisioner

  # Set StorageClass as the default StorageClass
  # Ignored if storageClass.create is false
  defaultClass: false

  # Set a StorageClass name
  # Ignored if storageClass.create is false
  name: dongna-nfs-retain

  # Allow volume to be expanded dynamically
  allowVolumeExpansion: true

  # Method used to reclaim an obsoleted volume
  reclaimPolicy: Retain

  # When set to false your PVs will not be archived by the provisioner upon deletion of the PVC.
  archiveOnDelete: true

  # Set access mode - ReadWriteOnce, ReadOnlyMany or ReadWriteMany
  accessModes: ReadWriteOnce

## For RBAC support:
rbac:
  # Specifies whether RBAC resources should be created
  create: true

# If true, create & use Pod Security Policy resources
# https://kubernetes.io/docs/concepts/policy/pod-security-policy/
podSecurityPolicy:
  enabled: false

## Set pod priorityClassName
# priorityClassName: ""

serviceAccount:
  # Specifies whether a ServiceAccount should be created
  create: true

  # The name of the ServiceAccount to use.
  # If not set and create is true, a name is generated using the fullname template
  name:

resources: {}
  # limits:
  #  cpu: 100m
  #  memory: 128Mi
  # requests:
  #  cpu: 100m
  #  memory: 128Mi

nodeSelector: {}

tolerations: []

affinity: {}
```

Tạo một namespace riêng cho phần storage để dễ quản lý rồi cài 2 Storage Class bằng helm

```
kubectl create namespace "storage"
helm install nfs-storage-delete --namespace storage -f /home/sysadmin/kubernetes_installation/nfs-storage/values-nfs-delete.yaml nfs-client-provisioner
helm install nfs-storage-retain --namespace storage -f /home/sysadmin/kubernetes_installation/nfs-storage/values-nfs-retain.yaml nfs-client-provisioner
```

Kết quả:
```
kubectl get pods -n storage
```
Output
```
NAME                                                         READY   STATUS    RESTARTS   AGE
nfs-storage-delete-nfs-client-provisioner-74b99ddd9b-bg7jm   1/1     Running   0          69s
nfs-storage-delete-nfs-client-provisioner-74b99ddd9b-g9zsw   1/1     Running   0          69s
nfs-storage-delete-nfs-client-provisioner-74b99ddd9b-tzhxg   1/1     Running   0          69s
nfs-storage-retain-nfs-client-provisioner-99cdf9f5d-dp6zc    1/1     Running   0          90s
nfs-storage-retain-nfs-client-provisioner-99cdf9f5d-fh668    1/1     Running   0          90s
nfs-storage-retain-nfs-client-provisioner-99cdf9f5d-v5n8v    1/1     Running   0          90s
```
```
kubectl get sc
```
Output
```
NAME                PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
dongna-nfs-delete   dongna-nfs-storage-delete-provisioner   Delete          Immediate           true                   2d1h
dongna-nfs-retain   dongna-nfs-storage-retain-provisioner   Retain          Immediate           true                   2d1h
```

# Kiểm tra nfs-storageclass bằng cách tạo thử pvc
Giờ tạo 1 PVC xem SC nfs-storageclass nó có tự động tạo PV không, tạo file config cho PVC có reclaim policy là delete như sau, lưu ý tham số storageClassName: dongna-nfs-delete được gán đúng với tên SC mình đã tạo ở bước trước:

Tạo yaml file cho PVC
```
cat <<EOF >> /home/sysadmin/kubernetes_installation/nfs-storage/test-pvc-delete.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc-delete
spec:
  storageClassName: dongna-nfs-delete
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
EOF
```

Tạo PVC bằng yaml file ở trên vừa tạo
```
kubectl apply -f /home/sysadmin/kubernetes_installation/nfs-storage/test-pvc-delete.yaml
```

```
kubectl get pvc
```
Output
```
NAME              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS        AGE
test-pvc-delete   Pending                                      dongna-nfs-delete   6s
```

**NOTE:**<br>
Tại sao PVC vẫn ở trạng thái `Pending` mà không phải là `Bound`? <br> 
=> Đây là một known-issue của Kubernetes từ các phiên bản 1.20 trở về sau, bạn có thể tham khảo thêm về solution tại [đây](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/pull/26#issuecomment-743745469)

**Solution:** <br>
Chỉnh sửa 2 file value `values-nfs-delete.yaml` và `values-nfs-retain.yaml`
```
vim /home/sysadmin/kubernetes_installation/nfs-storage/values-nfs-delete.yaml
vim /home/sysadmin/kubernetes_installation/nfs-storage/values-nfs-retain.yaml
```

Thay thế value của repository và tag như sau
```
image:
  #  repository: quay.io/external_storage/nfs-client-provisioner
  #  tag: v3.1.0-k8s1.11
  repository: rkevin/nfs-subdir-external-provisioner
  tag: fix-k8s-1.20
  pullPolicy: IfNotPresent
```
Sau đó upgrade lại helm chart `nfs-storage-delete` và `nfs-storage-retain` với 2 file value đã được chỉnh sửa

```
helm upgrade nfs-storage-delete stable/nfs-client-provisioner -f values-nfs-delete.yaml -n storage
helm upgrade nfs-storage-retain stable/nfs-client-provisioner -f values-nfs-retain.yaml -n storage
```

Sau khi upgrade 2 helm chart `nfs-storage-delete` và `nfs-storage-retain` ta check lại xem pvc đã được bound chưa

```
kubectl get pvc
```
Output
```
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
test-pvc-delete   Bound    pvc-e0f829ee-1436-4787-9ae3-00d53871acb5   10Mi       RWO            dongna-nfs-delete   78m
```

Bây giờ chắc các bạn cũng nóng lòng muốn biết khi tạo PVC có reclaim policy là delete hay retain thì khác nhau ra sao. Ok, xóa hết cờ đi làm lại, sau khi test SC đã hoạt động ok. Xóa PVC bên trên đi:
```
kubectl delete pvc test-pvc-delete
```

Tạo thêm một yaml file cho PVC có relaim policy là retain:
```
cat <<EOF >> /home/sysadmin/kubernetes_installation/nfs-storage/test-pvc-retain.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc-retain
spec:
  storageClassName: dongna-nfs-retain
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
EOF
```

Giờ tạo 2 PVC, một qua SC có relaim policy là `delete`, một qua SC có reclaim policy là `retain`:
```
kubectl apply -f /home/sysadmin/kubernetes_installation/nfs-storage/test-pvc-delete.yaml
kubectl apply -f /home/sysadmin/kubernetes_installation/nfs-storage/test-pvc-retain.yaml
```

```
kubectl get pvc
```
Output
```
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
test-pvc-delete   Bound    pvc-96f5330a-c141-4623-b8c6-e56e05ff1832   10Mi       RWO            dongna-nfs-delete   6m5s
test-pvc-retain   Bound    pvc-228aa21d-a480-4d4c-aa90-1329026ee8fe   10Mi       RWO            dongna-nfs-retain   6s
```

Như vậy PVC `test-pvc-delete` và `test-pvc-retain đã được bound vào 2 PV như trên, 2 PV này tương ứng với 2 phân vùng được tạo trên NFS-Server. <br>
Login vào `rancher` server (NFS-Server) để kiểm tra phân vùng tạo trên NFS-Server ra sao:

```
sysadmin@rancher:~$ tree /data2
/data2
├── delete
│   └── default-test-pvc-delete-pvc-81132005-b7c3-4fc2-a29f-c351ab5dbc9c
└── retain
    └── default-test-pvc-retain-pvc-edeaadf9-29ad-4186-9193-fe052a73449e
```

Như vậy ta thấy các phân vùng được tạo ở đúng các thư mục như cấu hình SC<br>
Login vào `cicd` server, thử xóa cả 2 PVC xem chuyện gì xảy ra:
```
kubectl delete pvc test-pvc-delete
kubectl delete pvc test-pvc-retain
```

```
kubectl get pv,pvc
```
Output
```
sysadmin@cicd:~/kubernetes_installation/nfs-storage$ kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                     STORAGECLASS        VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-edeaadf9-29ad-4186-9193-fe052a73449e   10Mi       RWO            Retain           Released   default/test-pvc-retain   dongna-nfs-retain   <unset>                          5m56s
```

Login vào `rancher` server (NFS-Server) để kiểm tra:
```
sysadmin@rancher:~$ tree /data2
/data2
├── delete
└── retain
    └── default-test-pvc-retain-pvc-edeaadf9-29ad-4186-9193-fe052a73449e
```

Giờ thì các bạn đã rõ sự khác biệt.<br>
* Reclaim policy là `delete`, khi xóa PVC, PV cũng sẽ bị xóa => Phân vùng lưu trữ trên thiết bị storage cũng sẽ bị xóa luôn. <br>
* Reclaim policy là `retain`, khi xoá PVC => Phân vùng sẽ vẫn còn lưu trên thiết bị lưu trữ.
