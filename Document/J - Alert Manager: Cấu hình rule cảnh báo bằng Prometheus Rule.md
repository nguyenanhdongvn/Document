# Giới thiệu
Mục tiêu:<br>
- Hiểu cách cấu hình rule cho service
- Hiểu cách sử dụng Prometheus Rule để cấu hình rule cho service
- Hiểu cách cấu hình AlertManager để gửi cảnh báo
- Cách trouble-shoot các vấn đề phát sinh trong khi cấu hình
- Trước hết các bạn hãy cùng review lại về workflow lấy metric và gửi cảnh báo của Prometheus và AlertManager như sau:
Monitoring workflow
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

- Cấu hình Prometheus đọc các Prometheus Rule ở các namespace nhất định và match với các label nhất định
- Với mỗi application cần khai báo rule, ta sẽ tạo một file Prometheus Rule .yaml, trong đó có 2 phần quan trọng:
- - Label gán cho object Prometheus Rule này phải match với ruleSelector đã cấu hình ban đầu để đảm bảo khi Prometheus Rule được tạo ra thì sẽ được tự động load vào cấu hình Rule của Prometheus
- - Các cấu hình Rule cảnh báo của application (là các regex so sánh các Metrics với các Threshold để sinh Alert)

**Cấu hình rule bằng Prometheus Rule giải quyết được vấn đề khó quản lý như đã đề cập bên trên.<br>
Ta chỉ cần quản lý các file rule .yaml cho từng service, khi cần update thì chỉnh sửa trên file yaml và cập nhật vào K8s bằng lệnh `kubectl apply`. <br>
Ta hoàn toàn có thể tái sử dụng các rule này cho cùng application khi deploy cho các project khác.**

# Cấu hình ruleSelector cho Prometheus

Về lý thuyết mình cần cấu hình 2 tham số:

- ruleNamespaceSelector: Để chỉ cho Prometheus sẽ load các Prometheus Rule ở namespace nào. By default, Prometheus sẽ load rule ở tất cả các namespace
- ruleSelector: Filter các Prometheus Rule sẽ được load vào Prometheus. Ở đây ta sẽ filter theo label và load tất cả các Prometheus Rule có label với `key: app` và có value trong (operator: In) các value sau: "kube-prometheus-stack", "dongna-prometheus-rule".
Giải thích thêm là khi ta cài bộ kube-prometheus-stack thì đi kèm với nó đã có các rule mặc định và gán label "app=kube-prometheus-stack, còn label "app=dongna-prometheus-rule" là ta gán cho các Prometheus Rule được tạo sau này.

Giờ ta chỉnh sửa parameter trong file value.yaml của helm chart `kube-prometheus-stack` như sau:

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
helm upgrade prometheus-stack -f values-prometheus.yaml kube-prometheus-stack -n monitor
```

Sau khi apply, Prometheus sẽ đọc tất cả các object Prometheus Rule ở tất cả các namespace có label là "app=kube-prometheus-stack hoặc "app=dongna-prometheus-rule"

# Cấu hình Prometheus Rule
Khi cấu hình Prometheus Rule mới, phải khai báo label match với ruleSelector của Prometheus, cụ thể trong trường hợp này cần gán label là "app=dongna-prometheus-rule"

Ta khai báo file minio-AlertRule.yaml có nội dung như sau:

```
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
```

Tạo object Prometheus Rule ở namespace `monitor`
```
kubectl apply -f minio-AlertRule.yaml -n monitor
```

Giải thích:

Sau bước trên, ta đã có object Prometheus Rule trên K8s tên là dongna-minio-prometheus-rule ở namespace monitor


```
kubectl get PrometheusRule -n monitor
NAME                                                              AGE
dongna-minio-prometheus-rule                                      24h
```

- Object Prometheus Rule này có label "app=dongna-prometheus-rule" nên sẽ được load vào cấu hình rule của Prometheus
- Phần `Spec` có nội dung là "nếu giá trị minio_cluster_nodes_offline_total khác 1 trong vòng 1 phút trở lên thì sẽ tạo Alert tên là MinioDiskOffline và có message là "Minio Disk offline"


Kiểm tra trên Prometheus UI xem rule đã được load vào Prometheus chưa

Như vậy, Prometheus đã load rule thành công. Ta kiểm tra giá trị của metric `minio_cluster_nodes_offline_total` đang là bao nhiêu:

Giá trị `minio_cluster_nodes_offline_total=0` do đó về lý thuyết thì sau 1 phút sẽ có cảnh báo.

Ta tiếp tục kiểm tra Alert Manager xem có xuất hiện cảnh báo hay chưa:

Như vậy là đã có cảnh báo. Việc cấu hình đang rất suôn sẻ.

**Câu chuyện lại tiếp tục như sau. Khi bạn cấu hình rule và khi service cần giám sát có vấn đề và có cảnh báo thì bạn không thể nhìn vào màn hình của alert manager 24/7 được. Do đó bài toán đặt ra là cần phải gửi thông báo qua một kênh nào đó nhưng MS Teams, Telegram , Email mỗi khi có cảnh báo để người trực hệ thống nắm được vào vào xử lý một cách chủ động. Đó là lúc bạn tiếp tục phải cấu hình cho Alert Manager**

# Cấu hình Alert Manager gửi Notification qua Email
## Cấu hình Gmail
