# Giới thiệu
- Logging trên K8s có gì khác so với trên Server?
- Khi ta deploy application lên K8s thì application sẽ được chạy với nhiều Pods (Replicas), các Replicas này sẽ được K8s schedule lên các Worker Node 1 cách tự động, khi đó ta sẽ gặp phải các vấn đề sau
  - Nếu muốn xem log của Pod thì ta sẽ dùng `kubectl logs <pod_name> -n <namespace>`, vậy nếu application được deploy với 5 Replicas (Pods) thì sao
  - Hoặc nếu ta muốn xem log file của 1 container trên Node thì việc đầu tiên là ta phải check xem Pod đó đang chạy trên Worker Node nào, sau đó ssh vào Worker Node đó và check log tại /var/log/containers

- Để giải quyết các vấn đề trên thì logging soulution được ra đời
- Hiện tại, có các logging solution phổ biến như: ELK, EFK, Promtail/Loki, ...

# ELK (ElasticSearch - Logstash - Kibana)
## Architecture
![image](https://github.com/user-attachments/assets/6b25bdf8-7a8b-474b-89d0-1b20b0a4ffb9)

## ELK components
### Beat (Filebeat, Metricbeat)
- Chịu trách nhiệm thu thập logs từ các Targets trên K8s
- Output của Beats sẽ đẩy sang cho Logstash để xử lý và tổng hợp logs trước khi lưu vào ElasticSearch hoặc cũng có thế lưu trực tiếp vào ElasticSearch mà không cần thông qua Logstash để parse log

### Logstash: 
- Chịu trách nhiệm xử lý và tổng hợp logs (Data Aggregation & Data Processing) từ các Targets
- Ta define các `pipeline` để xử lý input data (logs) bằng các filter-plugin (ta có thể tùy chọn với từng loại log input để thêm một số trường mới, hoặc xử lý các dữ liệu log không có cấu trúc thành các dạng log có cấu trúc bằng các pattern (thường xử lý regex))
**Lưu ý rằng bước cấu hình log parsing là khó nhất trong việc xây dựng một hệ thống logging**

### ElasticSearch
- Chịu trách nhiệm lưu trữ log (Database lưu log) để phục vụ cho việc query logs sau này

### Kibana
- Chịu trách nhiệm query data từ ElasticSearch và hiển thị theo các template một cách trực quan cho người dùng theo dạng list hoặc graph

**NOTE: Trong trường hợp chỉ muốn lưu trữ log mà không cần parsing log, ta có thể bỏ qua Logstash và lưu trực tiếp log từ Beat đến ElasticSearch**<br>
![image](https://github.com/user-attachments/assets/73025db2-73f0-4ffd-b733-60f37ac6501d)

# EFK (ElasticSearch - Fluentd - Kibana)
## Architecture
![image](https://github.com/user-attachments/assets/3edb2b85-daa5-4300-826c-b23dc030a9f7)

- Nói tới EFK thì không thể không đề cập tới 2 điểm sau:
  - Fluentbit và Fluentd có performance rất tốt, và sử dụng rất ít tài nguyên
  - Fluentd được chứng nhận bởi CNCF, do đó xu hướng logging trên K8s sẽ ưu tiên dùng thằng EFK hơn ELK
- Các bạn có thể tham khảo thêm các bài viết so sánh giữa Fluentd và Logstash ở đây: https://www.openlogic.com/blog/fluentd-vs-logstash#:~:text=FluentD and Logstash are both,offers many benefits over Logstash.

## EFK components
### Fluentbit/Fluentd: 
- Cả Fluentbit và Fluentd đều có thể thu thập log từ Targets trên K8s. Khác biệt lớn nhất là
  - Fluentbit chuyên dùng để thu thập logs. (tương đương Filebeat của ELK)
  - Fluentd chuyện dùng để xử lý và tổng hợp logs. (tương đương Logstash của ELK)

### ElasticSearch:
- Chịu trách nhiệm lưu trữ log (Database lưu log) để phục vụ cho việc query logs sau này

### Kibana:
- Chịu trách nhiệm query data từ ElasticSearch và hiển thị theo các template một cách trực quan cho người dùng theo dạng list hoặc graph

**NOTE: Trong trường hợp chỉ muốn lưu trữ log mà không cần parsing log, ta có thể bỏ qua Fluentd và lưu trực tiếp log từ Fluentbit đến ElasticSearch**
![image](https://github.com/user-attachments/assets/64583779-3a2d-48f0-ac75-4d54b854bbb0)
