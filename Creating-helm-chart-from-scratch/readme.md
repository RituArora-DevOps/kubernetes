Creating a Helm chart from scratch involves organizing the necessary Kubernetes manifests and Helm-specific files to define, package, and deploy your application. Below are the steps to create a Helm chart that deploys two replicas of Redis on a Kubernetes cluster.

Step 1: Create Helm Chart Structure
Create the basic structure for your Helm chart:

bash
Copy code
mkdir redis-chart
cd redis-chart
helm create my-redis
Step 2: Modify Chart.yaml
Open my-redis/Chart.yaml and modify it with relevant information:

yaml
Copy code
apiVersion: v2
name: my-redis
description: A Helm chart for deploying Redis
version: 0.1.0
Step 3: Modify values.yaml
Open my-redis/values.yaml and modify it. Set the number of replicas and other configurations:

yaml
Copy code
replicaCount: 2

redis:
  image:
    repository: "redis"
    tag: "latest"
    pullPolicy: "IfNotPresent"
  password: ""
  port: 6379
Step 4: Modify templates/deployment.yaml
Replace the content of my-redis/templates/deployment.yaml with the following:

yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-redis
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-redis
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-redis
    spec:
      containers:
        - name: redis
          image: "{{ .Values.redis.image.repository }}:{{ .Values.redis.image.tag }}"
          imagePullPolicy: {{ .Values.redis.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.redis.port }}
          env:
            - name: REDIS_PASSWORD
              value: {{ .Values.redis.password | quote }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-redis
spec:
  selector:
    app: {{ .Release.Name }}-redis
  ports:
    - protocol: TCP
      port: 6379
      targetPort: {{ .Values.redis.port }}
Step 5: Install the Helm Chart
Now, you can install the Helm chart:

bash
Copy code
helm install my-redis ./my-redis
Step 6: Verify the Deployment
Check the status of the deployed Redis instances:

bash
Copy code
kubectl get pods,svc
Step 7: Cleanup (Optional)
If needed, you can uninstall the Helm release:

bash
Copy code
helm uninstall my-redis
This example Helm chart deploys two replicas of Redis. You can further customize the Helm chart based on your specific requirements, such as configuring persistent storage, using specific Redis configurations, etc.





