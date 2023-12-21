Tạo folder chứa kubespray
```
 mkdir /home/sysadmin/kubernetes_installation/ && cd /home/sysadmin/kubernetes_installation/
```

Clone kubesrpay
```
git clone https://github.com/kubernetes-sigs/kubespray.git --branch release-2.16
```

Copy folder `sample` ra folder `dongna-cluster`
```
cp -rf /home/sysadmin/kubernetes_installation/kubespray/inventory/sample /home/sysadmin/kubernetes_installation/kubespray/inventory/dongna-cluster
```

Cấu hình file inventory `host.yaml` cho Ansible
```
vim /home/sysadmin/kubernetes_installation/kubespray/inventory/dongna-cluster/host.yaml
```

```
[all]
master1  ansible_host=192.168.10.11      ip=192.168.10.11
master2  ansible_host=192.168.10.12      ip=192.168.10.12
master3  ansible_host=192.168.10.13      ip=192.168.10.13
worker1  ansible_host=192.168.10.14      ip=192.168.10.14
worker2  ansible_host=192.168.10.15      ip=192.168.10.15
worker3  ansible_host=192.168.10.16      ip=192.168.10.16

[kube-master]
master1
master2
master3

[kube-node]
worker1
worker2
worker3

[etcd]
master1
master2
master3

[k8s-cluster:children]
kube-node
kube-master

[calico-rr]

[vault]
master1
master2
master3
worker1
worker2
worker3
```

Đổi CNI (Container Network Interface) 
```
vim /home/sysadmin/kubernetes_installation/kubespray/inventory/dongna-cluster/group_vars/k8s_cluster/k8s-cluster.yml
```

```
Từ
kube_network_plugin: calico
Thành
kube_network_plugin: flannel
```

Cài Docker lên `cicd` server để chạy Ansible bằng container
```
curl -fsSL https://get.docker.com/ | sh
sudo usermod -aG docker sysadmin
sudo newgrp docker
sudo systemctl start docker
sudo systemctl enable docker
```

Chạy container đã cài Ansible
```
docker run --rm -it --mount type=bind,source=/home/sysadmin/kubernetes_installation/kubespray/inventory/dongna-cluster,dst=/inventory \
  --mount type=bind,source=/home/sysadmin/.ssh/id_rsa,dst=/root/.ssh/id_rsa \
  --mount type=bind,source=/home/sysadmin/.ssh/id_rsa,dst=/home/sysadmin/.ssh/id_rsa \
  quay.io/kubespray/kubespray:v2.16.0 bash
```

Trong container chạy ansible command để cài đặt K8S cluster
```
ansible-playbook -i /inventory/host.yaml cluster.yml --user=sysadmin --ask-pass --become --ask-become-pass
```

Cấu hình kube config trên 3 Master Node để connect được đến K8S cluster
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kiểm tra kết quả sau khi cấu hình kube config
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
- Trên `rancher` server, cài đặt Rancher bằng cách chạy Rancher container:
```
docker run --name rancher-server -d --restart=unless-stopped -p 6860:80 -p 6868:443 --privileged rancher/rancher:v2.5.7 
```
_**Note:**_
Giữa Rancher và Kubernetes có bảng tương thích, ta có thể check trên trang chủ của Rancher. Ví dụ: Kubernetes v1.20.7 tương thích với Rancher v2.5.7.
Expose Rancher bằng port 6860 cho HTTP và 6868 cho HTTPS để sau này sẽ cho các connection này đi qua HAproxy


- Truy cập Rancher tại:
```
https://192.168.10.19:6868
```
![image](https://github.com/nguyenanhdongvn/Document/assets/90097692/3a704e7a-602e-4a18-86e1-a2aea4cb9ce8)

- Thêm K8S cluster vào Rancher
```
Add cluster -> Other Cluster -> nhập Cluster Name -> Create
```

- Sau khi create, sẽ mất vài phút để add K8S Cluster vào Rancher, Rancher sẽ hiện ra cluster registration command, ta sẽ chạy command đó trên 1 node Master bất kỳ
```
curl --insecure -sfL https://192.168.10.19:6868/v3/import/fqskn9m6v9s5jgfvhl79qf7hcwdbd7khrdcpxqqmwz6pfpkrmwzbm7_c-tw6bv.yaml | kubectl apply -f -
```
![image](https://github.com/nguyenanhdongvn/Document/assets/90097692/4cf1b815-f479-4b57-a180-b0dc6633cb31)

_**Note:**_
Rancher UI báo Controller Manager unhealthy ta cần thêm các bước sau:
```
sudo sed -i 's|- --port=0|#- --port=0|' /etc/kubernetes/manifests/kube-scheduler.yaml
sudo sed -i 's|- --port=0|#- --port=0|' /etc/kubernetes/manifests/kube-controller-manager.yaml
sudo systemctl restart kubelet
```
