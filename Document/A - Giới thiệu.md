# Các thành phần chính K8s

`Control Plane`: tập hợp các Master Node

`Master Node`: dùng để quản lý, lên lịch, theo dõi các Worker Node
- `ETCD`: là database dạng key-value dùng để lưu dữ liệu của K8s Cluster như là: Node, Pod, Service, Deployment, Configmap, ...
- `Kube-Scheduler`: điều phối các pod mới lên các Woker Node phù hợp dựa trên một số tiêu chí nhất định như: resource, node affinity, ...
- `Kube-Controller-Manager`: quản lý K8s cluster với nhiều Controller khác nhau
  - `Node-Controller`: theo dõi và phát hiện Worker Node nào bị down hoặc bị destroy
  - `Replication-Controller`: đảm bảo duy trì số lượng POD đang chạy trong 1 Replication
- `Kube-Apiserver`: là bộ não của K8s Cluster, có trách nhiệm làm việc với các thành phần trong K8s Cluster


`Worker Node`: các POD sẽ được điều phối lên Worker Node để host application
- `Kubelet`: là 1 agent chạy trên từng Woker Node trong K8s Cluster (Kubelet có thể chạy trên cả Master Node), lắng nghe các chỉ thị từ Kube-Apiserver để deploy/destroy POD trên Worker Node, theo dõi và báo cáo status của các Worker Node và POD cho Kube-Apiserver một cách định kỳ
- `Kube-proxy`: đảm bảo các POD trong K8s Cluster có thể giao tiếp được với nhau

![image](https://github.com/nguyenanhdongvn/Document/assets/90097692/fdfcc364-2c38-4dbb-9f30-b8d2accd8939)

# Servers Specification
| Server  | IP          | RAM | CPU   | DISK                                   | Service                                           |
|---------|-------------|-----|-------|----------------------------------------|---------------------------------------------------|
| master1 |192.168.10.11| 3GB | 2vCPU | - sda 50GB<br>- sdb 25GB               |- K8S master<br> - Keepalived<br> - HAproxy        |
| master2 |192.168.10.12| 3GB | 2vCPU | - sda 50GB<br>- sdb 25GB               |- K8S master<br> - Keepalived<br> - HAproxy        |
| master3 |192.168.10.13| 3GB | 2vCPU | - sda 50GB<br>- sdb 25GB               |- K8S master<br> - Keepalived<br> - HAproxy        |
| worker1 |192.168.10.14| 4GB | 2vCPU | - sda 50GB<br>- sdb 25GB               |- K8S worker                                       |
| worker2 |192.168.10.15| 4GB | 2vCPU | - sda 50GB<br>- sdb 25GB               |- K8S worker                                       |
| worker3 |192.168.10.16| 4GB | 2vCPU | - sda 50GB<br>- sdb 25GB               |- K8S worker                                       |
| gitlab  |192.168.10.18| 2GB | 2vCPU | - sda 50GB<br>- sdb 25GB               |- Gitlab                                           |
| rancher |192.168.10.19| 2GB | 2vCPU | - sda 50GB<br>- sdb 25GB<br>- sbc 45GB |- Rancher<br>- Harbor-Registry<br>- NFS-Server     |
| cicd    |192.168.10.20| 2GB | 2vCPU | - sda 50GB<br>- sdb 25GB               |- Jenkins<br>- kubespray<br>- Helm<br>- kubectl    |

# Chuẩn bị cài sẵn các gói và cấu hình cần thiết cho tất cả node

* thêm `sysadmin` user vào sudoer file
```
sudo su -
vi +108 /etc/sudoers
sysadmin    ALL=(ALL) NOPASSWD: ALL
```

* Cấu hình NetworkManager không khởi tạo lại file /etc/resolv.conf khi server bị reboot
```
sed -i 's/#plugins=keyfile,ifcfg-rh/dns=none/g' /etc/NetworkManager/NetworkManager.conf
systemctl restart NetworkManager
```

* Cấu hình DNS server trên tất cả node
```
cat <<EOF > /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
EOF
```

* Cấu hình host file cho tất cả node
```
cat <<EOF >> /etc/hosts
192.168.10.11   master1
192.168.10.12   master2
192.168.10.13   master3
192.168.10.14   worker1
192.168.10.15   worker2
192.168.10.16   worker3
192.168.10.18   gitlab
192.168.10.19   rancher
192.168.10.20   cicd
EOF
```

* Patching và cài đặt các package tool cần thiết cho tất cả node
```
sudo yum update -y
sudo yum install vim git telnet net-tools bind-utils bash-completion -y
source /etc/profile.d/bash_completion.sh
```

* Cài Docker lên tất cả node
```
curl -fsSL https://get.docker.com/ | sh
sudo usermod -aG docker sysadmin
sudo newgrp docker
sudo systemctl start docker
sudo systemctl enable docker
```

* Tắt SELinux và firewalld
```
sudo setenforce 0
sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

* Câu hình ip_forward
```
sudo sysctl -w net.ipv4.ip_forward=1
```

* Cấu hình Alias cho tất cả node
```
cat <<EOF >> $HOME/.bashrc

# Shell Prompt Color
export PS1="\[\033[01;33m\]\u\[\033[01;30m\]@\[\033[01;32m\]\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ "

# Custom Alias by Dong
alias cicd='ssh sysadmin@192.168.10.20'
alias rancher='ssh sysadmin@192.168.10.19'
alias gitlab='ssh sysadmin@192.168.10.18'
alias master1='ssh sysadmin@192.168.10.11'
alias master2='ssh sysadmin@192.168.10.12'
alias master3='ssh sysadmin@192.168.10.13'
alias worker1='ssh sysadmin@192.168.10.14'
alias worker2='ssh sysadmin@192.168.10.15'
alias worker3='ssh sysadmin@192.168.10.16'
alias k='kubectl'
EOF

source $HOME/.bashrc
```

* Tạo welcome message cho tất cả server
```
cat <<EOF > /etc/profile.d/motd.sh
#!/bin/bash
#
echo -e "
###############################################

             Welcome to \"`hostname`\"

       OS:     `cat /etc/redhat-release`
       Kernel: `uname -r`

       You are logged in as \"`whoami`\"

###############################################
"
EOF
```

* Tạo partition cho /data (longhorn) và /data2 (nfs)
```
sudo su -
# Tạo partition cho ổ mới
fdisk /dev/nvme0n2

# Tạo file system cho ổ mới
mkfs.xfs -f /dev/nvme0n2
mkdir /data
mount /dev/nvme0n2 /data 
mount | grep /nvme0n2

# Add fstab config để auto mount cho partition mới
cat <<EOF >> /etc/fstab
/dev/nvme0n2 /data                              xfs     defaults        0 0
EOF

mount -a && systemctl daemon-reload
```
