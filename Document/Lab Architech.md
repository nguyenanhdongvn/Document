| Server  | IP          | RAM | CPU   | DISK                                   | Service                                           |
|---------|-------------|-----|-------|----------------------------------------|---------------------------------------------------|
| master1 |192.168.10.11| 3GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S master - Longhorn                             |
| master2 |192.168.10.12| 3GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S master                                        |
| master3 |192.168.10.13| 3GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S master                                        |
| worker1 |192.168.10.14| 4GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S worker                                        |
| worker2 |192.168.10.15| 4GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S worker                                        |
| worker3 |192.168.10.16| 4GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S worker                                        |
| gitlab  |192.168.10.18| 2GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | Gitlab                                            |
| rancher |192.168.10.19| 2GB | 2vCPU | - sda 50GB<br>- sdb 20GB<br>- sbc 40GB | - Rancher<br>- Harbor-Registry<br>- NFS-Server    |
| cicd    |192.168.10.20| 2GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | - Jenkins<br>- kubespray<br>- Helm<br>- kubectl   |

Configure DNS server on all nodes
```
cat <<EOF > /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
EOF
```

Configure hosts file on all nodes
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

Update and Install the needed tools for the servers
```
sudo yum update -y
sudo yum install vim git telnet net-tools bind-utils ntp bash-completion bash-completion-extras -y
source /etc/profile.d/bash_completion.sh
```

* Cài Docker lên các servers
```
curl -fsSL https://get.docker.com/ | sh
sudo usermod -aG docker sysadmin
sudo newgrp docker
sudo systemctl start docker
sudo systemctl enable docker
```

Disable SELinux
```
sudo setenforce 0
sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

Turn off firewalld
```
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

Configure ip_forward
```
sudo sysctl -w net.ipv4.ip_forward=1
```
