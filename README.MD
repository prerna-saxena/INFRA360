Setting Up Minikube
Install Minikube:

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

Start Minikube:
minikube start

 Deploy EFK Stack on Minikube
kubectl create namespace efk

Deploy Elasticsearch:
Create a file named elasticsearch-deployment.yaml


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



Apply the deployment:

kubectl apply -f elasticsearch-deployment.yaml


 Deploy Kibana
Create a file named kibana-deployment.yaml

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


Apply the deployment:
kubectl apply -f kibana-deployment.yaml

apiVersion: v1
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


Create a file named fluentd-daemonset.yaml:

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


Access Kibana UI
Get the Kibana service URL:
minikube service kibana -n efk --url


Open Kibana in browser:
Navigate to the URL obtained from the above command.

Configure Kibana
Create an Index Pattern:

Go to Management > Stack Management > Index Patterns.
Click Create index pattern.
Enter fluentd-* as the index pattern and click Next.
Choose the time field (e.g., @timestamp) and click Create index pattern.
Visualize Logs:

Go to Discover and select the index pattern you created.
Create visualizations and dashboards based on the logs.

 README
Create a README.md file with the following content:

# EFK Stack Deployment on Minikube

## Prerequisites
- Minikube installed
- kubectl configured

Steps

1. Start Minikube
```bash
minikube start


2. Deploy Elasticsearch

kubectl create namespace efk
kubectl apply -f elasticsearch-deployment.yaml

3. Deploy Kibana
kubectl apply -f kibana-deployment.yaml
4. Deploy Fluentd
kubectl apply -f fluentd-configmap.yaml
kubectl apply -f fluentd-daemonset.yaml

5. Access Kibana UI
minikube service kibana -n efk --url


6. Configure Kibana
Create an index pattern fluentd-*.
Create visualizations and dashboards.
Configuration Files
elasticsearch-deployment.yaml
kibana-deployment.yaml
fluentd-configmap.yaml
fluentd-daemonset.yaml























