```
 mkdir kubernetes_installation/ && cd kubernetes_installation/
```

git clone https://github.com/kubernetes-sigs/kubespray.git --branch release-2.16

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
