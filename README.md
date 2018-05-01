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

Update the Ubuntu packages and install Docker engine, node.js and the node package manager in a single step by typing the following in a single line command. When asked if you would like to proceed, respond by typing “y” and pressing enter.
 sudo apt-get update && sudo apt install docker-ce nodejs npm
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

```markdown
az ad sp create-for-rbac --role='Contributor' --scope="/subscriptions/60e79550-d86a-4c92-a4e1-c7faa8c6ae74" --name="MyDemos-sp-sol"
```
![image of sp](/images/service_principal.png)

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

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/davisanc/lab.aks.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
