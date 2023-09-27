# helmchart
# Deploy application on EKS using helm chart

### Importatnt Links
1. [Building first Helm Chart with Spring Boot Microservices](https://jhooq.com/building-first-helm-chart-with-spring-boot/)
1. [YouTube Video](https://www.youtube.com/watch?v=2dqQcou_MCU&list=PL7iMyoQPMtANm_35XWjkNzDCcsw9vy01b&index=2) ``` Helm Chart Demo - How to create your first Helm Chart ```

1. How to Install Helm Chart?
Installing Helm is fairly easy and there are various package manager available for you -

### Binary

Get the Binary - [Download Binary](https://github.com/helm/helm/releases)
Unpack it using - tar -zxvf helm-vxxx-xxxx-xxxx.tar.gz
Move it - mv linux-amd64/helm /usr/local/bin/helm

### Using Script

There is one more way to install latest Helm Version using script. Refer to the following terminal command for installing latest version of Helm -
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh

```

### Verify Helm Chart Installation

To verify the installation use the following command

```
which helm

```
### create First Helm Chart

```
kubectl get all

Output -
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.233.0.1   <none>        443/TCP   27d

helm create springboot
output -
```
### Helm Chart Structure

Before we deep dive into the nitty gritty of Helm Chart, let's go through the Helm Chart Skeleton. 
Run the following command to see the tree structure of our Springboot Helm Chart -

![image](https://github.com/anand40090/helmchart/assets/32446706/00661d29-9d5e-43d9-8b7b-67d6ffe614ec)

### Chart.yaml 
This file contains all the metadata about our Helm Chart for example -
![image](https://github.com/anand40090/helmchart/assets/32446706/97ae50bc-d399-4e01-8d5c-1b6c05f5935c)

You do not need to define each configuration here but you must need to define -
- apiVersion
- name
- version
- Other configurations are optional

### values.yaml

As the name suggests we have do something with the values and yes you are right about it. 
This configuration file holds values for the configuration.
```
replicaCount: 1
image:
  repository: rahulwagh17/kubernetes:jhooq-k8s-springboot #updated url
  pullPolicy: IfNotPresent
  tag: ""
imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""
serviceAccount:
  create: true
  annotations: {}
  name: ""
podAnnotations: {}
podSecurityContext: {}

securityContext: {}
service:
  type: ClusterIP
  port: 8080  #updated port
ingress:
  enabled: false
  annotations: {}
  hosts:
    - host: chart-example.local
      paths: []
  tls: []
resources: {}

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity: {}

```
It looks quite enormous but trust me you do not need to write or remember every configuration by heart. Helm Create command generates the bare minimum values.yaml for you.

Since in this example we are going to try our Helm Chart for spring boot application, lets go through the configuration which we need to modify for deploying our spring boot application.

repository : image:repository: rahulwagh17/kubernetes:jhooq-k8s-springboot
port: 8080
So in the whole values.yaml you need to update the configuration at two places repository and port

Note :You should have a docker image of your spring boot application uploaded into the docker hub

Alright now we are ready with our Chart.yaml and values.yaml, lets jump to next configuration yamls

deployment.yaml
The next configuration we need to update is deployment.yaml and as the name suggest it is used for deployment purpose. So lets see how does it looks like

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "springboot.fullname" . }}
  labels:
    {{- include "springboot.labels" . | nindent 4 }}
spec:
{{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
{{- end }}
  selector:
    matchLabels:
      {{- include "springboot.selectorLabels" . | nindent 6 }}
  template:
    metadata:
    {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
    {{- end }}
      labels:
        {{- include "springboot.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "springboot.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}"   #update here
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080     #update here
              protocol: TCP           #comment it
              #livenessProbe:           #comment it
              #httpGet:           #comment it
              #path: /           #comment it
              #port: http           #comment it
              #readinessProbe:           #comment it
              #httpGet:           #comment it
              #path: /           #comment it
              #port: http           #comment it
          resources:
            {{- toYaml .Values.resources | nindent 12  }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}

```
Do not get scared we do not need to update each and every configuration. But we need to update the containerPort to 8080

containerPort : 8080
Because we need to deploy our spring boot application at port 8080.

service.yaml
The service.yaml is basically used for exposing our kubernetes springboot deployment as service. The good thing over here we do not need to update any configuration here.

```
apiVersion: v1
kind: Service
metadata:
  name: {{ include "springboot.fullname" . }}
  labels:
    {{- include "springboot.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "springboot.selectorLabels" . | nindent 4 }}

```
### Run $ helm template springboot
Now we are into the step no 4 and we are pretty much ready with our first Helm Chart for our spring boot application.

But wait - Wouldn't it be nice if you could see the service.yaml, deployment.yaml with its actual values before running the helm install command?

Yes you can do that with the helm template command -

```
helm template springboot
```
(You can not run the above command from inside the springboot directory, so you should get out from the springboot directory and then execute the command)

After running the above command it should return you will service.yaml, deployment.yaml and test-connection.yaml with actual values

```
---
# Source: springboot/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: RELEASE-NAME-springboot
  labels:
    helm.sh/chart: springboot-0.1.0
    app.kubernetes.io/name: springboot
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
---
# Source: springboot/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: RELEASE-NAME-springboot
  labels:
    helm.sh/chart: springboot-0.1.0
    app.kubernetes.io/name: springboot
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: springboot
    app.kubernetes.io/instance: RELEASE-NAME
---
# Source: springboot/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: RELEASE-NAME-springboot
  labels:
    helm.sh/chart: springboot-0.1.0
    app.kubernetes.io/name: springboot
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: springboot
      app.kubernetes.io/instance: RELEASE-NAME
  template:
    metadata:
      labels:
        app.kubernetes.io/name: springboot
        app.kubernetes.io/instance: RELEASE-NAME
    spec:
      serviceAccountName: RELEASE-NAME-springboot
      securityContext:
        {}
      containers:
        - name: springboot
          securityContext:
            {}
          image: "rahulwagh17/kubernetes:jhooq-k8s-springboot"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          resources:
            {}
---
# Source: springboot/templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "RELEASE-NAME-springboot-test-connection"
  labels:
    helm.sh/chart: springboot-0.1.0
    app.kubernetes.io/name: springboot
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['RELEASE-NAME-springboot:8080']
  restartPolicy: Never

```

I love this command because it locally renders the templates by replacing all the placeholders with its actual values.

There is one more sanitary command lint provided by helm which you could run to identify possible issues forehand.

