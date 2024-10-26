# Giới thiệu
Helm là một package manager cho Kubernetes, nó đóng gói các Resource của Kubernetes vào một đơn vị triển khai logic duy nhất được gọi là Chart. Bên trong Chart sẽ có các Template, là template cho các Resource sẽ được tạo trên K8s.

Cụ thể, Để deploy application lên K8s, ta cần tạo 3 file yaml gồm deployment.yaml, service.yaml và ingress.yaml. Các file này định nghĩa các tham số cấu hình cho việc triển khai ứng dụng. Tuy nhiên khi cần thay đổi tham số thì việc sử dụng các file đó sẽ trở nên cồng kềnh và khó kiểm soát, không có quản lý version trên K8s.

Helm giải quyết được vấn đề trên, Helm Chart có các file resource nhưng ở dạng template. Template của các file deployment hay service cơ bản vẫn vậy, nhưng thay vì gán value kiểu hard code, thì ta có thể refer đến các value được khai báo ở trong file value.yaml.

# Cách tạo Helm Chart
- Trước khi tạo được helm ta cần phải cài helm và kubectl sau đó cấu hình kube config connect đến K8s cluster
```
mkdir -p $HOME/k8s/helm-chart
helm create demo-helmchart
```

- Output
```
ll $HOME/k8s/helm-chart/demo-helmchart/
total 12
drwxr-xr-x 2 sysadmin sysadmin    6 Oct 21 13:53 charts
-rw-r--r-- 1 sysadmin sysadmin 1144 Oct 21 13:53 Chart.yaml
drwxr-xr-x 3 sysadmin sysadmin  162 Oct 21 13:53 templates
-rw-r--r-- 1 sysadmin sysadmin 4294 Oct 21 13:53 values.yaml
```

- Mặc định, khi tạo một Helm Chart, ta sẽ có các template cho: `Deployment`, `Service`, `Service Account`, `Ingress`, `HPA` và các `tests`. So sánh với phần trước khi deploy application bằng các manifest file thì ta đã viết 3 file gồm: `deployment.yaml`, `service.yaml` và `ingress.yaml`

- Thay vì ta phải tự viết từng tham số trong file deployment.yaml, thì khi dùng Helm Chart ta sẽ có một file template gần như đầy đủ các thành phần của một Deployment, còn dùng những parameter tuỳ biến thì ta khai báo trong file values.yaml

**NOTE:**
- **Trong phần trước ta có gán thêm các biến môi trường vào Deployment, các biến môi trường này trong template Deployment không có => Ta phải customize template Deployment**
- **Trong template Deployment mặc định chỉ có 1 container và port mặc định là 80, port này sẽ dùng cho liveness probe và readiness probe của port. Application hiện tại đang listen port 8080 nên để không phải sửa code thì ta sẽ sửa value của Deployment thành 8080.**

**=> Để customize 1 application trước khi deploy lên K8s bằng Helm Chart thì ta không cần phải tạo ra từng file manifest cho từng resource, ta chỉ cần update 1 file values duy nhất.**

# Giải thích 1 vài value trong file value.yaml
```
# Default values for app-demo.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 3 # <=> số lượng replicas trong file deployment.yaml

image:
  repository: harbor.dongna.com/demo-nodejs/demo-nodejs # <=> image trong file deployment.yaml
  pullPolicy: Always # <=> imagePullPolicy trong file deployment.yaml
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}

service:
  type: LoadBalancer # <=> service/type trong file service.yaml
  port: 5000 # <=> service/port trong file service.yaml
  targetPort: 8080
  NodePort:30888

ingress:
  enabled: true
  className: "nginx" # <=> ingressClass trong file ingress.yaml
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: demo-helmchart.dongna.com # <=> ingress/host trong file ingress.yaml
      paths:
        - path: / #Tương ứng ingress/path trong file ingress.yaml
          pathType: ImplementationSpecific
  tls: []
resources: {}
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}
```

- Với template của deployment, cần bổ sung thêm phần cấu hình biến môi trường env:
```
      containers:
          ...
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: MY_POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: MY_POD_SERVICE_ACCOUNT
              valueFrom:
                fieldRef:
                  fieldPath: spec.serviceAccountName
        livenessProbe
        ...
```

- Sau khi cập nhật các tham số thì ta sẽ tiến hành cài đặt helmchart này lên k8s:
```
kubectl create ns demo-helmchart
cd $HOME/k8s/helm-chart/
helm -n demo-helmchart install demo-helmchart -f demo-helmchart/value-demo-helmchart.yaml demo-helmchart
```
**NOTE: để install được helm chart, ta phải đứng ở đường dẫn chứa folder helm chart đó**

- Truy cập thử https://demo-helmchart.dongna.com/ <br>
![image](https://github.com/user-attachments/assets/360c8584-a84c-4461-824b-100c4cdc5e48)


- Kiểm tra
```
kubectl -n demo-helmchart get all
```

- Output
```
NAME                                  READY   STATUS    RESTARTS   AGE
pod/demo-helmchart-8597d6d7dc-bzqb2   1/1     Running   0          3m3s
pod/demo-helmchart-8597d6d7dc-fwkn6   1/1     Running   0          3m3s
pod/demo-helmchart-8597d6d7dc-s4lkl   1/1     Running   0          3m3s

NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/demo-helmchart   LoadBalancer   10.233.17.81   <pending>     5000:31210/TCP   3m3s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demo-helmchart   3/3     3            3           3m3s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/demo-helmchart-8597d6d7dc   3         3         3       3m3s
```


- Như vậy, ta đã hoàn thành đóng gói application nodejs bằng Helm Chart và sẵn sàng để deploy trên K8S
- Helm Chart có thể tái sử dụng. Nếu có 1 application tương tự về các manifest file thì ta có thể tái sử dụng Chart cũ, chỉ cần customize file values.yaml để setup riêng paremeter cho từng application khác nhau khi deploy
- Ta có thể tạo ra các template cho Statefulset, Daemonset để thêm vào Helm Chart. Ta cũng có thể customize thêm các config của Persistent, Configmap, Secrete, ... trong template của Deployment.
- Một kinh nghiệm của mình là lấy các template trong các Helm Chart của các opensource phổ biến, hầu hết config ở đó đều rất đầy đủ, cần phần nào thì copy về customize và sử dụng
