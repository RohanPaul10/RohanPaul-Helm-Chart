Helm Chart Structure
retail-app/
│── Chart.yaml
│── values.yaml
└── templates/
    │── mongodb-deployment.yaml
    │── mongo-service.yaml
    │── mongo-secret.yaml
    │── mongo-pvc.yaml
    │── userprofile-deployment.yaml
    │── userprofile-service.yaml
    │── user-secrets.yaml
    │── app-configmap.yaml
    │── ingress.yaml

Chart.yaml:-

apiVersion: v2
name: retail-app
description: A Helm chart for deploying Retail App (MongoDB + UserProfile Service)
type: application

version: 0.1.0
appVersion: "1.0.0"

maintainers:
  - name: Teja
    email: chagantyteja2502@gmail.com

values.yaml:- 

namespace: rohan
replicaCount: 2
image:
  repository: dockerimageyour   # replace with your image repo
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 3130

mongodb:
  image: mongo
  tag: latest
  username: admin
  password: admin123
  database: myDatabase
  storage: 1Gi

userprofile:
  sessionSecret: "1234"
  emailUser: "chagantyteja2502@gmail.com"
  emailPass: "yxoq bjuk rdnt alzp"

ingress:
  enabled: false
  className: nginx
  host: retail.local
  path: /
  pathType: Prefix

mongodb-deployment.yaml:-

templates/mongodb-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-mongodb
  namespace: {{ .Values.namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Release.Name }}-mongodb
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-mongodb
    spec:
      containers:
        - name: mongodb
          image: "{{ .Values.mongodb.image }}:{{ .Values.mongodb.tag }}"
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-mongo-secret
                  key: MONGO_USER
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-mongo-secret
                  key: MONGO_PASS
            - name: MONGO_INITDB_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: {{ .Release.Name }}-app-config
                  key: MONGODB_DATABASE
          volumeMounts:
            - name: mongo-storage
              mountPath: /data/db
      volumes:
        - name: mongo-storage
          persistentVolumeClaim:
            claimName: {{ .Release.Name }}-mongo-pvc

mongo-service.yaml:-

templates/mongo-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-mongodb
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: {{ .Release.Name }}-mongodb
  ports:
    - port: 27017
      targetPort: 27017
      protocol: TCP
  type: ClusterIP

mongo-secret.yaml:-

templates/mongo-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-mongo-secret
  namespace: {{ .Values.namespace }}
type: Opaque
stringData:
  MONGO_USER: "{{ .Values.mongodb.username }}"
  MONGO_PASS: "{{ .Values.mongodb.password }}"

mongo-pvc.yaml:-

templates/mongo-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ .Release.Name }}-mongo-pvc
  namespace: {{ .Values.namespace }}
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.mongodb.storage }}
  storageClassName: ""   # default storage class

userprofile-deployment.yaml:-

templates/userprofile-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-userprofile
  namespace: {{ .Values.namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-userprofile
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-userprofile
    spec:
      containers:
        - name: usernode-js
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          env:
            - name: MONGODB_URI
              value: "mongodb://$(MONGO_USER):$(MONGO_PASS)@{{ .Release.Name }}-mongodb:27017/$(MONGODB_DATABASE)?authSource=admin"
          envFrom:
            - secretRef:
                name: {{ .Release.Name }}-mongo-secret
            - secretRef:
                name: {{ .Release.Name }}-user-secrets
            - configMapRef:
                name: {{ .Release.Name }}-app-config

userprofile-service.yaml:-

templates/userprofile-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-userprofile-service
  namespace: {{ .Values.namespace }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}-userprofile
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
      protocol: TCP
      
user-secrets.yaml:-

templates/user-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-user-secrets
  namespace: {{ .Values.namespace }}
type: Opaque
stringData:
  SESSION_SECRET: "{{ .Values.userprofile.sessionSecret }}"
  EMAIL_USER: "{{ .Values.userprofile.emailUser }}"
  EMAIL_PASS: "{{ .Values.userprofile.emailPass }}"
  
app-configmap.yaml:-

templates/app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-app-config
  namespace: {{ .Values.namespace }}
data:
  PORT: "{{ .Values.service.port }}"
  MONGODB_DATABASE: "{{ .Values.mongodb.database }}"

ingress.yaml:-

templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
  namespace: {{ .Values.namespace }}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: {{ .Values.ingress.pathType }}
            backend:
              service:
                name: {{ .Release.Name }}-userprofile-service
                port:
                  number: {{ .Values.service.port }}
{{- end }}


kubectl get all -n rohan



Userprofile app (Deployment + Service + Secret + ConfigMap)

Optional Ingress
