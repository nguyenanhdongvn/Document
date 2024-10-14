# Giới thiệu
Để thực hành phần cài đặt và cấu hình logging này, thì quan trọng là ta cần biết ta đang giải quyết bài toán nào.

Ta đặt ra bài toán logging cho hệ thống như sau:

- Hệ thống có sử dụng một vài opensource như vernemq, kafka --> Cần lưu log tập trung và lưu trữ trong 7 ngày
- Các application (do developer viết) được deploy dưới dạng deployment --> Cần lưu log tập trung trong 30 ngày
- Các log không cần parsing, tuy nhiên vẫn cần xây dựng hệ thống phân tích log, các rule parsing log sẽ được bổ sung sau
- Thực hiện khai báo các template để query dữ liệu log của từng application trên Kibana

# Idea để deploy ELK
- Deploy dongna-order application bằng K8s deployment với 5 Replicas (5 Pods)
- Deploy dongna-billing application bằng K8s deployment với 3 Replicas (3 Pods)
- Mỗi container của application trên, mỗi giây sẽ gửi log ra output các thông tin về Node, Pod Name, Pod IP, Namespace, ...
- Cài đặt opensource service gồm vernemq, kafka/zookeeper

## FileBeat
- Deploy FileBeat lên K8s bằng helm chart
- FileBeat sẽ được deploy dưới dạng Daemonset (Replicas được schedule trên tất cả các Worker Node) để collect log
- Cấu hình Input của FileBeat để collect log của application `dongna-order`, `dongna-billing` và đánh tags là `myapplication` trước khi đẩy sang Logstash
- Cấu hình Input của FileBeat để collect log của opensource serice `vernemq`, `kafka` và `zookeeper` và đánh tags là `myopensource` trước khi đẩy sang Logstash
- Input của FileBeat là đường dẫn tới log file, ta sẽ cấu hình theo prefix của tên file.
- Output của FileBeat là địa chỉ của Logstash

## Logstash:
- Deploy Logstash lên K8s bằng helm chart
- Logstash sẽ được deploy dưới dạng Deployment để nhận logs từ FileBeat gửi về rồi xử lý và tổng hợp logs, sau đó đẩy Output đến ElasticSearch
- Cấu hình Input nhận logs từ FileBeat
- Xử lý logs:
  - Với Input có Tag là `myapplication` thì đẩy vào Index có tên `logstash-myapplication-{YYYY-MM-dd}`
  - Với Input có Tag là `myopensource` thì đẩy vào Index có tên `logstash-myopensource-{YYYY-MM-dd}`
- Output của Logstash là địa chỉ của ElasticSearch và Index mà nó sẽ ghi logs vào
**NOTE: Do yêu cầu bài toán đặt ra, mỗi service được cấu hình lưu logs khác nhau. `myapplication` thì được lưu 30 ngày, trong khi `myopensource` chỉ lưu 7 ngày. Do đó ta phải tách riêng Index trên ElasticSearch để dữ thực hiện Log Rotation sau này**

## Elastic Search:
- Deploy ElasticSearch lên K8s bằng helm chart
- ElasticSearch là nhận và lưu trữ logs sau khi đã được xử lý và tổng hợp bởi Logstash
- Việc xóa logs cũ trên ElasticSearch được thực hiện bằng cách xóa các Indices cũ (theo thời gian)
- Việc xóa Indices có thể thực hiện tự động bằng tool gọi là `elastic-curator`, sẽ được đề cập trong phần sau

## Kibana:

- Deploy Kibana lên K8s bằng helm chart
- Kibana là Web UI để thực hiện các thao tác xem log và thực hiện các tác vụ thống kê.
- Cấu hình Input của Kibana là thông tin của ElasticSearch và tạo các Index pattern để đọc logs từ ElasticSearch
- Ta sẽ filer log của từng application bằng các keyword tương ứng

## Architechture của logging solution ELK
![image](https://github.com/user-attachments/assets/3aadfda1-f99e-49f0-886e-845b904c8351)
### FileBeat
- FileBeat collect log của Container trên các Worker Node ở đường dẫn `/var/log/containers`.
- Ta cấu hình FileBeat đọc tất cả file log, nếu file log có tên format như `dongna-*` thì FileBeat sẽ đánh Tag là `myapplication`, và nếu file log có tên dạng `vernemq*|kafka*|zookeeper*` thì đánh Tag là `myopensource`.

### Logstash
- Logstash nhận logs từ FileBeat gửi về.
- Với các logs có Tag là `myapplication` thì nó chuyển tới Output là Index có tên `logstash-myapplication-{YYYY-MM-dd}` trên ElasticSearch.
- Với các logs có Tag là `myopensource` thì nó chuyển tới Output là Index có tên `logstash-myopensource-{YYYY-MM-dd}` trên ElasticSearch.

**Sau này khi cần parse log thì ta sẽ thêm các rule filter vào Pipeline của Logstash sau mà không cần thay đổi cấu hình của các thành phần khác của hệ thống.**

# Deploy ELK
## Deploy VerneMQ
- Deploy Vernemq bằng helm chart
```
mkdir -p /home/sysadmin/open-sources/vernemq && cd /home/sysadmin/open-sources/vernemq
helm repo add vernemq https://vernemq.github.io/docker-vernemq
helm search repo vernemq
helm pull vernemq/vernemq --version 
tar -xzf vernemq-.tgz
cp vernemq/values.yaml value-vernemq.yaml
```
- Cấu hình file helm-value (value-vernemq.yaml):
```
additionalEnv:
  - name: DOCKER_VERNEMQ_ALLOW_REGISTER_DURING_NETSPLIT
    value: "on"
  - name: DOCKER_VERNEMQ_ALLOW_PUBLISH_DURING_NETSPLIT
    value: "on"
  - name: DOCKER_VERNEMQ_ALLOW_SUBSCRIBE_DURING_NETSPLIT
    value: "on"
  - name: DOCKER_VERNEMQ_ALLOW_UNSUBSCRIBE_DURING_NETSPLIT
    value: "on"
  - name: API_KEY
    value: "xxxxxyyyyzzzz"
  - name: DOCKER_VERNEMQ_USER_vernemq
    value: "xxxxxyyyyzzzz"
  - name: DOCKER_VERNEMQ_ACCEPT_EULA
    value: "yes"
  - name: ALLOW_UNSECURED_MODE
    value: "true"
```
- Cài đặt Vernemq 
```
helm -n prod install vernemq -f value-vernemq.yaml vernemq
```
- Output
```
NAME: vernemq
LAST DEPLOYED: Wed Jul 20 05:48:55 2022
NAMESPACE: prod
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Check your VerneMQ cluster status:
  kubectl exec --namespace prod vernemq-0 /vernemq/bin/vmq-admin cluster show

2. Get VerneMQ MQTT port
  Subscribe/publish MQTT messages there: 127.0.0.1:1883
  kubectl port-forward svc/vernemq 1883:1883
```

## Deploy Zookeeper
- Deploy Zookeeper bằng helm chart
```
mkdir -p /home/sysadmin/open-sources/bitnami-zookeeper && cd /home/sysadmin/open-sources/bitnami-zookeeper
helm repo add bitnami https://charts.bitnami.com/bitnami
helm pull bitnami/zookeeper --version
tar -xzf zookeeper-.tgz
cp zookeeper/values.yaml value-zookeeper.yaml
```
- Cấu hình file helm-value (value-zookeeper.yaml):
```
persistence:
  enabled: false
  # storageClass: "-"
  accessModes:
    - ReadWriteOnce
  size: 8Gi
  annotations: {}
```
- Cài đặt Zookeeper 
```
helm -n prod install zookeeper -f value-zookeeper.yaml zookeeper
```
- Output
```
NAME: zookeeper
LAST DEPLOYED: Wed Jul 20 05:57:37 2022
NAMESPACE: prod
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

ZooKeeper can be accessed via port 2181 on the following DNS name from within your cluster:

    zookeeper.prod.svc.cluster.local

To connect to your ZooKeeper server run the following commands:

    export POD_NAME=$(kubectl get pods --namespace prod -l "app.kubernetes.io/name=zookeeper,app.kubernetes.io/instance=zookeeper,app.kubernetes.io/component=zookeeper" -o jsonpath="{.items[0].metadata.name}")
    kubectl exec -it $POD_NAME -- zkCli.sh

To connect to your ZooKeeper server from outside the cluster execute the following commands:

    kubectl port-forward --namespace prod svc/zookeeper 2181:2181 &
    zkCli.sh 127.0.0.1:2181
```

## Deploy Kafka
- Deploy Kafka bằng helm chart
```
mkdir -p /home/sysadmin/open-sources/bitnami-kafka && cd /home/sysadmin/open-sources/bitnami-kafka
helm repo add bitnami https://charts.bitnami.com/bitnami
helm pull bitnami/kafka --version 
tar -xzf kafka-.tgz
cp kafka/values.yaml value-kafka.yaml
```
- Cấu hình file helm-value (value-kafka.yaml):
```
  ## Persistence parameters
  ##
  persistence:
     enabled: false
    ## A manually managed Persistent Volume and Claim
    ## If defined, PVC must be created manually before volume will be bound
  zookeeper:
    enabled: false
    auth:
      ## Enable Zookeeper auth
  externalZookeeper:
    ## Server or list of external zookeeper servers to use.
    ##
   servers: zookeeper.prod.svc.cluster.local
```
- Cài đặt Kafka 
```
helm -n prod install kafka -f value-kafka.yaml kafka
```
- Output
```
NAME: kafka
LAST DEPLOYED: Wed Jul 20 06:02:16 2022
NAMESPACE: prod
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka.prod.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-0.kafka-headless.prod.svc.cluster.local:9092

To create a pod that you can use as a Kafka client run the following commands:

    kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:2.8.0-debian-10-r0 --namespace prod --command -- sleep infinity
    kubectl exec --tty -i kafka-client --namespace prod -- bash

    PRODUCER:
        kafka-console-producer.sh \

            --broker-list kafka-0.kafka-headless.prod.svc.cluster.local:9092 \
            --topic test

    CONSUMER:
        kafka-console-consumer.sh \

            --bootstrap-server kafka.prod.svc.cluster.local:9092 \
            --topic test \
            --from-beginning
```

- Cài đặt Kafka Client để test thử pub/sub tới kafka:
```
kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:2.8.0-debian-10-r0 --namespace prod --command -- sleep infinity
```

- Public và subscribe vào topic test:
```
kubectl exec --tty -i kafka-client --namespace prod -- bash
I have no name!@kafka-client:/$
I have no name!@kafka-client:/$ kafka-console-producer.sh --broker-list kafka-0.kafka-headless.prod.svc.cluster.local:9092 --topic test
>hello
>hello2
>hello3
>^CI have no name!@kafka-client:/$
I have no name!@kafka-client:/$ kafka-console-consumer.sh --bootstrap-server kafka.prod.svc.cluster.local:9092 --topic test --from-beginning
hello
hello2
hello3
```
- Như vậy là Kafka/Zookeeper đã cài thành công và có thể gửi/nhận message bình thường.

## Deploy dongna-order và dongna-billing Application
- Tạo folder chứa manifest file của Application
```
mkdir /home/sysadmin/myapplication && cd /home/sysadmin/myapplication
```

- Tạo deployment file cho `dongna-order` application 
```
cat <<EOF > /home/sysadmin/myapplication/dongna-order-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    meta.helm.sh/release-name: dongna-order
    meta.helm.sh/release-namespace: prod
  labels:
    app.kubernetes.io/name: dongna-order
  name: dongna-order
  namespace: prod
spec:
  progressDeadlineSeconds: 600
  replicas: 5
  selector:
    matchLabels:
      app.kubernetes.io/instance: dongna-order
      app.kubernetes.io/name: dongna-order
  template:
    metadata:
      annotations:
      labels:
        app.kubernetes.io/instance: dongna-order
        app.kubernetes.io/name: dongna-order
    spec:
      containers:
        - name: order
          image: busybox
          imagePullPolicy: IfNotPresent
          args: [/bin/sh, -c, while true; do echo "$(date) mynode=$MY_NODE_NAME podName=$MY_POD_NAME namespace=$MY_POD_NAMESPACE podIp=$MY_POD_IP serviceAccountName=$MY_POD_SERVICE_ACCOUNT";  sleep 1;  done]
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

- Tạo deployment file cho `dongna-billing` application
```
cat << EOF > /home/sysadmin/myapplication/dongna-billing-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    meta.helm.sh/release-name: dongna-billing
    meta.helm.sh/release-namespace: prod
  labels:
    app.kubernetes.io/name: dongna-billing
  name: dongna-billing
  namespace: prod
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/instance: dongna-billing
      app.kubernetes.io/name: dongna-billing
  template:
    metadata:
      annotations:
      labels:
        app.kubernetes.io/instance: dongna-billing
        app.kubernetes.io/name: dongna-billing
    spec:
      containers:
        - name: billing
          image: busybox
          imagePullPolicy: IfNotPresent
          args: [/bin/sh, -c, while true; do echo "$(date) mynode=$MY_NODE_NAME podName=$MY_POD_NAME namespace=$MY_POD_NAMESPACE podIp=$MY_POD_IP serviceAccountName=$MY_POD_SERVICE_ACCOUNT";  sleep 1;  done]
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
EOF
```

- Deploy 2 deployment này lên K8s
```
kubectl -n prod apply -f dongna-order-deployment.yaml
kubectl -n prod apply -f dongna-billing-deployment.yaml
```

- Kiểm tra lại kết quả
```
kubectl -n prod get pods -owide
```
- Output
```

```

- Check log của một Pod ta thấy nó đã in ra được các thông tin của Pod đang chạy như Pod Name, Pod Ip, Namespace...
```
Wed Jul 20 10:23:14 UTC 2022 mynode=dongna-worker3 podName=dongna-order-777d5f86df-6m8n2 namespace=prod podIp=10.233.68.6 serviceAccountName=default
Wed Jul 20 10:23:15 UTC 2022 mynode=dongna-worker3 podName=dongna-order-777d5f86df-6m8n2 namespace=prod podIp=10.233.68.6 serviceAccountName=default
Wed Jul 20 10:23:16 UTC 2022 mynode=dongna-worker3 podName=dongna-order-777d5f86df-6m8n2 namespace=prod podIp=10.233.68.6 serviceAccountName=default
Wed Jul 20 10:23:17 UTC 2022 mynode=dongna-worker3 podName=dongna-order-777d5f86df-6m8n2 namespace=prod podIp=10.233.68.6 serviceAccountName=default
Wed Jul 20 10:23:18 UTC 2022 mynode=dongna-worker3 podName=dongna-order-777d5f86df-6m8n2 namespace=prod podIp=10.233.68.6 serviceAccountName=default
```

- Kiểm tra log file sinh ra trên các worker node Ví dụ pod dongna-order-777d5f86df-d96fd đang chạy trên node dongna-worker2 --> Ta vào node đó kiểm tra:
```
ls -lrt /var/log/containers/ | grep grep dongna-order-777d5f86df-d96fd
```

Như vậy việc setup môi trường cho bài toán đã xong. Hiện tại trong Namespace `prod` ta đang có 3 open source `vernemq|kafka|zookeeper` và 2 application `dongna-order|dongna-billing`
