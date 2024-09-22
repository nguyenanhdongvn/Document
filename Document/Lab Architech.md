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
