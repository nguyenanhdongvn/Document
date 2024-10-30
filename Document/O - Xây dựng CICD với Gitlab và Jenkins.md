# Cài đặt và cấu hình Gitlab trên `gitlab` server
## Cài đặt Gitlab
- Trên `gitlab` server, đổi port ssh của server thành port 23 vì gitlab chạy trên port 22
```
sudo sed -i 's/#Port 22/Port 23/g' /etc/ssh/sshd_config 
sudo systemctl restart sshd
```

- Tạo folder chứa log, data, config cho gitlab
```
export GITLAB_HOME=/srv/gitlab
sudo mkdir -p /srv/gitlab
sudo mkdir -p /srv/gitlab/logs
sudo mkdir -p /srv/gitlab/data
sudo mkdir -p /srv/gitlab/config
```

- Enable HTTPS cho gitlab
    - Từ `local machine`, tạo folder `/home/sysadmin/ssl` và copy cert lên `gitlab` server
    ```
    ssh -p 23 sysadmin@gitlab "mkdir -p /home/sysadmin/ssl"
    scp -P 23 /home/dong/ssl/dongna_app.crt /home/sysadmin/ssl/server.crt
    scp -P 23 /home/dong/ssl/dongna_app.key /home/sysadmin/ssl/server.key
    ```

    - Trên `gitlab` server, tạo đường dẫn chứa ssl certificate trong gitlab config folder, copy file cert+key vào
      ```
      sudo mkdir -p $GITLAB_HOME/config/ssl/gitlab.dongna.com/
      mv /home/sysadmin/ssl/server.* $GITLAB_HOME/config/ssl/gitlab.dongna.com/
      ```

    - Cấu hình enable https cho gitlab trong file `$GITLAB_HOME/config/gitlab.rb`    
    ```
    ## Gitlab URL
    external_url 'https://gitlab.dongna.com:443'
    ...
    ################################################################################
    ## GitLab NGINX
    ##! Docs: https://docs.gitlab.com/omnibus/settings/nginx.html
    ################################################################################
    nginx['enable'] = true
    nginx['client_max_body_size'] = '250m'
    nginx['redirect_http_to_https'] = true
    nginx['redirect_http_to_https_port'] = 80
    ...
    nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.dongna.com/server.crt"           # /etc/gitlab/ssl  is the path in docker container
    nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.dongna.com/server.key"
    ...
    nginx['listen_addresses'] = ['*', '[::]']
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


- Từ `local machine`, copy nội dung file public key để nhập vào trong Gitlab UI
```
cat ~/.ssh/id_rsa.pub

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC+Af66TOFI4gRFYmzB6ZWtCdSVw76MpBPXbnzvtmD7cvuB0gHl6dCyw+vThpsY4JqFflWKOB0vrbbWHHf1nSve0NEF+YakDj5BTSNVQztb8tYSZq50/GM0FP6/fSw9XXufjY1mkQg08jwnkiXqnm80Qf6wJm6k+b5ibNr/qX9UdgxFJSkdv18Eo2pXzD/aGW/WTXbYGN0WG5oXUt/lbcIjEg/fm4069iqIrqKGyOrxS7l/qRabo2CgC0S1/1C9fI12NEEisX/hOTn8UHwnqjVEYnYZQE6j2FuL0Dn4AqJfFh/Gc7PoaC6gVhElDJIk6IKn4dI8jjZ/t6MD6h+WYxYoxD97M5HrinTVDrJh1s4kdnFhgzS1yvVCsqNCPsUbi5dkS3Wiy1BFaHosCgOM3TZYogmcEuJc6SfOoYgbPXU2I1zeUzXFrksVTAyWQhwDFhYQc7JpdPnKoefUFF6Y6t2/EYB/uDP3NU9ySZ3RuLOtJuLdK67QXrw3mzXHOVTQYeM= dong@DONG-PC
```

- Nhập nội dung public key vào Gitlab UI
![image](https://github.com/user-attachments/assets/f59be020-2cd6-4b5d-92fc-6d785bc85a17)

- Test login Gitlab từ `local machine`
```
ssh -T git@gitlab.dongna.com
The authenticity of host 'gitlab.dongna.com (192.168.10.18)' can't be established.
ED25519 key fingerprint is SHA256:eaYNCrGx1BpkxW0eT3bfqRACfW8+C9sgzQEG5C0ThhU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'gitlab.dongna.com' (ED25519) to the list of known hosts.
Welcome to GitLab, @dongna!
```

## Cấu hình Gitlab
- Từ `local machine`, khởi tạo local repo và push local repo lên gitlab repo
    - `nodejs-demo` repository
    ```
    cd $HOME/argocd-demo/nodejs-demo/
    git init
    git remote add origin git@gitlab.dongna.com:dongna/nodejs-demo.git
    git add .
    git commit -m "Initinal commit"
    git push -u origin master
    ```
    
    - `helmchart-demo` repository
    ```
    cd $HOME/argocd-demo/helmchart-demo/
    git init
    git remote add origin git@gitlab.dongna.com:dongna/helmchart-demo.git
    git add .
    git commit -m "Initinal commit"
    git push -u origin master
    ```
- Trên Gitlab UI, tạo user `jenkins_gitlab`, gán permission `developer` cho user `jenkins_gitlab` trong project (repo) `nodejs-demo` và `helmchart-demo` để Jenkins có thể móc vào Gitlab và pull source code
![image](https://github.com/user-attachments/assets/b8a31351-6ca6-419c-b4a5-d676572534af)

# Cài đặt và cấu hình Jenkins trên `cicd` server
## Cài đặt Jenkins
- Trên `cicd` server, cài đặt Jenkins như sau
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

## Cấu hình Jenkins
- Cài đặt plugin: git, Docker pipline
    - Dashboard > Manage > Plugins > Available Plugin
    ![image](https://github.com/user-attachments/assets/afd60d2d-bbc8-46d6-857a-f7d2d5f16777)

- Tạo Credential `jenkins_gitlab` và `jenkins_harbor` để Jenkins connect vào Gitlab Repository và Harbor Registry (nhập ID trùng với Username): Dashboard > Manage Jenkins > Credentials > Domains (global) > Add Credentials
    - `jenkins_gitlab`<br>
    ![image](https://github.com/user-attachments/assets/251e5638-dcae-4e5b-9474-f768e20f1a26)
    - `jenkins_harbor` <br>
    ![image](https://github.com/user-attachments/assets/053d9adb-9699-4523-bea7-4550cc35930b)

- Trên `cicd` server, cấu hình cho `jenkins` user có quyền chạy docker without sudo
```
sudo usermod -aG docker jenkins
sudo service jenkins restart
```

- Trên `cicd` server, cấu hình cho Jenkins có thể connect vào Harbor Registry bằng URL (add file hosts và cấu hình certificate cho docker connect đến Harbor Registry)
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
cp /home/sysadmin/ssl/rootCA.pem /etc/docker/certs.d/harbor.dongna.com/rootCA.pem

sudo systemctl restart docker
```

- Test thử push image lên Harbor bằng user `jenkins_harbor`
```
docker login harbor.dongna.com -u jenkins_harbor
docker push harbor.dongna.com/demo/hello-world:v1
```

- Cấu hình cho user `jenkins` có thể kết nối tới K8s, để `jenkins` có thể deploy/upgrade application bằng lệnh kubectl/helm trên K8S
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
      
    - Sau khi cài Jenkins, mặc định user `jenkins` cũng được tạo ra nhưng không login bằng user này được. Ta phải cấu hình cho user `jenkins` có thể login vào được trên `cicd` server
      ```
      sudo vim /etc/passwd
      ```
      
      ```
      from   "jenkins:x:992:988:Jenkins Automation Server:/var/lib/jenkins:/bin/false"
      to     "jenkins:x:992:988:Jenkins Automation Server:/var/lib/jenkins:/bin/bash"
      ```
      
    - Login vào user `jenkins` từ user `root` và tạo file kube config ở đường dẫn `/home/jenkins/.kube/config` tương tự với lúc cấu hình kubectl trên Master
      ```
      sudo su -
      su - jenkins
      mkdir -p $HOME/.kube/
      scp sysadmin@master1:~/.kube/config  $HOME/.kube/
      sed -i 's/127.0.0.1/192.168.10.11/g' $HOME/.kube/config
      ```
    - Từ user `jenkins` thử chạy command `kubectl` và `helm`
      ```
      kubectl get node
      helm list
      ```

- **Như vậy, ta đã cấu hình cho Jenkins có thể chạy các command `docker/kubectl/helm` để build/deploy và để có thể kết nối vào Gitlab Repo, Harbor Registry, K8S.**


# Cài đặt ArgoCD
- Tạo folder chứa ArgoCD Helm Chart
```
mkdir $HOME/k8s/argocd && cd $HOME/k8s/argocd
```

- Tải ArgoCD Helm Chart
```
helm repo add argo https://argoproj.github.io/argo-helm
helm search repo argo-cd
helm pull argo/argo-cd --version 7.6.12
tar -xzf argo-cd-7.6.12.tgz
cp argo-cd/values.yaml values-argocd.yaml
```

- Cấu hình ArgoCD Helm Chart bằng cách sửa file values-argocd.yaml
```
global:
  domain: argo.dongna.com
...
configs:
  params:
    server.insecure: true
...
server:
  ingress:
    enabled: true
    ingressClassName: "nginx"
    hostname: argocd.dongna.com
```

- Tạo namespace `argocd` và cài đặt ArgoCD Helm Chart
```
kubectl create ns argocd
helm -n argocd install argocd -f values-argocd.yaml argo/argo-cd 
```

**NOTE: Nếu gặp phải error với CRDs, sửa `crds.install` value trong file values.yaml thành `false`**
    - Error message
    ```
    Error: INSTALLATION FAILED: Unable to continue with install: CustomResourceDefinition "applications.argoproj.io" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-namespace" must equal "argocd": current value is "default"
    ```
    - sửa file values-argocd.yaml
    ```
    crds:
      # -- Install and upgrade CRDs
      install: false
    ```

- Cấu hình service thành NodePort:
```
kubectl patch svc -n argocd argocd-server --patch '{"spec": {"type": "NodePort"}}'
```

- Kết quả
```
k -n argocd get all

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                    1/1     Running   0          2m5s
pod/argocd-applicationset-controller-75d8c9495-bj2hp   1/1     Running   0          2m31s
pod/argocd-dex-server-7c9b44b9f9-fnn6d                 1/1     Running   0          2m31s
pod/argocd-notifications-controller-77f49c7745-xkv2c   1/1     Running   0          2m27s
pod/argocd-redis-575c96bc4f-n7js4                      1/1     Running   0          2m20s
pod/argocd-repo-server-7f44b474d7-xgbcf                1/1     Running   0          2m17s
pod/argocd-server-5f4dd5d648-2psxc                     1/1     Running   0          2m8s

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.233.24.253   <none>        7000/TCP,8080/TCP            3m9s
service/argocd-dex-server                         ClusterIP   10.233.62.41    <none>        5556/TCP,5557/TCP,5558/TCP   3m9s
service/argocd-metrics                            ClusterIP   10.233.52.42    <none>        8082/TCP                     3m8s
service/argocd-notifications-controller-metrics   ClusterIP   10.233.34.179   <none>        9001/TCP                     3m6s
service/argocd-redis                              ClusterIP   10.233.40.10    <none>        6379/TCP                     3m5s
service/argocd-repo-server                        ClusterIP   10.233.41.36    <none>        8081/TCP,8084/TCP            3m5s
service/argocd-server                             ClusterIP   10.233.34.204   <none>        80/TCP,443/TCP               3m4s
service/argocd-server-metrics                     ClusterIP   10.233.5.81     <none>        8083/TCP                     2m57s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           2m44s
deployment.apps/argocd-dex-server                  1/1     1            1           2m39s
deployment.apps/argocd-notifications-controller    1/1     1            1           2m30s
deployment.apps/argocd-redis                       1/1     1            1           2m22s
deployment.apps/argocd-repo-server                 1/1     1            1           2m19s
deployment.apps/argocd-server                      1/1     1            1           2m11s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-75d8c9495   1         1         1       2m33s
replicaset.apps/argocd-dex-server-7c9b44b9f9                 1         1         1       2m33s
replicaset.apps/argocd-notifications-controller-77f49c7745   1         1         1       2m28s
replicaset.apps/argocd-redis-575c96bc4f                      1         1         1       2m21s
replicaset.apps/argocd-repo-server-7f44b474d7                1         1         1       2m18s
replicaset.apps/argocd-server-5f4dd5d648                     1         1         1       2m9s

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     2m6s
```

- Để truy cập ArgoCD ta cần
    - Thông tin NodePort được config trên service `argocd-server`
      ```
      kubectl -n argocd get svc argocd-server
      ```
    - Thông tin default password để login ArgoCD lần đầu
      ```
      kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
      ```
- Truy cập vào IP bất kỳ của K8s Cluster (worker/master nodes) VD: http://[master/worker_IP]:[NodePort_port]
![image](https://github.com/user-attachments/assets/e7780a0a-f19b-43d3-8367-a1f32019d23e)

# Tạo pipeline Job trên Jenkins
- Ta cần define Job cho jenkins thực hiện các task như sau:
    - Pull source code từ Gitlab Repo (`nodejs-demo` repo)
    - Build source code
    - Build Docker image
    - Push Docker image lên Registry (Harbor)
    - Update application version (image.tag) vào file Helm Chart values.yaml lưu trong Gitlab Repo (`helmchart-demo` repo)
- Ý tưởng là mỗi lần build source code thành docker image, ta sẽ sử dụng variable `BUILD_NUMBER` của Jenkins (tăng lần sau mỗi lần build) gán tag cho Image. Do đó ta cũng cần update variable này cho file Helm Chart values.yaml (nằm trong `helmchart-demo` repo) để deploy lên K8s. Để thực hiện được ý tưởng này, ta có nhiều solution như sau:
    - Pull Helm Chart từ Gitlab Repo (`helmchart-demo` repo)
    - Update values image.tag theo đúng tag mới build
    - Commit changes lên Gitlab Repo (`helmchart-demo` repo)

```
def appSourceRepo = 'http://gitlab.dongna.com/dongna/nodejs-demo.git'
def appSourceBranch = 'master'

def appConfigRepo = 'http://gitlab.dongna.com/dongna/helmchart-demo.git'
def appConfigBranch = 'master'
def helmRepo = "helmchart-demo"
def helmChart = "helmchart-demo"
def helmValueFile = "app-demo-value.yaml"

def dockerhubAccount = 'jenkins_harbor'
def gitlabAccount = 'jenkins_gitlab'

dockerBuildCommand = './'
def version = "v1.${BUILD_NUMBER}"

pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'https://harbor.dongna.com'
        DOCKER_IMAGE_NAME = "nodejs-demo/nodejs-demo"
        DOCKER_IMAGE = "harbor.dongna.com/${DOCKER_IMAGE_NAME}"
    }

    stages {
        stage('Checkout project') {
          steps {
            git branch: appSourceBranch,
                credentialsId: gitlabAccount,
                url: appSourceRepo
          }
        }
        stage('Build And Push Docker Image') {
            steps {
                script {
                    sh "git reset --hard"
                    sh "git clean -f"
                    app = docker.build(DOCKER_IMAGE_NAME, dockerBuildCommand)
                    docker.withRegistry( DOCKER_REGISTRY, dockerhubAccount ) {
                       app.push(version)
                    }

                    sh "docker rmi ${DOCKER_IMAGE_NAME} -f"
                    sh "docker rmi ${DOCKER_IMAGE}:${version} -f"
                }
            }
        }

        stage('Update value in helm-chart') {
            steps {
                                withCredentials([usernamePassword(credentialsId: 'jenkins_gitlab', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                                sh """#!/bin/bash
                                           [[ -d ${helmRepo} ]] && rm -r ${helmRepo}
                                           git clone http://${GIT_USERNAME}:${GIT_PASSWORD}@gitlab.dongna.com/dongna/helmchart-demo.git --branch ${appConfigBranch}
                                           cd ${helmRepo}
                                           sed -i 's|  tag: .*|  tag: "${version}"|' ${helmValueFile}
                                           git add . ; git commit -m "Update to version ${version}";git push http://${GIT_USERNAME}:${GIT_PASSWORD}@gitlab.dongna.com/dongna/helmchart-demo.git
                                           cd ..
                                           [[ -d ${helmRepo} ]] && rm -r ${helmRepo}
                                           """
                                }
            }
        }
    }
}

```
- 
