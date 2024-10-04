# Giới thiệu
## Mục tiêu
- Lấy Metrics của application bằng cách cấu hình Job trong scrapeConfig
- Lấy Metrics của application bằng cách cấu hình Service Monitor
- Cài đặt một application hỗ trợ cả Metric và ServiceMonitor
- Troubleshoot các issue trong khi cấu hình

## Kiến trúc
![image](https://github.com/user-attachments/assets/072f413f-af2d-4397-b9d2-05b9cabd445a)

## Workflow của monitoring bằng Prometheus sẽ được mô tả như sau:
- Prometheus pull Metrics từ các target cần được monitor, các target này có thể chia thành 2 loại:
    - Target hỗ trợ Expose Metrics (nghĩa là các target cần được monitor đã có sẵn api để Prometheus có thể truy cập đến và pull Metrics)
    - Target không hỗ trợ Expose Metrics (phải cài thêm exporter cho các target để exporter gửi Metrics về Prometheus)
**Note:** Quá trình Prometheus pull Metric từ target được gọi là "Job". Các "Job" này chứa thông tin Target nó cần pull Metrics cũng như cách lấy Metric (Interval, Retention...). Và để tạo các "Job" này có 2 cách:<br>
        - Cấu hình "scrape-config" trong Prometheus: Cách này là cách truyền thống và thủ công, rất khó để quản lý nếu số lượng Job lớn thì file config sẽ rất dài và khó đọc. Hơn nữa, mỗi lần update config mới thì ta sẽ phải update lại Prometheus (helm upgreade release)
        - Cấu hình "server monitor": Cách này hiệu quả hơn vì ta sẽ quản lý từng Target với từng file .yaml tương ứng. Hơn nữa, mỗi lần có config mới ta cũng sẽ không cần phải update lại Prometheus (ko cần helm upgrade release)
- Khi Prometheus đã có được Metrics thì nó sẽ enrich Metrics bằng cách gán thêm các label như: namespace, jobname, servicename, ... để phân loại và ghi vào Time-Series Database của Prometheus
- Prometheus đọc các Rule (các hàm so sánh giá trị Metrics với các Threshold) để quyết định những rule nào cần được raise alert để đẩy về Alert Manager
- Alert Manager sẽ có config riêng để gửi Notification đến các Receiver khác nhau, quá trình này gọi là "route". Alert Manager hỗ trợ khá nhiều các Receiver như: Email, Slack, MsTeams, Telegram

# Prometheus Job và Service Monitor
## Prometheus Job
Job là nơi cấu hình cách lấy Metrics của Target (Applications, Nodes), Cấu hình của Job được đặt trong "scrape_config" của Prometheus.
Nếu không cấu hình Job tại scrape_config thì config sẽ được load mặc định từ global.
Với "kube-prometheus-stack" helm chart thì việc khai báo Job trong scrape_config sẽ được khai báo ở dưới session `additionalScrapeConfigs`
```
    additionalScrapeConfigs:
    - job_name: minio-job
        scrape_interval: 30s
        scrape_timeout: 5s
        metrics_path: /minio/v2/metrics/cluster
        scheme: http
      static_configs:
      - targets: ['minio.monitor.svc.cluster.local:9000']
```

Trong đó: <br>
**job_name:** Prometheus Job
**scrape_interval:** Thời gian lấy Metrics là mỗi 30s (mỗi 30s, Prometheus gọi tới endpoint của application Minio để lấy Metrics)
**scrape_timeout:** Thời gian timedout là 5s, (sau 5s, Prometheus sẽ tính là timeout nếu không có Metrics được trả về)
**metrics_path:** đường dẫn để lấy Metrics
**target:** endpoint để truy cập tới application Minio

**NOTE: với application cùng namespace với Prometheus thì chỉ cần trỏ đến tên application là đủ (minio), thay vì phải trỏ đến FQDN (minio.monitor.svc.cluster.local)**

Upgrade lại `kube-prometheus-stack` helm chart để cập nhật config mới
```
helm upgrade prometheus-stack -f value-prometheus.yaml kube-prometheus-stack -n monitor
```

Bài toán đặt ra là khi cần lấy Metric của 100 application, thì tương ứng mỗi application ta phải cấu hình một Job trong scrapeConfig này. Chưa kể sau khi viết thêm Job bạn sẽ phải update Prometheus để nó apply config mới bằng cách upgrade helm chart.
Như vậy, mỗi khi có Job mới để lấy Metrics từ application mới thì ta phải chỉnh sửa file value.yaml của helm chart và upgrade helm chart để apply config mới

Đây là lúc Service Monitor phát huy tác dụng

## Service Monitor
Service Monitor là một object của K8s giúp cho Prometheus có thể tự động nhận ra được các Target cần lấy Metric. Service Monitor object sẽ chứa các thông tin về Target giống như Prometheus Job trong scrape-config vậy.
## Kiến trúc
![image](https://github.com/user-attachments/assets/3cc31bfe-b29b-4ad1-a7ef-8a7a48a202a9)

## Workflow của Service Monitor
Prometheus sẽ apply config của Job từ scrapeConfig và từ ServiceMonitor, tất cả các đối tượng này sẽ hiển thị ở phần targets của Prometheus

Apply config từ scrapeConfig là quá rõ ràng rồi, nó được cấu hình ở file config của Prometheus

Apply config từ ServiceMonitor dựa vào cấu hình của Prometheus gồm 2 tham số:

serviceMonitorNamespaceSelector: Là cấu hình chỉ đọc các ServiceMonitor ở một số namespace nhất định, by default là đọc từ tất cả namespace

serviceMonitorSelector: Là cấu hình về cách chọn các ServiceMonitor sẽ được đọc để lưu cấu hình Prometheus Job. Ta có thể cấu hình theo nhiều rule khác nhau. Ở đây ta chọn cách cấu hình theo `label` của ServiceMonitor là cách đơn giản nhất

Trong cấu hình của ServiceMonitor sẽ có thông tin label của nó (dùng cho Prometheus select như trên) và các thông tin về Target cần monitor (giống các thông tin khai báo cho Prometheus Job trong scrapeConfig vậy)

Như vậy, ta cần cấu hình lại Prometheus để update lại cấu hình ServiceMonitor selector. Ở đây mình sẽ select từ tất cả namespace, và sẽ filter các ServiceMonitor có label là `app.kubernetes.io/instance` và value của label này là dongna-service-monitor.

Ta sẽ sửa lại tham số "serviceMonitorSelector:" trong helm-value của kube-prometheus-stack (file values-prometheus.yaml) như sau:

```
    serviceMonitorSelector:
      matchExpressions:
      - key: app.kubernetes.io/instance
        operator: In
        values:
          - dongna-service-monitor
          - nginx-ingress-controller          
          - prometheus-stack
```
Như vậy, tất cả các serviceMonitor ở tất cả các namespace có label là `app.kubernetes.io/instance` và có 1 trong 3 value như trên thì sẽ được apply vào Prometheus config.<br>

Đến đây thì việc monitor thêm 100 application mới đã không còn là vấn đề nữa. Ta chỉ cần gán 1 label cho toàn bộ các serviceMonitor mới sau này trùng với cấu hình serviceMonitorSelector trong Prometheus là nó sẽ tự động được đọc và không cần phải upgrade helm chart Prometheus. Ngoài ra, với cách tiếp cận này, mỗi service cần giám sát bạn sẽ tạo một file serviceMonitor .yaml để tiện lợi cho việc quản lý, cập nhật và tái sử dụng sau này.<br>

Đây là cấu hình mẫu của ServiceMonitor để giám sát application Minio, khá giống với cấu hình scrapeConfig, ta tạo file serviceMonitor-minio.yaml với nội dung như sau:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: minio-monitor
  # Change this to the namespace the Prometheus instance is running in
  namespace: monitor
  labels:
    app.kubernetes.io/instance: dongna-service-monitor    
spec:
  endpoints:
  - port: 9000
    interval: 15s
    scheme: http
    metrics_path: /minio/v2/metrics/cluster
  namespaceSelector:
    matchNames:
    - monitor
  selector:
    matchLabels:
      app.kubernetes.io/instance: minio
```

Trong đó:
namespace: namespace mà Prometheus đang chạy
app.kubernetes.io/instance: dongna-service-monitor match với 1 trong 3 serviceMonitorSelector đã khai báo trong Prometheus config
spec: thông tin Target


Tạo Service Monitor từ file .yaml
```
kubectl apply -f serviceMonitor-minio.yaml
```

Output
```
NAME                                 AGE
fluentd                              6d17h
longhorn-prometheus-servicemonitor   21d
minio-monitor                        21d
redis-cluster                        18d
```

Object serviceMonitor tên `minio-monitor` đã được deploy vào namespace `monitor`, và gán key "app.kubernetes.io/instance: dongna-service-monitor" match với serviceMonitorSelector trong cấu hình của Prometheus nên Prometheus sẽ apply vào cấu hình của nó.

Trong Object serviceMonitor này sẽ có session `spec` chính là cấu hình thông tin lấy Metric từ Target cần giám sát, gồm endpoints (port/metric_path) và thông tin object (namespaceSelector và selector)

Chi tiết hơn, ta có thể get service ở namespace `monitor` với label là `app.kubernetes.io/instance=minio`

```
kubectl get service -n monitor -l "app.kubernetes.io/instance=minio"
```

**NOTE:** Target có 2 loại: Target hỗ trợ metric hoặc Target cần dùng exporter.

Trong trường hợp này minio hỗ trợ sẵn metric nên Prometheus sẽ pull Metrics bằng cách trỏ tới application Minio ở port và path mà nó cung cấp metric (endpoint).

Trường hợp application không hỗ trợ metric mà phải dùng exporter thì ta sẽ cần cấu hình phần endpoints và selector trỏ tới service exporter.


## Cài đặt application hỗ trợ sẵn cả Metric và Service Monitor (Redis)
Có nhiều open source cài qua helm-chart hỗ trợ cả Metric và template để tạo serviceMonitor + Prometheus Rule. <br>
Ta sẽ cài thử redis bằng helm.

```
mkdir -p /home/sysadmin/open-sources
cd /home/sysadmin/open-sources
mkdir redis
cd redis
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo redis
helm pull bitnami/redis-cluster --version 7.5.1
tar -xzf redis-cluster-7.5.1.tgz
cp redis-cluster/values.yaml values-redis.yaml
vim values-redis.yaml
```

Chỉnh sửa file `value-redis.yaml`
```
usePassword: false
redis: "30101"
type: NodePort
storageClass: "longhorn-storage-delete"
size: 1Gi

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: "monitor"
    labels:
      app.kubernetes.io/instance: dongna-service-monitor
```

Cài redis helm chart lên namespace `prod`
```
helm install redis-cluster -f values-redis.yaml -n prod
```

Kiểm tra lại cài đặt
```
kubectl get pod -n prod -l "app.kubernetes.io/instance=redis-cluster"
kubectl get servicemonitors.monitoring.coreos.com redis-cluster -n monitor
```
Ta đã có một cụm redis-cluster gồm 6 pods, 3 service ở namespace prod và một object serviceMonitor là `redis-cluster` được tạo ở namespace `monitor` (theo đúng những gì ta cấu hình ở file values-redis.yaml)<br>
Kiểm ra xem serviceMonitor của redis có gì:

```
kubectl describe describe servicemonitors.monitoring.coreos.com redis-cluster -n monitor
```
Output
```

```

Trong đó:<br>
**Labels: app.kubernetes.io/instance=dongna-service-monitor:** Label để Prometheus lựa chọn đọc thông tin từ serviceMonitor này. Với mỗi serviceMonitor mới ta sẽ đều thêm label này vào để Prometheus tự động đọc.
**Spec:** Spec của Target cần get Metric. Ở đây nó tự hiểu cần trỏ đến application ở namespace `prod` mà có gán các Label như bên dưới, ở port tên là metrics.
```
Spec:
  Endpoints:
    Port:  metrics
  Namespace Selector:
    Match Names:
      prod
  Selector:
    Match Labels:
      app.kubernetes.io/component:  metrics
      app.kubernetes.io/instance:   redis-cluster
      app.kubernetes.io/name:       redis-cluster
```
Verify lại thông tin service được chỉ định theo spec trên bằng lệnh:
```
kubectl get service -l "app.kubernetes.io/component=metrics" -l "app.kubernetes.io/instance=redis-cluster" -l "app.kubernetes.io/name=redis-cluster" -n prod
```

Quay lại trang Prometheus UI xem application redis-cluster đã được hiển thị trong phần Targets hay chưa:


Tiếp theo, ta import dashboard cho redis-cluster lên Grafana



Tương tự như redis, khi cài đặt NGINX-Ingress controller thì ta có thể enable metric và serviceMonitor cho nó (tuỳ chỉnh trong file value.yaml) sau đó cũng tạo dashboard trên Grafana để hiển thị metrics

**Bonus:**
Nếu sử dụng Longhorn storage cho K8s, có thể sẽ gặp phải vấn đề cấu hình scrapeConfig xong thì Prometheus sẽ chỉ lấy được Metrics của 1 node (trong 3 node mà ta config).<br>
Muốn lấy Metrics của Longhorn thì buộc phải sử dụng serviceMonitor để Prometheus get Metrics đầy đủ từ tất cả các longhorn node

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: longhorn-prometheus-servicemonitor
  namespace: monitoring
  labels:
    name: longhorn-prometheus-servicemonitor
    release:  prometheus-grafana-stack
    k8s-app: longhorn
    app: kube-prometheus-stack
    app.kubernetes.io/instance: dongna-service-monitor
spec:
  selector:
    matchLabels:
      app: longhorn-manager
  namespaceSelector:
    matchNames:
    - storage
  endpoints:
  - port: manager  
```

Sau đó tìm dashboard cho Grafana (keyword: longhorn example v1.1.0) để hiển thị thông tin lên dashboard:

