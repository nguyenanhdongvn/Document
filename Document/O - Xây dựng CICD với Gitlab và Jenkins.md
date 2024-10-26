Đổi port ssh của server vì gitlab chạy trên port 22
```
sudo sed -i 's/#Port 22/Port 23/g' /etc/ssh/sshd_config 
sudo systemctl restart sshd
```

Tạo folder chứa log, data, config cho gitlab
```
export GITLAB_HOME=/srv/gitlab
sudo mkdir -p /srv/gitlab
sudo mkdir -p /srv/gitlab/logs
sudo mkdir -p /srv/gitlab/data
sudo mkdir -p /srv/gitlab/config
```

- Cài đặt gitlab
```
sudo docker run --detach --hostname gitlab.dongna.com --publish 443:443 --publish 80:80 --publish 22:22 --name gitlab --restart always --volume $GITLAB_HOME/config:/etc/gitlab --volume $GITLAB_HOME/logs:/var/log/gitlab --volume $GITLAB_HOME/data:/var/opt/gitlab --shm-size 256m gitlab/gitlab-ce:latest
```

- Lấy root password để login gitlab UI
```
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

- Truy cập http://gitlab.dongna.com/ và đăng nhập bằng user `root`
![image](https://github.com/user-attachments/assets/9919a924-dc05-47e6-aa78-5eab55daa9c1)


Từ local host, copy nội dung file public key để nhập vào trong Gitlab UI
```
cat ~/.ssh/id_rsa.pub

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC+Af66TOFI4gRFYmzB6ZWtCdSVw76MpBPXbnzvtmD7cvuB0gHl6dCyw+vThpsY4JqFflWKOB0vrbbWHHf1nSve0NEF+YakDj5BTSNVQztb8tYSZq50/GM0FP6/fSw9XXufjY1mkQg08jwnkiXqnm80Qf6wJm6k+b5ibNr/qX9UdgxFJSkdv18Eo2pXzD/aGW/WTXbYGN0WG5oXUt/lbcIjEg/fm4069iqIrqKGyOrxS7l/qRabo2CgC0S1/1C9fI12NEEisX/hOTn8UHwnqjVEYnYZQE6j2FuL0Dn4AqJfFh/Gc7PoaC6gVhElDJIk6IKn4dI8jjZ/t6MD6h+WYxYoxD97M5HrinTVDrJh1s4kdnFhgzS1yvVCsqNCPsUbi5dkS3Wiy1BFaHosCgOM3TZYogmcEuJc6SfOoYgbPXU2I1zeUzXFrksVTAyWQhwDFhYQc7JpdPnKoefUFF6Y6t2/EYB/uDP3NU9ySZ3RuLOtJuLdK67QXrw3mzXHOVTQYeM= dong@DONG-PC
```

Nhập nội dung public key vào Gitlab UI
![image](https://github.com/user-attachments/assets/f59be020-2cd6-4b5d-92fc-6d785bc85a17)


Test login gitlab
```
ssh -T git@gitlab.dongna.com
The authenticity of host 'gitlab.dongna.com (192.168.10.18)' can't be established.
ED25519 key fingerprint is SHA256:eaYNCrGx1BpkxW0eT3bfqRACfW8+C9sgzQEG5C0ThhU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'gitlab.dongna.com' (ED25519) to the list of known hosts.
Welcome to GitLab, @dongna!
```

- Trên Gitlab UI, tạo user `jenkins_gitlab` để Jenkins có thể móc vào gitlab và pull source code, gán permission cho `jenkins` user có permission với project `demo`






# Install Jenkins
```
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade -y
# Add required dependencies for the jenkins package
sudo yum install -y fontconfig java-17-openjdk
sudo yum install -y jenkins
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

- Lấy password admin để login Jenkins UI
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

# Cài đặt cấu hình Jenkins
- Cài đặt plugin: git, Docker pipline
- Tạo Credential `jenkins_gitlab` để Jenkins connect vào Gitlab Repository (nhập ID trùng với Username)
- Tạo Credential `jenkins_harbor` để Jenkins connect vào Harbor Registry (nhập ID trùng với Username)
- Cấu hình cho `jenkins` user có quyền chạy docker without sudo
```
sudo usermod -aG docker jenkins
sudo service jenkins restart
```
- Cấu hình cho Jenkins `cicd` server connect tới Harbor Registry (add file hosts và cấu hình certificate cho docker connect đến Harbor Registry)
```
# Add file /etc/hosts
cat << EOF >> /etc/hosts

# Gitlab Repository
192.168.10.18   gitlab.dongna.com
# Harbor Registry
192.168.10.20   harbor.dongna.com
EOF

# Put CA cert into /etc/docker/certs.d/harbor.dongna.com/
mkdir -p /etc/docker/certs.d/harbor.dongna.com/
scp sysadmin@cicd:/home/sysadmin/ssl/rootCA.pem /etc/docker/certs.d/harbor.dongna.com/rootCA.pem
```

- Test thử push image lên Harbor bằng user `jenkins`
```
docker login harbor.dongna.com -u jenkins
docker push harbor.dongna.com/demo/hello-world:v1
```

- Cấu hình user `jenkins` kết nối tới K8s, để `jenkins` có thể deploy/upgrade application bằng lệnh kubectl/helm trên K8S
    - Cài đặt kubectl/helm
      ```
      # Install kubectl ver 1.30.4
      curl -LO https://dl.k8s.io/release/v1.30.4/bin/linux/amd64/kubectl
      sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
      kubectl version --client

      # Install helm
      curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
      sudo chmod 700 get_helm.sh
      ./get_helm.sh
      ```
    - Khi cài Jenkins thì user `jenkins` cũng được tạo ra nhưng không ssh bằng user này được. Ta phải cấu hình server `gitlab` có thể ssh vào bằng user `jenkins`
      ```
      vim /etc/passwd

      from   "jenkins:x:992:988:Jenkins Automation Server:/var/lib/jenkins:/bin/false"
      to     "jenkins:x:992:988:Jenkins Automation Server:/var/lib/jenkins:/bin/bash"
      ```
    - Login vào user `jenkins` và tạo file config ở đường dẫn /home/jenkins/.kube/config tương tự với lúc cấu hình kubectl trên Master vậy
      ```
      su - jenkins
      mkdir -p $HOME/.kube/
      scp sysadmin@master1:~/.kube/config  $HOME/.kube/
      sed -i 's/127.0.0.1/192.168.10.11/g' $HOME/.kube/config
      ```
    - Từ user `jenkins` thử chạy command kubectl và helm xem ok chưa
      ```
      kubectl get node
      helm list
      ```

- **Như vậy, ta đã cấu hình cho Jenkins có thể chạy các command `docker/kubectl/helm` để build/deploy và để có thể kết nối vào Gitlab Repo, Harbor Registry, K8S.**

## Tạo Jenkins Job
