## Azure Container Service (AKS) - Running a Damn Vulnerable Web Application on AKS

This is a follow up from a previous post , where you can set up a Damn Vulnerable Web Application (DVWA from now on) to test [Azure WAF to protect your Web Application](https://blogs.msdn.microsoft.com/ukhybridcloud/2018/03/20/azure-waf-to-protect-your-web-application/)

Now, I would like to run the DVWA within a container using the Azure Container Service (AKS), where Azure manages your hosted Kubernetes environment

Detail documentation about Azure AKS is [here](https://docs.microsoft.com/en-us/azure/aks/)

### Step1: Work on your Build-VM

This assumes you have a machine already provisioned on Azure. In my case I am working on an ubuntu VM

On your ubuntu Build-VM, create a dvwa folder and pull the docker image
https://hub.docker.com/r/vulnerables/web-dvwa/

Update the Ubuntu packages and install curl and support for repositories over HTTPS in a single step by typing the following in a single line command. When asked if you would like to proceed, respond by typing “y” and pressing enter.

```markdown
sudo apt-get update && sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

Add Docker’s official GPG key by typing the following in a single line command

```markdown
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
 ```
 Add Docker’s stable repository to Ubuntu packages list by typing the following in a single line command
 ```markdown
 sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Update the Ubuntu packages and install Docker engine in a single step by typing the following in a single line command. When asked if you would like to proceed, respond by typing “y” and pressing enter

```markdown
 sudo apt-get update && sudo apt install docker-ce 
 ```
 
 Now, upgrade the Ubuntu packages to the latest version by typing the following in a single line command. When asked if you would like to proceed, respond by typing “y” and pressing enter

```markdown
 sudo apt-get upgrade
 ```
 When the command has completed, check the Docker version installed by executing this command

```markdown
Docker version
```

Download the dvwa image from docker hub
```markdown
docker pull vulnerables/web-dvwa
```

Run docker image on build-VM. If you want your host port and container port on 80,

```markdown
docker run --rm -it -p 80:80 vulnerables/web-dvwa
```

You may want to change the host port

Write a dockerfile to expose port 8080
```markdown
EXPOSE 8080
```

Create your docker image with the dockerfile

Check your docker images

### Step2: Create a Service Principal

![image of sp](/images/sp2.jpg)

### Step3: Create a RG and AKS Cluster

```markdown
az login
```
Create your Resource Group:
```markdown
az group create --name MyDemos-AKS --location westeurope
```

Create your cluster (by default it will use 3 nodes)

```markdown
az aks create --name MyDemos-AKS -g MyDemos-RG --generate-ssh-keys --kubernetes-version 1.9.6
```

Install kubectl:

```markdown
az aks install-cli
```

Instruct kubectl to connect to your AKS cluster:

```markdown
az aks get-credentials --name MyDemos-AKS -g MyDemos-RG
```

Check you have 3 nodes running:

```markdown
Kubectl get nodes
```

Check you are in the current context:
```markdown
kubectl config current-context
```

Open Kubernetes dashboard:

```markdown
Az aks browse -g MyDemos-RG --name MyDemos-AKS
```

### Step4: Create an ACR

In the Azure Portal, navigate to the ACR you created in Before the hands-on lab.
Select Access keys under Settings on the left-hand menu.

![image of keys](/images/access_keys.png)

### Step5: Prepare your docker image and push it to ACR

On your build VM, login to your ACR with the ACR access keys

```markdown
docker login [LOGINSERVER] –u [USERNAME] –p [PASSWORD]
```

Run the following commands to properly tag your images to match your ACR account name. 

```markdown
docker tag vulnerables/web-dvwa [LOGINSERVER]/web-dvwa
```

![image of docker tag](/images/docker_tag.png)

push docker image to ACR

```markdown
docker push [LOGINSERVER]/web-dvwa
```

![image of docker push](/images/docker_push.png)

In the Azure Portal, navigate to your ACR account, and select Repositories under Services on the left-hand menu. 

### Step6: Deploy the DVWA to your Kubernetes cluster

Assuming you have az installed on your machine. I like to run these commands from my PC, using Virtual Studio Code

```markdown
Az --version
```

check the installation of the Kubernetes CLI (kubectl) by running the following command

```markdown
Kubectl version
```

Login into your Azure subscription

```markdown
Az login
```

Verify that you are connected to the correct subscription with the following command to show your default subscription:

```markdown
Az account show
```

If you are not connected to the correct subscription, list your subscriptions and then set the subscription by its id with the following commands

```markdown
 az account list
 az account set --subscription {id}
 ```
 
 Test that the configuration is correct by running a simple kubectl command to produce a list of nodes: 

```markdown
kubectl get nodes
```

![image of get nodes](/images/get_nodes.png)

Tunnel into your AKS cluster: Create an SSH tunnel linking a local port (8001) on your machine to port 80 on the management node of the cluster. Execute the command below replacing the values as follows:

```markdown
az aks browse --name MyDemos-AKS --resource-group MyDemos-RG 
```
![image of aks browse](/images/aks_browse.png)

K8s dashboard will be open in our browser

![image of dashboard](/images/dashboard.png)

### Step7: Create the DVWA application from the container

Now we deploy the DVWA web service using the K8s dashboard

* From the Kubernetes dashboard, select Create in the top right corner
* From the Resource creation view, select Create an App
* Give the app a name, followed by the container image (with your ACR Login Server), 1 pod, Service Internal, Port and Target Port equals to 80
* CPU to 0.125 and Memory to 128

![image of create app](/images/create_app.png)

![image of create app2](/images/create_app2.png)

Now navigate to your deployments to see the DVWA container is successfully deployed

![image of deployments](/images/deployments.png)

### Step 8: Create a public Load Balancer to access the DVWA from the outside

Any external access to Kubernetes pods requires a load balancer. We will define the DVWA service with the type LoadBalancer in the YAML description, so you can access the web application using the public IP.  When you change the type of the service to LoadBalancer, the AKS will create a public-facing load balancer with a public IP address.

* From the navigation menu, select Services under Discovery and Load Balancing. From the view’s Services list, select the DVWA service. 
* Select Edit 
* From the Edit a Service dialog, scroll down to the type setting and change it from ClusterIP to LoadBalancer. Select Update to save the changes

![image of loadbalancer](/images/LoadBalancer.png)

Once the service is update you can see the public ip address assigned to the Azure Load Balancer

![image of services](/images/services.png)

Now check you can access the application with your browser

![image of browse](/images/browse.png)

The Load Balancer is created as part of the AKS infrastructure in Azure. When we created the AKS cluster, there are 2 RG created (this is done by design): one containing your K8S master controller and another one containing the rest of the infrastructure needed (nodes, load balancers, NSG, Route tables, NIC, Disks, etc)

![image of k8sinfra](/images/k8infra.jpg)

And we can see there is a Fronted IP configuration with the Public IP address to access the DVWA 

![image of publicip](/images/pip.jpg)

