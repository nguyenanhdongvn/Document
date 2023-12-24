# Các thành phần chính K8s

`Control Plane`: tập hợp các Master Node

`Master Node`: dùng để quản lý, lên lịch, theo dõi các Worker Node
- `ETCD`: là database dạng key-value dùng để lưu dữ liệu của K8s Cluster như là: Node, Pod, Service, Deployment, Configmap, ...
- `Kube-Scheduler`: điều phối các pod mới lên các Woker Node phù hợp dựa trên một số tiêu chí nhất định như: resource, node affinity, ...
- `Kube-Controller-Manager`: quản lý K8s cluster với nhiều Controller khác nhau
  - `Node-Controller`: theo dõi và phát hiện Worker Node nào bị down hoặc bị destroy
  - `Replication-Controller`: đảm bảo duy trì số lượng POD đang chạy trong 1 Replication
- `Kube-Apiserver`: là bộ não của K8s Cluster, có trách nhiệm làm việc với các thành phần trong K8s Cluster


`Worker Node`: các POD sẽ được điều phối lên Worker Node để host application
- `Kubelet`: là 1 agent chạy trên từng Woker Node trong K8s Cluster (Kubelet có thể chạy trên cả Master Node), lắng nghe các chỉ thị từ Kube-Apiserver để deploy/destroy POD trên Worker Node, theo dõi và báo cáo status của các Worker Node và POD cho Kube-Apiserver một các định kỳ
- `Kube-proxy`: đảm bảo các POD trong K8s Cluster có thể giao tiếp được với nhau

![image](https://github.com/nguyenanhdongvn/Document/assets/90097692/fdfcc364-2c38-4dbb-9f30-b8d2accd8939)

