---
layout: post
title:  "Terraforming an Azure Kubernetes Service cluster"
date:   2021-09-14
categories: jekyll update
---

## Background


[Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/) is simply a hosted Kubernetes service with extra snacks. [Terraform](https://www.terraform.io/intro/index.html) is an Infrastructure As Code (IaC) tool. In this post I'll provision an AKS cluster using Terraform.

## Setup

This post assumes that you have 
- [An active Azure account](https://azure.microsoft.com/en-us/free/)
- [Installed the Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- [Installed `kubectl` ](https://kubernetes.io/docs/tasks/tools/)
- [Installed the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)



## Creating a Terraform configuration

Let's create a directory for our terraform configuration 
{% highlight bash %}
mkdir ~/Documents/test-terraform
cd ~/Documents/test-terraform
{% endhighlight %}
and create a new file using your favorite editor
{% highlight bash %}
code main.tf
{% endhighlight %}

In the file, we insert the following block of text:
{% highlight terraform %}
provider "azurerm" {
    features {}
}

# We want a resource group to hold our cluster
resource "azurerm_resource_group" "example" {
  name     = "rg-test-terraform"
  location = "West Europe"
} 

resource "azurerm_kubernetes_cluster" "cluster" {
  name                = "aks-test-terraform"
  # We want our cluster to live in the resource group we just created
  # ... and use the same location.
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  dns_prefix          = "test"

  default_node_pool {
    name       = "default"
    node_count = 1
    vm_size    = "Standard_B2s"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = {
    Environment = "Production"
    Deployment = "Terraform"
  }
}
{% endhighlight %}
The first `resource` block tells Terraform that we want it to create a resource group. As Terraform stores the configuration of the resources, we can access the name and location of the group in the cluster block. 

In this demonstration we don't require a lot of processing power nor RAM. To keep cost down, we use the relatively cheap `B2s` VM size and acquire only a single node. This bad boy'll run you about `35USD/mo`

## Terraform init

Having created our configuration plan, we are ready to initialize terraform. 
```
terraform init
```
Hopefully you'll see
```bash
Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/azurerm...
- Installing hashicorp/azurerm v2.76.0...
- Installed hashicorp/azurerm v2.76.0 (signed by HashiCorp)

...
```
which tells use that Terraform has detected that we will be provisioning resources from the `azurerm` provider, found the latest version, and installed it for us. That's neat!

## Terraform plan

In real-world applications, the configuration can quickly become convoluted and hard to read. It is then a sound idea to investigate that the configuration plan provisions exactly what you want it to, and nothing more. Naturally you can do this by executing the script, but Terraform offers an alternative in which it foresees what will be provisioned once the configuration is applied.
```bash
terraform plan
```

```bash
...
Terraform will perform the following actions:

  # azurerm_kubernetes_cluster.cluster will be created
  + resource "azurerm_kubernetes_cluster" "cluster" {
      + dns_prefix                          = "test"
      + fqdn                                = (known after apply)

...
      + default_node_pool {
          + vm_size              = "Standard_B2s"
        }
...


  # azurerm_resource_group.example will be created
  + resource "azurerm_resource_group" "example" {
      + id       = (known after apply)
      + location = "westeurope"
      + name     = "rg-test-terraform"
    }

Plan: 2 to add, 0 to change, 0 to destroy.
...
```
Awesome! Terraform tells us that if this configuration is applied as is, it will create a single resource group and a single AKS. In addition, we observe that the correct node size `B2s` is selected.

## Terraform apply

Having validated our configuration, we are ready to execute
```
terraform apply -auto-approve
```
which should yield
```bash
azurerm_resource_group.example: Creating...
azurerm_resource_group.example: Creation complete after 1s [id=/subscriptions/***/resourceGroups/rg-test-terraform]
azurerm_kubernetes_cluster.cluster: Creating...
azurerm_kubernetes_cluster.cluster: Creation complete after 4m9s [id=/subscriptions/***/resourcegroups/rg-test-terraform/providers/Microsoft.ContainerService/managedClusters/aks-test-terraform]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

## Creating our K8s deployment
To create a deployment in our newly created cluster, we need to configure it as a context for `kubectl`. The Azure CLI offers a convenient command 
```
az aks get-credentials --resource-group rg-test-terraform --name aks-test-terraform
```
for achieving this. 

Finally, we deploy the 'hello-world' example of Kubernetes
```bash
kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0
```
and expose it.
```
kubectl expose deployment web --type=LoadBalancer --port=8080
```
After some seconds we can see the external IP of the service
```
kubectl get services
```
and we can `curl` the contents of the web app
```
curl <EXTERNAL-IP>:8080
```
which gives us
```
Hello, world!
Version: 1.0.0
Hostname: web-79d88c97d6-hc4fz
```


## Cleaning up

To make sure we don't accrue any unnecessary costs in Azure, we will delete the resource group. Instead of using the Azure CLI, we can use Terraform's own command for deprovisioning the configuration.
```
terraform destroy 
```
Ensure that you're in the folder you created the configuration in.

If something in this guide didn't work, please [open an issue on my Github page](https://github.com/mikalst/mikalst.github.io/issues/new?title=New+issue&body=Describe+the+problem)