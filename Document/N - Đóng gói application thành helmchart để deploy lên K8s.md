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

replicaCount: 2 # <=> số lượng replicas trong file deployment.yaml

image:
  repository: harbor.dongna.com/demo/my-app # <=> image trong file deployment.yaml
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
  type: ClusterIP # <=> service/type trong file service.yaml
  port: 80 # <=> service/port trong file service.yaml

ingress:
  enabled: true
  className: "local" # <=> ingressClass trong file ingress.yaml
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: helm-demo.dong.com # <=> ingress/host trong file ingress.yaml
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
```

- Sau khi cập nhật các tham số thì ta sẽ tiến hành cài đặt helmchart này lên k8s:
```
kubectl create ns helm-demo
helm -n helm-demo install my-app -f app-demo-value.yaml app-demo
```

- Kiểm tra
```
kubectl -n helm-demo get all
```

- Output
```

```

Okela, vậy là mình đã hoàn thành đóng gói ứng dụng này bằng helm và ready cho việc mang đi triển khai trên các hệ thống K8S rồi. Việc tạo helmchart cho ứng dụng này hoàn toàn có thể tái sử dụng. Nghĩa là bạn có một nhóm ứng dụng tương đồng nhau về cách triển khai thì có thể dùng chung chart, chỉ khác nhau ở các customized values sẽ được set riêng cho từng ứng dụng khi triển khai.

Mở rộng ra thêm bạn có thể tạo ra các template cho statefulset, daemonset để bổ sung vào helmchart của bạn. Bạn cũng có thể bổ sung thêm cấu hình về persistent, configmap hay secrete.. trong template của deployment. Một kinh nghiệm của mình là lấy các template trong các chart của các opensource phổ biến, hầu hết config ở đó đều rất đầy đủ, cần phần nào thì copy về tùy biến và dùng thôi.
