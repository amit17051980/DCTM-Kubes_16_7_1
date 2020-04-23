# DCTM-Kubes_16_7_1
OpenText Documentum 16.7.1 K8S Cluster

# Testing Environment 
* **Operating System** : MacOS Catalina 10.15.3 (19D76)
* **RAM** : 16 GB
* **CPU** : 2.2 GHz 6-Core Intel Core i7
* **Docker (Optional)** : 19.03.5, build 633a0ea ([Use Minikube add-on for docker registry](https://minikube.sigs.k8s.io/docs/tasks/docker_registry/))
* **Kubernetes (Standalone)** : Minikube v1.6.1 on Darwin 10.15.3
* **Helm + Tiller**	: Version "2.16.1"

# Install Docker (Optional)
Please follow installation guide below to install docker for MacOS. The package manager **brew** could also be used. It is recommended to follow docker documentations for this test exercise.

[Docker for MacOS](https://docs.docker.com/docker-for-mac/install/)

# Install Minikube
Please follow the guide below to install Kubernetes (Minikube) on MacOS. This guide is not enterprise ready, hence seek additional assistance from Service Provide or independent consultants. This will certainly help explore the Kubernetes in the Developer’s modern machine.

[Before You Begin](https://kubernetes.io/docs/tasks/tools/install-minikube/#minikube-before-you-begin-1 )

[Installation Guide](https://kubernetes.io/docs/tasks/tools/install-minikube/#tab-with-md-1)

# Install Helm with Tiller
The Helm3 is not supported by OpenText officially yet. The OpenText guides are based on Helm2. The instructions provided by OpenText are associated with Client-Server Helm architecture and cannot be used against Helm3. The Tiller has been removed on Helm3, and doesn't need to be initialised on Kubernetes cluster. 

```
# Setting Helm2 Client
curl --url https://get.helm.sh/helm-v2.16.1-darwin-amd64.tar.gz --output helm.tar.gz
tar -xvf helm.tar.gz
export PATH=$PATH:$PWD/darwin-amd64

# Starting Kubernetes Standalone Cluster
minikube start

# Installaing Helm2 Server (Tiller)
helm init
```

# Prepare Documentum Images and HelmCharts
Contact your OpenText DSE to enable [Documentum Content Server Docker Image](https://mimage.opentext.com/support/ecm/secure/software/dell/documentum/documentumcontentserver/16.7.1/contentserver_16.7.1000_docker_centos.tar) download under OpenText Support Site.

FYI.. [Reference Only. Follow the next section for continuing the setup].

Example commands used before initiating this repo:
```
mkdir -p CS_16_7_1000
cd CS_16_7_1000
tar -xvf ../contentserver_16.7.1000_docker_centos.tar
# HelmCharts has been repackaged to address a minor issue. The same has been factored in this repo.
tar -xvf HelmChart.tar
tar -xvf Scripts.tar
tar -xvf cs-secrets-16.7.1000-0847.tgz
# Graylog Monitoring has not been experienced in this test!
# docker pull graylog/graylog:3.1.4
# The docker image will be loaded to minikube docker context
# docker load -i contentserver_centos_16.7.1000.0847.tar
```

## Upload Images (Postgres and Content Server) to local registry
**Note**: If you have installed [Docker for MacOS], the steps below can simply be used without mounting and changing the shell to minikube. Just use `eval $(minikube -p minikube docker-env)` command from MacOS shell. This will bring Minikube Docker context to MacOs Docker context. This makes all the docker commands executed inside the minikube without going to minikube shell.

The code below assumes that the **contentserver_centos_16.7.1000.0847.tar** has been extracted from **contentserver_16.7.1000_docker_centos.tar** as shown in the example above.
```
# Minikube Docker Registry Add-on
minikube addons enable registry
minikube addons list
```
|         ADDON NAME          | PROFILE  |    STATUS    |
|-----------------------------|----------|--------------|
| dashboard                   | minikube | enabled ✅   |
| default-storageclass        | minikube | enabled ✅   |
| registry                    | minikube | enabled ✅   |
| storage-provisioner         | minikube | enabled ✅   |
```
# Mounting Home Drive to Minikube Cluster
minikube mount $HOME:/host &
minikube ssh
```
```
                         _             _            
            _         _ ( )           ( )           
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __  
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)
```
```
docker load -i /host/Downloads/CS_16_7_1000/content-server-16.7.1000-0847.tgz
docker tag contentserver/centos/stateless/cs:16.7.1000.0847 localhost:5000/contentserver/centos/stateless/cs:16.7.1000.0847
docker rmi contentserver/centos/stateless/cs:16.7.1000.0847
docker push localhost:5000/contentserver/centos/stateless/cs:16.7.1000.0847

docker pull postgres:10.3
docker tag postgres:10.3 localhost:5000/postgres:10.3
docker rmi postgres:10.3
docker push localhost:5000/postgres:10.3

docker image ls|grep 5000
logout
```
```
localhost:5000/contentserver/centos/stateless/cs   16.7.1000.0847      e7a75d670afb        2 months ago        3.4GB
localhost:5000/cs/pg                               10.3                b5ed9a4ab65b        22 months ago       234MB
```

## Clone the Project and run the HelmCharts
Please remember to initialise the Tiller using 'helm init'. _**In this article we are not complicating the use of service accounts etc.**_

```
# Use cs-secrets/values.yaml to review the passwords
helm install --name my-secrets cs-secrets/
helm install --name my-db db/
helm install --name my-docbroker docbroker/
helm install --name my-cs content-server/
```

Review Kubernetes logs and dashboard. Review the Sample log file in the project. It will give you the indication of the timelines. Follow https://minikube.sigs.k8s.io/docs/overview/ to familiarise with Minikube and its add-on.

# Thanks and good luck!
_If you have any concerns or questions, please comment here. Will try to answer as appropriate.
I'll progress with the rest of instructions to make it complete at later stage. But please compare the charts from official sites to understand how I enabled complete stack on Minikube without using External storage or dependencies._

**Note**: If you want to run Documentum Administrator or REST Service or xCP Designer or xDA on your MacOS tomcat instance instead of Minikube cluster, please create another service that exposes Docker Statefulset with 1489 port (optionally 50000 of docbase /etc/services). This is currently giving little challenges which I am working on. 
But if you are using DCTM-REST as the entry point for Documentum projects, please simply use below commands. It is being assumed that the dctm-rest.war and da.war are available in current directory and has been modified to use the right dfc.properties.

```
helm install --name my-da stable/tomcat --set image.tomcat.repository=amit17051980/tomcat,image.tomcat.tag=8.5
POD=$(kubectl get pod -l app=tomcat -o jsonpath="{.items[0].metadata.name}");
kubectl cp da.war $POD:/usr/local/tomcat/webapps/
kubectl cp dctm-rest.war $POD:/usr/local/tomcat/webapps/
```

# Reference Documents 
* https://knowledge.opentext.com/knowledge/llisapi.dll/Open/77242433 
* https://minikube.sigs.k8s.io/docs/overview/
* https://rancher.com/docs/rancher/v2.x/en/installation/options/helm2/helm-init/
* https://developer.epages.com/blog/tech-stories/kubernetes-deployments-with-helm/


