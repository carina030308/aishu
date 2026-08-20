# Flask "Hello world"

app.py
-----

from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World from Kubernetes!'

@app.route('/healthz')
def health_check():
    return 'OK', 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)


******

requirements.txt
----

Flask==3.0.3
gunicorn==22.0.0

******
dockerfile
----

FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]

*****
Docker commands
----

docker build -t hello-world-flask:v1

docker run -p 5050:5000 hello-world-flask:v1

docker tag hello-world-flask:v1 <docker-hub-username>/hello-world-flask:v1

docker login -

docker push hello-world-flask:v1 <docker-hub-username>/hello-world-flask:v1

*****

deployment.yaml
----

apiVersion: apps/v1
kind: Deployment

metadata:
  name: hello-world-deployment
  labels:
    app: hello-world

spec:
  replicas: 2

  selector:
    matchLabels:
      app: hello-world

  template:
    metadata:
      labels:
        app: hello-world

    spec:
      containers:
        - name: hello-world
          image: carina030308/hello-world-flask:v1
          ports:
            - containerPort: 5000


Then:

kubectl apply -f deployment.yaml

kubectl get deployments
kubectl get pods

*****
service.yaml
----

apiVersion: v1
kind: Service

metadata:
  name: hello-world-service

spec:
  selector:
    app: hello-world

  type: LoadBalancer

  ports:
    - port: 80
      targetPort: 5000

Apply:
kubectl apply -f service.yaml

kubectl get svc

curl http://<EXTERNAL-IP>/

Phase 2nd: 

Create monitoring Stack
-----------

Now create a separate namespace for monitoring:

kubectl create namespace monitoring

Verify: kubectl get namespaces

Check Helm:

helm version

It should be:
Helm: v4.2
Kubernetes client: v1.36

-----
Step 1 — Add Grafana Helm repository

helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

Verify:
helm repo list

You should see:

grafana  https://grafana.github.io/helm-charts

-----
Step 2 — Check Loki chart

helm search repo grafana/loki

Run these three things now:

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo grafana/loki

-----

Step: 3

loki-values.yaml
---------

deploymentMode: SingleBinary

loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 1

  storage:
    type: filesystem
    bucketNames:
      chunks: chunks
      ruler: ruler
      admin: admin

  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

singleBinary:
  replicas: 1

  persistence:
    enabled: true
    size: 10Gi

gateway:
  enabled: false

backend:
  replicas: 0

read:
  replicas: 0

write:
  replicas: 0

chunksCache:
  enabled: false

resultsCache:
  enabled: false

minio:
  enabled: false

----

Then:

helm install loki grafana/loki \
  --namespace monitoring \
  -f loki-values.yaml

Verify

Check Loki Pod Run:

kubectl get pods -n monitoring
kubectl get svc -n monitoring

-------

Install Grafana

helm install grafana grafana/grafana \
  --namespace monitoring


Get Grafana password:

kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode


Access Grafana from Cloud Shell:

kubectl port-forward -n monitoring svc/grafana 3000:80


if not allowed issue:

nano grafana-values.yaml
----

