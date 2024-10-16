# Giới thiệu
- Ở session này ta sẽ deploy ELK stack lên K8s
**NOTE: ElasticSearch - Logstash - Kibana PHẢI CÙNG VERSION với nhau**

# Deploy ELK stack
- Tạo folder cài đặt cho ELK
```
mkdir /home/sysadmin/open-sources/ELK && cd /home/sysadmin/open-sources/ELK
helm repo add elastic https://Helm.elastic.co
helm search repo elastic/elasticsearch
helm pull elastic/elasticsearch --version 8.5.1
tar -xzf elasticsearch-8.5.1.tgz
cp elasticsearch/values.yaml value-elasticsearch.yaml
```

- Cấu hình file value-elasticsearch.yaml
```
########################### Config resource cho ES Pod
  resources:
    requests:
     cpu: "1000m"
      memory: "2Gi"
    limits:
      cpu: "1000m"
     memory: "2Gi"

########################### Config antiAffinity (chính sách phân bổ pod trên worker node):
  # Hard means that by default pods will only be scheduled if there are enough nodes for them
  # and that they will never end up on the same node. Setting this to soft will do this "best effort"
 antiAffinity: "soft"

########################### Config Ingress
 ingress:
   enabled: true
    annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
    className: "nginx"
    pathtype: ImplementationSpecific
    hosts:
     - host: elasticsearch.dongna.com
        paths:
          - path: /

########################### Config PV
persistence:
  enabled: true
  labels:
    # Add default labels for the volumeClaimTemplate of the StatefulSet
    enabled: false
  annotations: {}

########################### Config PVC
volumeClaimTemplate:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 30Gi

``` 

Trong đó:
`antiAffinity: "soft"`: mặc định là "hard". Khi set "hard", K8s sẽ bắt buộc các Pod của ElasticSearch không được schedule trên cùng một Node. Trong trường hợp lab chỉ có 2 Worker Node nhưng lại muốn chạy ElasticSearch Cluster với 3 Replicas thì set "hard" sẽ bị lỗi. Do đó trong môi trường lab thì nên set là "soft"

Nếu muốn lưu data ra Node thì enable persistence lên. NOTE: ElasticSearch không có cấu hình chọn Storage Class, nó sẽ dùng Storage Class default của K8s. Nếu muốn ElasticSearch sử dụng Storage Class nào thì phải set Storage Class default cho K8s trước khi cài đặt ElasticSearch helm chart

Ví dụ hệ thống của mình đang có sẵn 6 storage class như sau:
```
NAME                                PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
dongna-nfs-delete                   dongna-nfs-storage-delete-provisioner   Delete          Immediate           true                   8d
dongna-nfs-retain                   dongna-nfs-storage-retain-provisioner   Retain          Immediate           true                   8d
longhorn (default)                  driver.longhorn.io                      Delete          Immediate           true                   8d
longhorn-static                     driver.longhorn.io                      Delete          Immediate           true                   42m
longhorn-storage-delete (default)   driver.longhorn.io                      Delete          Immediate           true                   8d
longhorn-storage-retain             driver.longhorn.io                      Retain          Immediate           true                   8d
```
Hiện tại Storage Class default đang là `longhorn-storage-delete` và `longhorn`. Nếu ta muốn sử dụng `dongna-nfs-retain` Storage Class cho ElasticSearch thì: 

- Set `dongna-nfs-retain` Storage Class thành default
```
kubectl patch storageclass dongna-nfs-retain -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
- Set `longhorn-storage-delete` và `longhorn` Storage Class thành non-default
```
kubectl patch storageclass longhorn-storage-delete -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

- Kết quả là `dongna-nfs-retain` Storage Class đã thành default Storage Class
```
NAME                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
dongna-nfs-delete             dongna-nfs-storage-delete-provisioner   Delete          Immediate           true                   8d
dongna-nfs-retain (default)   dongna-nfs-storage-retain-provisioner   Retain          Immediate           true                   8d
longhorn                      driver.longhorn.io                      Delete          Immediate           true                   8d
longhorn-static               driver.longhorn.io                      Delete          Immediate           true                   47m
longhorn-storage-delete       driver.longhorn.io                      Delete          Immediate           true                   8d
longhorn-storage-retain       driver.longhorn.io                      Retain          Immediate           true                   8d
```

- Cài đặt ElasticSearch
```
helm -n prod install elasticsearch -f value-elasticsearch.yaml elasticsearch
```

