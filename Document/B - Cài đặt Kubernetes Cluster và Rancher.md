# Tạo SSH keys và cấu hình SSH 
* Trên `local machine`, tạo cặp ssh key với defaul option
```
ssh-keygen
```

* Từ `local machine`, copy ssh public key tới các Destination Server sẽ cài K8S (nhập password của user dùng để access đến Destination Server khi được yêu cầu):
```
ssh-copy-id master1
ssh-copy-id master2
ssh-copy-id master3
ssh-copy-id worker1
ssh-copy-id worker2
ssh-copy-id worker3
```

* Tạo folder chứa kubespray
```
 mkdir $HOME/k8s/ && cd $HOME/k8s/
```

* Clone kubesrpay repo v2.26 về folder `k8s` vừa tạo
```
git clone https://github.com/kubernetes-sigs/kubespray.git --branch release-2.26
```

* Copy folder `sample` ra folder `dongna-cluster`
```
cp -rf $HOME/k8s/kubespray/inventory/sample $HOME/k8s/kubespray/inventory/dongna-cluster
```

* Cấu hình file inventory `inventory.ini` cho Ansible
```
cat <<EOF > $HOME/k8s/kubespray/inventory/dongna-cluster/inventory.ini
[all]
master1  ansible_host=192.168.10.11      ip=192.168.10.11
master2  ansible_host=192.168.10.12      ip=192.168.10.12
master3  ansible_host=192.168.10.13      ip=192.168.10.13
worker1  ansible_host=192.168.10.14      ip=192.168.10.14
worker2  ansible_host=192.168.10.15      ip=192.168.10.15
worker3  ansible_host=192.168.10.16      ip=192.168.10.16

[kube_control_plane]
master1
master2
master3

[etcd]
master1
master2
master3

[kube_node]
worker1
worker2
worker3

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
EOF
```

* Đổi CNI từ Calico sang Flannel (Optional)
```
sed -i 's/kube_network_plugin: calico/kube_network_plugin: flannel/g' $HOME/k8s/kubespray/inventory/dongna-cluster/group_vars/k8s_cluster/k8s-cluster.yml
```

* Chạy container đã cài sẵn Ansible để chạy Kubespray
```
docker run --rm -it --mount type=bind,source=$HOME/k8s/kubespray/inventory/dongna-cluster,dst=/inventory \
  --mount type=bind,source=$HOME/k8s/.ssh/id_rsa,dst=/root/.ssh/id_rsa \
  --mount type=bind,source=$HOME/k8s/.ssh/id_rsa,dst=$HOME/k8s/.ssh/id_rsa \
  quay.io/kubespray/kubespray:v2.26.0 bash
```

* Trong container chạy ansible command để cài đặt K8S cluster
```
ansible-playbook -i /inventory/inventory.ini cluster.yml --user=sysadmin --ask-pass --become --ask-become-pass
```

**NOTE:**
Khi gặp phải lỗi như sau
```
TASK [container-engine/runc : Download_file | Download item] *************************************************************************************************************************************************
fatal: [worker2]: FAILED! => {"attempts": 4, "censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
fatal: [master1]: FAILED! => {"attempts": 4, "censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
fatal: [master2]: FAILED! => {"attempts": 4, "censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
fatal: [worker1]: FAILED! => {"attempts": 4, "censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
fatal: [master3]: FAILED! => {"attempts": 4, "censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
FAILED - RETRYING: [worker3]: Download_file | Download item (4 retries left).
FAILED - RETRYING: [worker3]: Download_file | Download item (3 retries left).
FAILED - RETRYING: [worker3]: Download_file | Download item (2 retries left).
FAILED - RETRYING: [worker3]: Download_file | Download item (1 retries left).
fatal: [worker3]: FAILED! => {"attempts": 4, "censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}

NO MORE HOSTS LEFT *******************************************************************************************************************************************************************************************
```
Thì ta login các K8s Nodes và comment lại dòng "search default.svc.cluster.local svc.cluster.local" trong file /etc/resolv.conf
```
sudo sed -e '/search/s/^/#/g' -i /etc/resolv.conf
```
Sau đó chạy lại ansible command trong container để cài đặt lại K8s cluster

* Cấu hình kube config trên 3 Master Node để connect được đến K8S cluster
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* Kiểm tra kết quả sau khi cấu hình kube config
```
kubectl get nodes -o wide
NAME      STATUS   ROLES                  AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                 CONTAINER-RUNTIME
master1   Ready    control-plane,master   9m40s   v1.20.7   192.168.10.11   <none>        CentOS Linux 7 (Core)   3.10.0-1160.105.1.el7.x86_64   docker://19.3.15
master2   Ready    control-plane,master   9m13s   v1.20.7   192.168.10.12   <none>        CentOS Linux 7 (Core)   3.10.0-1160.105.1.el7.x86_64   docker://19.3.15
master3   Ready    control-plane,master   9m2s    v1.20.7   192.168.10.13   <none>        CentOS Linux 7 (Core)   3.10.0-1160.105.1.el7.x86_64   docker://19.3.15
worker1   Ready    <none>                 8m12s   v1.20.7   192.168.10.14   <none>        CentOS Linux 7 (Core)   3.10.0-1160.105.1.el7.x86_64   docker://19.3.15
worker2   Ready    <none>                 8m12s   v1.20.7   192.168.10.15   <none>        CentOS Linux 7 (Core)   3.10.0-1160.105.1.el7.x86_64   docker://19.3.15
worker3   Ready    <none>                 8m12s   v1.20.7   192.168.10.16   <none>        CentOS Linux 7 (Core)   3.10.0-1160.105.1.el7.x86_64   docker://19.3.15
```

# Cài đặt Rancher để quản lý K8S Cluster

* Trên `rancher` server, cài đặt Rancher bằng cách chạy Rancher container:
```
docker run --name rancher-server -d --restart=unless-stopped -p 6860:80 -p 6868:443 --privileged rancher/rancher:v2.9.2 
```
_**Note:**_<br>
Giữa Rancher và Kubernetes có bảng tương thích, ta có thể check trên trang chủ của Rancher tại [đây](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-9-2/) . Ví dụ: Kubernetes v1.30.2 tương thích với Rancher v2.9.2 <br>
Expose Rancher bằng port 6860 cho HTTP và 6868 cho HTTPS để sau này sẽ cho các connection này đi qua HAproxy

Khi gặp phải error sau [error](https://github.com/rancher/rancher/issues/44279#issuecomment-1993796373)
```
2024/09/22 12:12:33 [INFO] Waiting for server to become available: Get "https://127.0.0.1:6444/version?timeout=15m0s": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:35 [INFO] Waiting for server to become available: Get "https://127.0.0.1:6444/version?timeout=15m0s": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:37 [INFO] Waiting for server to become available: Get "https://127.0.0.1:6444/version?timeout=15m0s": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:39 [INFO] Waiting for server to become available: Get "https://127.0.0.1:6444/version?timeout=15m0s": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:41 [INFO] Waiting for server to become available: Get "https://127.0.0.1:6444/version?timeout=15m0s": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:43 [INFO] Waiting for server to become available: Get "https://127.0.0.1:6444/version?timeout=15m0s": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:45 [INFO] Waiting for server to become available: Get "https://127.0.0.1:6444/version?timeout=15m0s": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:47 [INFO] Running in single server mode, will not peer connections
2024/09/22 12:12:47 [INFO] Applying CRD features.management.cattle.io
2024/09/22 12:12:47 [INFO] Waiting for CRD features.management.cattle.io to become available
2024/09/22 12:12:48 [INFO] Done waiting for CRD features.management.cattle.io to become available
2024/09/22 12:12:49 [ERROR] unable to create feature rke1-ui in initialize features: Post "https://127.0.0.1:6444/apis/management.cattle.io/v3/features": unexpected EOF
2024/09/22 12:12:49 [ERROR] unable to retrieve feature embedded-cluster-api in initialize features: Get "https://127.0.0.1:6444/apis/management.cattle.io/v3/features/embedded-cluster-api": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:49 [ERROR] unable to retrieve feature rke2 in initialize features: Get "https://127.0.0.1:6444/apis/management.cattle.io/v3/features/rke2": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:49 [ERROR] unable to create feature rke2 in initialize features: Post "https://127.0.0.1:6444/apis/management.cattle.io/v3/features": dial tcp 127.0.0.1:6444: connect: connection refused
2024/09/22 12:12:49 [FATAL] k3s exited with: exit status 1
INFO: Running k3s server --cluster-init --cluster-reset
ERROR:
```

Chạy 3 lệnh sau để fix
```
sudo modprobe iptable_filter
sudo modprobe iptable_nat
sudo modprobe iptable_mangle
```
**NOTE:** Sau khi reboot thì 3 modules này sẽ bị mất, để 3 modules này loaded on boot thì ta cần config (tham khảo thêm tại [đây](https://unix.stackexchange.com/questions/71064/systemd-automate-modprobe-command-at-boot-time))
```
cat << EOF > /etc/modules-load.d/iptable.conf
iptable_filter
iptable_nat
iptable_mangle
EOF
```

Sau đó xoá và chạy lại container
* Truy cập Rancher tại:
```
https://192.168.10.19:6868
```
![image](https://github.com/user-attachments/assets/9b7add7f-1f70-4ca5-992d-c6273c6c9f99)

* Thêm K8S cluster vào Rancher
```
Add cluster -> Generic -> nhập Cluster Name -> Create
```

* Sau khi create, sẽ mất vài phút để add K8S Cluster vào Rancher, Rancher sẽ hiện ra cluster registration command, ta sẽ chạy command đó trên 1 node Master bất kỳ
```
curl --insecure -sfL https://192.168.10.19:6868/v3/import/lsp8hpdkcdjb4ws592vr4vd9m8hgdjjg2gqhxswvznwhzc5qm6r8mr_c-m-9wr7j9ks.yaml | kubectl apply -f -
```
![image](https://github.com/user-attachments/assets/9e3c076a-9eb5-422d-9c7e-5b5cf63f3daf)

* Cluster Dashboard

![image](https://github.com/user-attachments/assets/af679057-6fe9-4794-9794-c42458d5cb4e)
