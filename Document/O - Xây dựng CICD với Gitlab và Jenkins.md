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

Cài đặt gitlab
```
sudo docker run --detach --hostname gitlab.dongna.com --publish 443:443 --publish 80:80 --publish 22:22 --name gitlab --restart always --volume $GITLAB_HOME/config:/etc/gitlab --volume $GITLAB_HOME/logs:/var/log/gitlab --volume $GITLAB_HOME/data:/var/opt/gitlab --shm-size 256m gitlab/gitlab-ce:latest
```

Lấy root password để login gitlab UI
```
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```


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

Trên Gitlab UI, tạo user `jenkins` để Jenkins có thể móc vào gitlab và pull source code, gán permission cho `jenkins` user có permission với project `demo`






# Install Jenkins
```
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade
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
