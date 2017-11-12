# Kubernetes in Azure
This repo contains my Kubernetes demo in Azure.

# Table of Contents
- [Azure Container Instance demo](#azure-container-instance-demo)
    - [Create resource group](#create-resource-group)
    - [Run SQL on Linux container](#run-sql-on-linux-container)
    - [Connect to SQL](#connect-to-sql)
    - [Delete container](#delete-container)
- [Building custom ACS cluster](#building-custom-acs-cluster)
    - [Download ACS engine](#download-acs-engine)
    - [Build cluster and copy kubectl configuratio file](#build-cluster-and-copy-kubectl-configuratio-file)
        - [Mixed cluster with standard ACS networking](#mixed-cluster-with-standard-acs-networking)
        - [Cluster with Azure Networking CNI](#cluster-with-azure-networking-cni)
        - [Create VM for testing](#create-vm-for-testing)
        - [Access GUI](#access-gui)
- [Using stateless app farms in mixed environment](#using-stateless-app-farms-in-mixed-environment)
    - [Deploy multiple pods with Deployment](#deploy-multiple-pods-with-deployment)
    - [Create service to balance traffic internally](#create-service-to-balance-traffic-internally)
    - [Create externally accesable service with Azure LB](#create-externally-accesable-service-with-azure-lb)
    - [Upgrade](#upgrade)
    - [Deploy IIS on Windows pool](#deploy-iis-on-windows-pool)
    - [Test Linux to Windows communication](#test-linux-to-windows-communication)
        - [Connect via internal service endpoint](#connect-via-internal-service-endpoint)
    - [Clean up](#clean-up)
- [Stateful applications and StatefulSet with Persistent Volume](#stateful-applications-and-statefulset-with-persistent-volume)
    - [Check storage class and create Persistent Volume](#check-storage-class-and-create-persistent-volume)
    - [Create StatefulSet with Volume template for Postgresql](#create-statefulset-with-volume-template-for-postgresql)
    - [Connect to PostgreSQL](#connect-to-postgresql)
    - [Destroy Pod and make sure StatefulSet recovers and data are still there](#destroy-pod-and-make-sure-statefulset-recovers-and-data-are-still-there)
    - [Continue in Azure](#continue-in-azure)
    - [Clean up](#clean-up)
- [Helm](#helm)
    - [Install](#install)
    - [Run Wordpress](#run-wordpress)
    - [Clean up](#clean-up)
- [CI/CD with Jenkins and Helm](#cicd-with-jenkins-and-helm)
    - [Install Jenkins to cluster via Helm](#install-jenkins-to-cluster-via-helm)
    - [Configure Jenkins and its pipeline](#configure-jenkins-and-its-pipeline)
    - [Run "build"](#run-build)
- [Monitoring](#monitoring)
    - [Prepare Log Analytics / OMS](#prepare-log-analytics-oms)
    - [Deploy agent](#deploy-agent)
    - [Generate message in app](#generate-message-in-app)
    - [Log Analytics](#log-analytics)
    - [Clean up](#clean-up)

# Azure Container Instance demo
Before we start with Kubernetes let see Azure Container Instances. This is top level resource in Azure so you don't have to create (and pay for) any VM, just create container directly and pay by second. In this demo we will deploy Microsoft SQL Server in Linux container.

## Create resource group
```
az group create -n aci-group -l westeurope
```

## Run SQL on Linux container
```
az container create -n mssql -g aci-group --cpu 2 --memory 4 --ip-address public --port 1433 -l eastus --image microsoft/mssql-server-linux -e 'ACCEPT_EULA=Y' 'SA_PASSWORD=my(!)Password' 
export sqlip=$(az container show -n mssql -g aci-group --query ipAddress.ip -o tsv)
watch az container logs -n mssql -g aci-group
```

## Connect to SQL
```
sqlcmd -S $sqlip -U sa -P 'my(!)Password' -Q 'select name from sys.databases'
```

## Delete container
```
az container delete -n mssql -g aci-group -y
az group delete -n aci-group -y
```

# Building custom ACS cluster
Azure Container Instance is deployment, upgrade and scaling tool to get open source orchestrators up and running in Azure quickly. ACS as native embedded Azure offering (in GUI, CLI etc.) is production-grade version of open source acs-engine (deployment tool). In order to get latest features we will download acs-engine so we are able to tweek some of its parameters that are not yet available in version embedded in ACS.

## Download ACS engine
```
wget https://github.com/Azure/acs-engine/releases/download/v0.8.0/acs-engine-v0.8.0-linux-amd64.zip
unzip acs-engine-v0.8.0-linux-amd64.zip
mv acs-engine-v0.8.0-linux-amd64/acs-engine .
```

## Build cluster and copy kubectl configuration file
We will build multiple clusters to show some additional options, but majority of this demo runs on first one.

### Mixed cluster with standard ACS networking
Our first cluster will be hybrid Linux and Windows agents, with RBAC enabled and with support for Azure Managed Disks as persistent volumes in Kubernetes. Basic networking will be used with integration to Azure Load Balancer (for Kubernetes LodaBalancer Service).

```
./acs-engine generate myKubeACS.json
cd _output/myKubeACS/
az group create -n mykubeacs -l westeurope
az group deployment create --template-file azuredeploy.json --parameters @azuredeploy.parameters.json -g mykubeacs
scp azureuser@mykubeacs.westeurope.cloudapp.azure.com:.kube/config ~/.kube/config
```

### Cluster with Azure Networking CNI
In this cluster we will use Azure Networkin CNI plugin. This allows pods to use directly IP addresses from Azure VNET and allows for Azure Networking features to be used with pods - for example Network Security Groups or direct communication between pods in cluster and VMs in the same VNET.

```
./acs-engine generate myKubeAzureNet.json
cd _output/myKubeAzureNet/
az group create -n mykubeazurenet -l westeurope
az group deployment create --template-file azuredeploy.json --parameters @azuredeploy.parameters.json -g mykubeazurenet
scp azureuser@mykubeazurenet.westeurope.cloudapp.azure.com:.kube/config ~/.kube/config-azurenet
```

### Create VM for testing
```
export vnet=$(az network vnet list -g mykubeacs --query [].name -o tsv)

az vm create -n myvm -g mykubeacs --admin-username azureuser --ssh-key-value "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDFhm1FUhzt/9roX7SmT/dI+vkpyQVZp3Oo5HC23YkUVtpmTdHje5oBV0LMLBB1Q5oSNMCWiJpdfD4VxURC31yet4mQxX2DFYz8oEUh0Vpv+9YWwkEhyDy4AVmVKVoISo5rAsl3JLbcOkSqSO8FaEfO5KIIeJXB6yGI3UQOoL1owMR9STEnI2TGPZzvk/BdRE73gJxqqY0joyPSWOMAQ75Xr9ddWHul+v//hKjibFuQF9AFzaEwNbW5HxDsQj8gvdG/5d6mt66SfaY+UWkKldM4vRiZ1w11WlyxRJn5yZNTeOxIYU4WLrDtvlBklCMgB7oF0QfiqahauOEo6m5Di2Ex" --image UbuntuLTS --nsg "" --vnet-name $vnet --subnet k8s-subnet --public-ip-address-dns-name mykubeextvm --size Basic_A0

ssh azureuser@mykubeextvm.westeurope.cloudapp.azure.com
```

### Access GUI
```
kubectl proxy
```

# Using stateless app farms in mixed environment
This set of demos focus on stateless applications like APIs or web frontend. We will deploy application, balance it internally and externally, do rolling upgrade, deploy both Linux and Windows containers and make sure they can access each other.

## Deploy multiple pods with Deployment
```
kubectl create -f deploymentWeb1.yaml
kubectl get deployments -w
kubectl get pods -o wide
```

## Create service to balance traffic internally
```
kubectl create -f podUbuntu.yaml
kubectl create -f serviceWeb.yaml
kubectl get services
kubectl exec ubuntu -- curl -s myweb-service
```

## Create externally accesable service with Azure LB
```
kubectl create -f serviceWebExt.yaml
```

## Upgrade
```
kubectl apply -f deploymentWeb2.yaml
```

## Deploy IIS on Windows pool
```
kubectl create -f IIS.yaml
kubectl get service
```

## Test Linux to Windows communication
### Connect via internal service endpoint
```
kubectl exec ubuntu -- curl -s myiis-service-ext
```

## Clean up
```
kubectl delete -f serviceWebExt.yaml
kubectl delete -f serviceWeb.yaml
kubectl delete -f podUbuntu.yaml
kubectl delete -f deploymentWeb1.yaml
kubectl delete -f deploymentWeb2.yaml
kubectl delete -f IIS.yaml
```

# Stateful applications and StatefulSet with Persistent Volume
Deployments in Kubernetes are great for stateless applications, but statful apps, eg. databases. might require different handling. For example we want to use persistent storage and make sure, that when pod fails, new is created mapped to the same persistent volume (so data are persisted). Also in stateful applications we want to keep identifiers like network (IP address, DNS) when pod fails and needs to be rescheduled. Also when multiple replicas are used we need to start them one by one, because aften first instance is going to be master and others slave (so we need to wait for first one to come up first). If we need to scale down, we want to do this from last instance (not to scale down by killing first instance which is likely to be master). More details can be found in documentation.

In this demo we will deploy single instance of PostgreSQL.

## Check storage class and create Persistent Volume
```
kubectl get storageclasses
kubectl create -f persistentVolumeClaim.yaml
kubectl get pvc
kubectl get pv
```

Make sure volume is visible in Azure. 

Clean up.
```
kubectl delete -f persistentVolumeClaim.yaml
```

## Create StatefulSet with Volume template for Postgresql
```
kubectl create -f statefulSetPVC.yaml
kubectl get pvc -w
kubectl get statefulset -w
kubectl get pods -w
kubectl logs postgresql-0
```

## Connect to PostgreSQL
```
kubectl exec -ti postgresql-0 -- psql -Upostgres
CREATE TABLE mytable (
    name        varchar(50)
);
INSERT INTO mytable(name) VALUES ('Azure User');
SELECT * FROM mytable;
\q
```

## Destroy Pod and make sure StatefulSet recovers and data are still there
```
kubectl delete pod postgresql-0
kubectl exec -ti postgresql-0 -- psql -Upostgres -c 'SELECT * FROM mytable;'
```

## Continue in Azure
Destroy statefulset and pvc, keep pv
```
kubectl delete -f statefulSetPVC.yaml
```

Go to GUI and map IaaS Volume to VM, then mount it and show content.
```
ssh azureuser@mykubeextvm.westeurope.cloudapp.azure.com
ls /dev/sd*
sudo mkdir /data
sudo mount /dev/sdc /data
sudo ls -lh /data/pgdata/
sudo umount /dev/sdc
```

Detach in GUI

## Clean up
```
kubectl delete pvc postgresql-volume-claim-postgresql-0
```

## Custom registry
### Create Azure Container Registry
```
az group create -n mykuberegistry -l westeurope
az acr create -g mykuberegistry -n mycontainers --sku Managed_Standard --admin-enabled true
az acr credential show -n mycontainers -g mykuberegistry
export acrpass=$(az acr credential show -n mycontainers -g mykuberegistry --query [passwords][0][0].value -o tsv)
```

### Push images to registry
```
docker.exe images
docker.exe tag tkubica/web:1 mycontainers.azurecr.io/web:1
docker.exe tag tkubica/web:2 mycontainers.azurecr.io/web:2
docker.exe tag tkubica/web:1 mycontainers.azurecr.io/private/web:1
docker.exe login -u mycontainers -p $acrpass mycontainers.azurecr.io
docker.exe push mycontainers.azurecr.io/web:1
docker.exe push mycontainers.azurecr.io/web:2
docker.exe push mycontainers.azurecr.io/private/web:1
az acr repository list -n mycontainers -o table
```

### Run Kubernetes Pod from Azure Container Registry
```
kubectl create -f podACR.yaml
```

## Clean up
```
kubectl delete -f clusterRoleBindingUser1.yaml
az group delete -n mykuberegistry -y --no-wait
docker.exe rmi mycontainers.azurecr.io/web:1
docker.exe rmi mycontainers.azurecr.io/web:2
docker.exe rmi mycontainers.azurecr.io/private/web:1

```

# Helm
Helm is package manager for Kubernetes. It allows to put together all resources needed for application to run - deployments, services, statefulsets, variables.

## Install
```
cd ./helm
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.6.1-linux-amd64.tar.gz
tar -zxvf helm-v2.6.1-linux-amd64.tar.gz
sudo cp linux-amd64/helm /usr/local/bin
```

## Run Wordpress
```
helm 
cd ./helm
helm install --name myblog -f values.yaml .
```

## Clean up
```
helm delete myblog --purge
```

# CI/CD with Jenkins and Helm
In this demo we will see Jenkins deployed into Kubernetes via Helm and have Jenkins Agents spin up automatically as Pods.

CURRENT ISSUE: at the moment NodeSelector for agent does not seem to be delivered to Kubernetes cluster correctly. Since our cluster is hybrid (Linux and Windows) in order to work around it now we need to turn of Windows nodes.

## Install Jenkins to cluster via Helm
```
helm install --name jenkins stable/jenkins -f jenkins-values.yaml
```

## Configure Jenkins and its pipeline
Use this as pipeline definition
```
podTemplate(label: 'mypod') {
    node('mypod') {
        stage('Do something nice') {
            sh 'echo something nice'
        }
    }
}

```

## Run "build"
Build project in Jenkins and watch containers to spin up and down.
```
kubectl get pods -o wide -w
```

# Monitoring
## Prepare Log Analytics / OMS
Create Log Analytics account and gather workspace ID and key.
Create Container Monitoring Solution.

## Deploy agent
Modify daemonSetOMS.yaml with your workspace ID and key.

```
kubectl create -f daemonSetOMS.yaml
```

## Generate message in app
```
kubectl create -f podUbuntu.yaml
kubectl exec -ti ubuntu -- logger My app has just logged something

```
## Log Analytics
Container performance example
```
Perf
 | where ObjectName == "Container" and CounterName == "Disk Reads MB"
 | summarize sum(CounterValue) by InstanceName, bin(TimeGenerated, 5m)
 | render timechart 
```

## Clean up
```
kubectl delete -f podUbuntu.yaml
kubectl delete -f daemonSetOMS.yaml
```
