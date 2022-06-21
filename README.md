# Tomcat Container On AKS by ACR
## References
Tomcat on Containers QuickStart<br>
https://github.com/Azure/tomcat-container-quickstart<br>
Authenticate with Azure Container Registry from Azure Kubernetes Service<br>
https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?tabs=azure-cli<br>
Tutorial: Deploy and use Azure Container Registry<br>
https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr?tabs=azure-cli<br>

## 1. Create a new AKS cluster with ACR integration
Step.1 set the name of your Azure Container Registry. It must be globally unique.<br>
```bash
MYACR=myACR0621
MYRG=neoResourceGroup
```
Step.2 Create a resource group.<br>
```bash
MYRG=neoResourceGroup
az group create -n $MYRG -l eastasia
```
Step.3 Create an Azure Container Registry.<br>
```bash
az acr create -n $MYACR -g $MYRG --sku basic
```
Step.4 Create an AKS Cluster with ACR integration.<br>
```bash
MYAKS=neoAKSCluster
az aks create -n $MYAKS -g $MYRG --generate-ssh-keys --attach-acr $MYACR
```
## 2. Working with ACR & AKS
Step.1 Import an image from docker hub into ACR.<br>
```bash
az acr import  -n $MYACR --source docker.io/library/nginx:latest --image nginx:v1
```
Step.2 Get the AKS clistercredentials.<br>
```bash
az aks get-credentials -g $MYRG -n $MYAKS
```
Step.3 Create a file called acr-nginx.yaml that containers the following. Replace the acr-name with your ACR.<br>
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx0-deployment
  labels:
    app: nginx0-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx0
  template:
    metadata:
      labels:
        app: nginx0
    spec:
      containers:
      - name: nginx
        image: <acr-name>.azurecr.io/nginx:v1
        ports:
        - containerPort: 80
```
Step.4 Deploy a nginx Pod in your AKS cluster.<br>
```bash
kubectl apply -f acr-nginx.yaml
```
Step.5 Check the deployment and you should have 2 running pods.<br>
```bash
kubectl get pods
```
## 3. Prepare tomac application(by Dockerfile) for AKS
Step1. Clone the GitHub repository on your laptop or local computer (with Docker Desktop) and navigate into the root of the repository.<br>
```bash
git clone https://github.com/Azure/tomcat-container-quickstart.git
cd tomcat-container-quickstart
```
Step.2. Check the Dockerfile of root of the repository, and the TOMACT_VERSION is 9.0.38. Build the docker image.<br>
```bash
docker build . -t tomcat
```
Step.3 Run the image on your laptop or local computer for confirm the tomact applicationi is work.<br>
```bash
docker run -p8080:8080 -d tomcat
docker ps
```
Step.4 Once the tomact application container is running, navigate to http://localhost:8080 in you browser. You should see the application come up.<br>
## 4. Push images to registry
Step.1 To use the ACR instance, you must first log in.<br>
```bash
az acr login --name $MYACR
```
Step.2 List of your current local images and find the tomcat - container image.<br>
```bash
docker images
```
Step.3 To get the login server address.<br>
```bash
az acr list --resource-group $MYRG --query "[].{acrLoginServer:loginServer}" --output table
```
Step.4 Tag your local tomcat image with the acrLoginServer address of the container register. Then verify the tags are applied.<br>
```bash
docker tag tomcat:latest myacr0621.azurecr.io/tomcat:v1
docker images
```
Step.5 Push the tomcat image to your ACR instance. It may take a few minutes to complete the image push to ACR.<br>
```bash
docker push myacr0621.azurecr.io/tomcat:v1
```
## 5. List images in Azure Container Registry
Step.1 List images on your ACR. See the tages for a specific image, e.g., Tomcat.<br>
```bash
az acr repository list --name $MYACR --output table
az acr repository show-tags --name $MYACR --repository tomact --output table
```
## 6. Deploy a tomcat application on your AKS
Step.1 Create a file called acr-tomact-all-in-one.yaml that contains the following.<br>
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat0-deployment
  labels:
    app: tomcat0-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tomcat0
  template:
    metadata:
      labels:
        app: tomcat0
    spec:
      containers:
      - name: tomcat
        image: <acr-name>.azurecr.io/tomcat:v1
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tomcat-front
spec:
  type: LoadBalancer
  ports:
  - port: 8080
  selector:
    app: tomcat0
```
Step.2 Deploy a tomcat application in your AKS cluster. Check the deployment, you should have 2 pods and 1 service.<br>
```bash
kubectl apply -f acr-tomcat-all-in-one.yaml
kubectl get pods,svc
```
Step.3 Once the containers are running, navidate to http://<EXTERNAL_IP:8080> in your browser. You should see the tomcat application come up.
