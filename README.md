# clients-api


                                                     

                                                  Step-by-Step Guide to Deploy Clients API with Kubernetes and CI/CD
                                       ==========================================================================================


This detailed commands to set up a Dockerfile, Kubernetes deployment for a Clients API with MongoDB, Load Balancer, Encrypt TLS, and a GitHub Actions CI/CD pipeline.


Prerequisites:
---------------
Before starting, ensure you have:

1. A DockerFile & Kubernetes cluster on AKS

2. kubectl configured (kubectl get nodes should work)

3. Helm installed (helm version should work)

4. GitHub repository for the Clients API code

5.Docker installed (docker --version)

6. A domain (we'll use clients.api.deltacapita.com)


Step 1: Set Up and create the Dockerfile & Kubernetes Cluster & Tools
----------------------------------------------------------------------

1.1 Create the Dockerfile:     [Clients API is a Node.js application]

---

FROM node:16-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]

---


1.2 Build & Test Locally:

docker build -t clients-api:latest .

docker run -p 3000:3000 clients-api

   
Verify at:

http://YOUR IP:3000



1.3 Install kubectl:

sudo apt-get update

sudo apt-get install -y kubectl

kubectl version --client



1.4 Install Helm:

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version


1.5 Connect to Kubernetes EKS Cluster:

aws eks --region <region> update-kubeconfig --name <cluster-name>  



Step 2: Install Ingress-Nginx (Load Balancer)
-------------------------------------------------

2.1 Install ingress-nginx:

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer


For Verifying Load Balancer IP:

kubectl get svc -n ingress-nginx


* After thet Wait for EXTERNAL-IP to be assigned 


Configure DNS:

Point "clients.api.deltacapita.com" to this IP.


Step 3: Install Cert-Manager (TLS Certificates)
--------------------------------------------------

3.1 Install cert-manager:


helm repo add jetstack https://charts.jetstack.io

helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.0 \
  --set installCRDs=true



3.2 Verify Installation:

kubectl get pods -n cert-manager
 

[* Now we can see 3 pods are showing as name as cert-manager, cert-manager-cainjector, and cert-manager-webhook}



Step 4: Deploy MongoDB
------------------------

4.1 Create Kubernetes Manifests (Create a k8s directory and add)

01-namespace.yaml:

---
apiVersion: v1
kind: Namespace
metadata:
  name: clients-api
---

02-mongodb/deployment.yaml:

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: clients-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secrets
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secrets
              key: password

---

02-mongodb/service.yaml:

---
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: clients-api
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017

---

02-mongodb/secret.yaml:

---
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secrets
  namespace: clients-api
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded "admin"
  password: c2VjcmV0cGFzc3dvcmQ=  # base64 encoded "secretpassword"

---


4.2 Now Apply MongoDB Deployment by these following commands:

kubectl apply -f k8s/01-namespace.yaml

kubectl apply -f k8s/02-mongodb/



4.3 Verify MongoDB:

kubectl get pods -n clients-api

kubectl logs -n clients-api deploy/mongodb



Step 5: Deploy Clients API
-------------------------------

5.1 Create Deployment Files(03-clients-api/deployment.yaml):

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clients-api
  namespace: clients-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: clients-api
  template:
    metadata:
      labels:
        app: clients-api
    spec:
      containers:
      - name: clients-api
        image: ghcr.io/your-username/clients-api:latest
        ports:
        - containerPort: 3000
        env:
        - name: MONGODB_URI
          value: "mongodb://mongodb:27017/clients"
---


5.2 Create Service Files(03-clients-api/service.yaml):

---
apiVersion: v1
kind: Service
metadata:
  name: clients-api
  namespace: clients-api
spec:
  selector:
    app: clients-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
---


5.2 Apply Clients API :

kubectl apply -f k8s/03-clients-api/



Step 6: Set Up Ingress & TLS
-------------------------------

6.1 Create file (04-ingress/issuer.yaml):


---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
---

6.2 Create file (04-ingress/ingress.yaml):

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: clients-api-ingress
  namespace: clients-api
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - clients.api.deltacapita.com
    secretName: clients-api-tls
  rules:
  - host: clients.api.deltacapita.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: clients-api
            port:
              number: 80
---


6.3 Apply Ingress & TLS

kubectl apply -f k8s/04-ingress/


6.4 Verify the Certificate

kubectl get certificate -n clients-api




Step 7: Set Up GitHub Actions CI/CD
------------------------------------------

7.1 Create file (.github/workflows/ci-cd.yaml):

---
name: Clients API CI/CD

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2

    - name: Login to GitHub Container Registry
      uses: docker/login-action@v1
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and Push Docker Image
      run: |
        docker build -t ghcr.io/your-username/clients-api:latest .
        docker push ghcr.io/your-username/clients-api:latest

    - name: Deploy to Kubernetes
      uses: azure/k8s-deploy@v1
      with:
        namespace: clients-api
        manifests: k8s/
        images: ghcr.io/your-username/clients-api:latest
---


* Then Push everyting to the GitHub Repositiry


Step 8: Verify Deployment
---------------------------


8.1 Check all of thr Services working perfectly or not:

kubectl get pods -n clients-api

kubectl get all -n clients-api

kubectl get ingress -n clients-api



8.2 Now access the API:

curl https://clients.api.deltacapita.com
  

 * Then you can see it Should return your API response



Final Architecture
======================

* Dockerfile → Containerized App
* GitHub Actions CI/CD → Builds & Pushes Image
* Kubernetes → Deploys Image
* MongoDB (Stateful) → Data Storage
* Ingress-Nginx → Load Balancer
* Let’s Encrypt TLS (HTTPS)
* Automatic Deployments on git push


Troubleshooting:
-------------------

Docker Build Fails:

First Check the Dockerfile syntax and then ensure the package.json exists or not


No External IP? 

Then Wait for 5 mins or check cloud provider LoadBalancer.


Certificate not issued:

kubectl describe certificate -n clients-api

kubectl describe order -n clients-api


API not working? Check logs:

kubectl logs -n clients-api deploy/clients-api



* Now the clients API is fully containerized, deployed, and automated with CI/CD, TLS, and Load Balancing.
