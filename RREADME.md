README.md file detailing the steps and commands used to deploy the EFK stack on a Minikube cluster, along with the configuration files:

# EFK Stack Deployment on Minikube

This guide provides instructions to deploy the EFK (Elasticsearch, Fluentd, Kibana) stack on a Minikube Kubernetes cluster for log collection, processing, and visualization.

## Prerequisites

- Minikube installed
- `kubectl` installed and configured to use Minikube
- Basic knowledge of Kubernetes

## Steps

### 1. Start Minikube

Start your Minikube cluster if it's not already running:

```bash
minikube start

Create a Namespace for EFK
Create a dedicated namespace for the EFK stack:

kubectl create namespace efk


3. Deploy Elasticsearch
Create and apply the Elasticsearch deployment configuration:

elasticsearch-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: efk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.10.0
        env:
        - name: discovery.type
          value: single-node
        ports:
        - containerPort: 9200
        volumeMounts:
        - name: elasticsearch-storage
          mountPath: /usr/share/elasticsearch/data
      volumes:
      - name: elasticsearch-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: efk
spec:
  ports:
  - port: 9200
    targetPort: 9200
  selector:
    app: elasticsearch

Apply the configuration:
kubectl apply -f elasticsearch-deployment.yaml


4. Deploy Kibana
Create and apply the Kibana deployment configuration:

kibana-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: efk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.10.0
        ports:
        - containerPort: 5601
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: efk
spec:
  ports:
  - port: 5601
    targetPort: 5601
  selector:
    app: kibana

  Apply the configuration:

kubectl apply -f kibana-deployment.yaml


5. Deploy Fluentd
First, create a ConfigMap for Fluentd's configuration:

fluentd-configmap.yamlapiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: efk
data:
  fluentd.conf: |
    <source>
      @type kubernetes
      @id input_kubernetes
      @label @FLUENTD
      tag kubernetes.*
      <parse>
        @type json
        time_key time
      </parse>
      <storage>
        @type local
      </storage>
      <buffer>
        @type memory
      </buffer>
    </source>

    <match kubernetes.**>
      @type elasticsearch
      @id output_elasticsearch
      host elasticsearch
      port 9200
      logstash_format true
      logstash_prefix fluentd
      <buffer>
        @type memory
      </buffer>
    </match>

Apply the ConfigMap:


kubectl apply -f fluentd-configmap.yaml

Next, create and apply the Fluentd DaemonSet configuration:

fluentd-daemonset.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: efk
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.12-debian-1
        env:
        - name: FLUENTD_CONF
          value: fluentd.conf
        volumeMounts:
        - name: fluentd-config
          mountPath: /fluentd/etc
          readOnly: true
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
      volumes:
      - name: fluentd-config
        configMap:
          name: fluentd-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
Apply the DaemonSet:


kubectl apply -f fluentd-daemonset.yaml

Configure Kibana
Navigate to Management > Stack Management > Index Patterns.
Create an index pattern fluentd-*.
Set the @timestamp field as the time filter.
You can now use Discover, Visualize, and Dashboard features to analyze the logs.
Configuration Files
elasticsearch-deployment.yaml: Configuration for Elasticsearch deployment and service.
kibana-deployment.yaml: Configuration for Kibana deployment and service.
fluentd-configmap.yaml: ConfigMap for Fluentd configuration.
fluentd-daemonset.yaml: DaemonSet configuration to deploy Fluentd on all nodes.
