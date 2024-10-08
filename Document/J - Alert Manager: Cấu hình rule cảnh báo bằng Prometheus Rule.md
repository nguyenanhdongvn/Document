# Giới thiệu
## Mục tiêu:<br>
- Hiểu cách cấu hình rule cho service
- Hiểu cách sử dụng Prometheus Rule để cấu hình rule cho service
- Hiểu cách cấu hình AlertManager để gửi cảnh báo
- Cách trouble-shoot các vấn đề phát sinh trong khi cấu hình
- Trước hết các bạn hãy cùng review lại về workflow lấy metric và gửi cảnh báo của Prometheus và AlertManager như sau:
![image](https://github.com/user-attachments/assets/6ea14b80-b658-46f7-b81f-8b0967b996e0)

## Monitoring workflows
- Prometheus pull metric từ Targets rồi lưu vào Time-Series Database của Prometheus. 
- Prometheus đọc các rule (là các regex so sánh Metric với các threshold) để quyết định những rule nào cần Alert để đẩy về Alert Manager.
  Có 2 cách để cấu hình rule cho Prometheus trong bộ kube-prometheus-stack này, đều bằng cách tùy biến file helm-value của stack này:
  - Cấu hình trực tiếp vào tham số additionalPrometheusRules:
  ```
  additionalPrometheusRules:
  - name: my-rule-file
  groups:
    - name: my_group
      rules:
      - record: my_record
        expr: 100 * my_record
  ```
  - Sử dụng đối tượng PrometheusRule để khai báo rule cho service. Để làm được việc này thì ta cần cấu hình tham số ruleNamespaceSelector và ruleSelector để chỉ định cách Prometheus đọc các Prometheus Rule của K8S.
- Alert Manager sẽ có config riêng thực hiện việc route alert tới các receiver khác nhau. Receiver hỗ trợ khá đa dạng từ Email, Slack, MS Teams, Telegram...

# Cấu hình rule trong file value.yaml của helm chart
**Not recommend:** Cấu hình các rule và dưới session `additionalPrometheusRules`
- Khi số lượng rule lớn, file value.yaml của helm chart sẽ trở nên dài và rối, khó quản lý
- Mỗi khi tạo thêm rule thì phải chỉnh sửa bằng tay file value.yaml của helm chart và update lại helm chart bằng lệnh helm upgrade
- Khó troubleshoot khi cấu hình bị sai syntax sẽ không update dc vào Prometheus sẽ mất thời gian debug


# Cấu hình rule bằng Prometheus Rule
**Recommend:** Cách cấu hình Prometheus Rule cũng tương tự như cách cấu hình serviceMonitor:

- Prometheus sẽ load các Prometheus Rule ở các namespace nhất định và match với các label nhất định
- Với mỗi application cần khai báo Rule, ta sẽ tạo một file Prometheus Rule .yaml, trong đó có 2 phần quan trọng:
  - Label gán cho object Prometheus Rule này phải match với ruleSelector đã cấu hình ban đầu để đảm bảo khi Prometheus Rule được tạo ra thì sẽ được tự động load vào cấu hình Rule của Prometheus
  - Các cấu hình Rule cảnh báo của application (là các regex so sánh các Metrics với các Threshold để sinh Alert)

**Cấu hình rule bằng Prometheus Rule giải quyết được vấn đề khó quản lý như đã đề cập bên trên.<br>
Ta chỉ cần quản lý các file rule .yaml cho từng service, khi cần update thì chỉnh sửa trên file yaml và cập nhật vào K8s bằng lệnh `kubectl apply`. <br>
Ta hoàn toàn có thể tái sử dụng các rule này cho cùng application khi deploy cho các project khác.**

# Cấu hình ruleSelector cho Prometheus

- Về lý thuyết mình cần cấu hình 2 tham số:

  - `ruleNamespaceSelector`: Để chỉ cho Prometheus sẽ load các Prometheus Rule ở namespace nào. By default, Prometheus sẽ load rule ở tất cả các namespace
  - `ruleSelector`: Filter các Prometheus Rule sẽ được load vào Prometheus. Ở đây ta sẽ filter theo label và load tất cả các Prometheus Rule có label với `key: app` và có value trong (`operator: In`) các value sau: `kube-prometheus-stack`, `dongna-prometheus-rule`.
- Giải thích thêm là khi ta cài bộ kube-prometheus-stack thì đi kèm với nó đã có các rule mặc định và gán label "app=kube-prometheus-stack, còn label "app=dongna-prometheus-rule" là ta gán cho các Prometheus Rule được tạo sau này.

- Giờ ta chỉnh sửa parameter trong file value.yaml của helm chart `kube-prometheus-stack` như sau:
```
ruleNamespaceSelector: {}

ruleSelector:
  matchExpressions:
  - key: app
    operator: In
    values:
    - dongna-prometheus-rule
    - kube-prometheus-stack  
```

Update lại helm chart

```
helm upgrade prometheus-grafana-stack -f values-prometheus.yaml kube-prometheus-stack -n monitor
```

Sau khi apply, Prometheus sẽ load tất cả các object Prometheus Rule ở tất cả các namespace có label là "app=kube-prometheus-stack hoặc "app=dongna-prometheus-rule"

# Cấu hình Prometheus Rule
Khi cấu hình Prometheus Rule mới, phải khai báo label match với ruleSelector của Prometheus, cụ thể trong trường hợp này cần gán label là "app=dongna-prometheus-rule"

Ta khai báo file minio-AlertRule.yaml có nội dung như sau:
```
cat <<EOF > /home/sysadmin/kubernetes_installation/prometheus/minio-AlertRule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    app: dongna-prometheus-rule #Label này match với cấu hình ruleSelector để nó được load tự động vào Prometheus
    role: alert-rules
  name: dongna-minio-prometheus-rule #Đây là tên hiển thị trong danh sách PrometheusRule trên K8S
spec:
  groups:
  - name: "dongna-minio-rule" #Tên Rule hiển thị trên mục Rule của Prometheus trên web Prometheus
    rules:
    - alert: MinioDiskOffline #Tên của cảnh báo
      for: 1m #Thời gian đạt điều kiện trước khi sinh cảnh báo. 
      expr: minio_cluster_nodes_offline_total != 1 #Điểu kiện so sánh để sinh cảnh báo
      annotations:
        message: Minio Disk offline
EOF
```

Tạo object Prometheus Rule ở namespace `monitor`
```
kubectl apply -f minio-AlertRule.yaml -n monitor
```

**Giải thích:**
- Sau bước trên, ta đã có object Prometheus Rule trên K8s tên là dongna-minio-prometheus-rule ở namespace monitor
```
kubectl get PrometheusRule -n monitor
NAME                                                              AGE
dongna-minio-prometheus-rule                                      24h
```

- Object Prometheus Rule này có label "app=dongna-prometheus-rule" nên sẽ được load vào cấu hình rule của Prometheus
- Phần `Spec` có nội dung là "nếu giá trị minio_cluster_nodes_offline_total khác 1 trong vòng 1 phút trở lên thì sẽ tạo Alert tên là MinioDiskOffline và có message là "Minio Disk offline"


Kiểm tra trên Prometheus UI xem rule đã được load vào Prometheus chưa
![image](https://github.com/user-attachments/assets/bdde58d8-a529-435f-b273-a4ffb0494094)

Như vậy, Prometheus đã load rule thành công. Ta kiểm tra giá trị của metric `minio_cluster_nodes_offline_total` đang là bao nhiêu:
![image](https://github.com/user-attachments/assets/90b90182-253d-4cfa-bc6c-cf967757035f)

Giá trị `minio_cluster_nodes_offline_total=0` do đó về lý thuyết thì sau 1 phút sẽ có cảnh báo.

Ta tiếp tục kiểm tra Alert Manager xem có xuất hiện cảnh báo hay chưa:
![image](https://github.com/user-attachments/assets/178a9eda-f6ea-4bdc-bfba-d396d5609938)

**Khi ta cấu hình Alert Rule và khi application cần monitor có Alert thì ta không thể theo dõi dashboard của alert manager 24/7 được. Ta cần phải gửi Notification qua một kênh thông báo như: MS Teams, Telegram , Email để mỗi khi có Alert thì ta sẽ được báo về các kênh thông báo đó.**

# Cấu hình Alert Manager gửi Notification qua Email
## Cấu hình Gmail
- Cần có 1 gmail dùng làm Notification Sender. Sau đó, ta setup App Password để Alert Manager dùng App Password kết nối vào Gmail để gửi email. Ngoài ra, Alert Manager phải có internet để kết nối với gmail.

- Để cấu hình App Password, ta làm theo các bước như sau:
  - Đăng nhập vào gmail dùng làm Notification Sender, chọn Manage your Google Account
  ![image](https://github.com/user-attachments/assets/fb929200-7a4a-4114-a885-adf69a2fcdf7)

  - Chọn Security, tìm `App Passwords` trên search bar (trong trường hợp ko thấy mục App Password thì có thể do bạn chưa cấu hình 2-Step Verification. Bạn hãy cấu hình 2-Step Verification trước rồi quay lại thực hiện lại bước này).
  ![image](https://github.com/user-attachments/assets/cd323b8d-e899-4909-859b-e97ef91cfe2a)

  - Sau khi thực hiện xong thì Copy password mới được tạo ra về để lát nữa sẽ dùng trong cấu hình của AlertManager.
  ![image](https://github.com/user-attachments/assets/96a20b3e-28ee-4041-9969-f94bcff8455c)


## Cấu hình Alert Manager
- Alert Manager có các parameter quan trọng như: Route và Receiver.
- Cấu hình Route có nhiệm vụ phân loại các Alert theo các Lable để route tới các Receiver khác nhau.
- Usecase cho cấu hình Route:
  - Routing theo namespace để Route Alert đến người quản lý namespace tương ứng đó:
    - Alert từ namespace "dev" thì gửi mail tới Receiver là "dev-team", Receiver này là gửi email tới alias mail của dev team
    - Alert từ namespace "prod" thì gửi mail tới Receiver là "ope-team", Receiver này là gửi email tới alias mail của operation team
  - Routing theo loại service để Route Alert đến người quản lý application tương ứng đó:
    - Alert liên quan tới Database thì gửi mail tới Receiver là "dba-team", Receiver này là alias mail của dba team
    - Alert liên quan tới opensource khác (ngoài DB) thì gửi mail tới Receiver là "devops-team", Receiver này là alias mail của devops team
    - Alert liên quan tới inhouse application thì gửi mail tới Receiver là "dev-team",  Receiver này là alias mail của dev team

- Trong cấu hình Receiver thì tùy từng loại (email, webhook..) mà có các tham số cấu hình khác nhau nhưng nói chung nó chứa thông tin để gửi cảnh báo tới nơi nhận đó.

- Để cấu hình cho Alert Manager gửi Notification qua Gmail và Telegram thì ta cần update lại file value.yaml của Prometheus helm chart:
```
alertmanager:
  enabled: true
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['namespace','job']
      group_wait: 5s
      group_interval: 5s
      repeat_interval: 5m
      receiver: 'gmail'
      routes:
      - receiver: 'gmail'
        group_wait: 10s
        matchers:
        -  namespace="monitor"
    receivers:
    - name: 'gmail'
      email_configs:
      - to: notificationlab.receiver@gmail.com
        from: notificationlab.sender@gmail.com
        smarthost: smtp.gmail.com:587
        auth_username: 'notificationlab.sender@gmail.com'
        auth_identity: 'notificationlab.sender@gmail.com'
        auth_password: 'jvsx helb hnsz iimv'
    templates:
    - '/etc/alertmanager/config/*.tmpl'
```

- Update Prometheus helm chart để apply config mới
```
helm upgrade prometheus-grafana-stack -f values-prometheus.yaml kube-prometheus-stack -n monitor
```
**Giải thích:**
- By default, Alert sẽ được gán vào Receiver là gmail, nếu nó match bất cứ Route nào tiếp theo thì nó sẽ được đổi sang cấu hình Receiver của Route đó
- Ta có cấu hình một Route là với các Alert gửi từ `namespace="monitor"` sẽ được route tới Receiver là gmail 
- Trong cấu hình Receiver thì ta cấu hình thông tin email gồm:
  - `to`: Địa chỉ email của Notification Receiver
  - `from`: địa chỉ email của Notification Sender
  - `smarthost`: địa chỉ server mail
  - `auth_username`, `auth_identity`:  địa chỉ gmail Sender
  - `auth_password`: là App Password đã tạo cho Alert Manager trong gmail
