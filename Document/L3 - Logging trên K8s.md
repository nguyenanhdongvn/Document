# Giới thiệu
- Ở session này ta sẽ deploy ELK stack lên K8s

# Deploy ELK stack
- Tạo folder cài đặt cho ELK
```
mkdir /home/sysadmin/open-sources/ELK && cd /home/sysadmin/open-sources/ELK
helm repo add elastic https://Helm.elastic.co
helm search repo elastic/elasticsearch
helm pull elastic/elasticsearch --version
tar -xzf elasticsearch-.tgz
cp elasticsearch/values.yaml value-elasticsearch.yaml
```

- Cấu hình file value-elasticsearch.yaml
```
# Config resource cho ES Pod
  esJavaOpts: "" # example: "-Xmx1g -Xms1g"
  resources:
    requests:
     cpu: "1000m"
      memory: "2Gi"
    limits:
      cpu: "1000m"
     memory: "2Gi"

# Config antiAffinity (chính sách phân bổ pod trên worker node):
  # Changing this to a region would allow you to spread pods across regions
  antiAffinityTopologyKey: "kubernetes.io/hostname"

  # Hard means that by default pods will only be scheduled if there are enough nodes for them
  # and that they will never end up on the same node. Setting this to soft will do this "best effort"
 antiAffinity: "soft"

  # This is the node affinity settings as defined in
  # https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature
  nodeAffinity: {}

# Config Ingress
 ingress:
   enabled: true
    annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
    className: "nginx"
    pathtype: ImplementationSpecific
    hosts:
     - host: elasticsearch.prod.viettq.com
        paths:
          - path: /

# Config PV
persistence:
  enabled: true
  labels:
    # Add default labels for the volumeClaimTemplate of the StatefulSet
    enabled: false
  annotations: {}

# Config PVC
volumeClaimTemplate:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 30Gi

``` 

