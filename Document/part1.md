| Server  | IP          | RAM | CPU   | DISK                                   | Service                                        |
|---------|-------------|-----|-------|----------------------------------------|------------------------------------------------|
| master1 |192.168.10.11| 3GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S master - Longhorn                          |
| master2 |192.168.10.12| 3GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S master                                     |
| master3 |192.168.10.13| 3GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S master                                     |
| worker1 |192.168.10.14| 4GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S worker                                     |
| worker2 |192.168.10.15| 4GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S worker                                     |
| worker3 |192.168.10.16| 4GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | K8S worker                                     |
| gitlab  |192.168.10.18| 2GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | Gitlab                                         |
| rancher |192.168.10.19| 2GB | 2vCPU | - sda 50GB<br>- sdb 20GB<br>- sbc 40GB | - Rancher<br>- Harbor-Registry<br>- NFS-Server |
| cicd    |192.168.10.20| 2GB | 2vCPU | - sda 50GB<br>- sdb 20GB               | - Jenkins<br>- kubespray                       |
