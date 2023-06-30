---
title: Setting up an AKS cluster with Application Gateway Ingress Controller,Web Application Firewall and Let's Encrypt using Terraform.
author: Thomas Luijken
date: 2023-06-13 21:35:00 +0200
categories: [Azure, DevOps, Infrastructure As Code, Terraform]
tags: [DevOps, Azure, Aks, Kubernetes, Terraform, Let's Encrypt, Firewall]
hidden: true
---

In today's microservice-oriented era, backend developers often find themselves
taking on DevOps tasks. However, let's face the truth - being a DevOps engineer
is a trade of its own, with which software developers usually have limited
familiarity. While developers can manage basic tasks, they can quickly find
themselves out of their comfort zone when faced with more complex challenges.

I recently found myself in this very situation. Having successfully set up
multiple Kubernetes clusters in Azure, Google Cloud, and other platforms using
various automation techniques, I was confident in my abilities to tackle a new
project. However, this time, the requirements were different. I needed to
automate the process using Terraform, while also incorporating incoming traffic
routing through an Application Gateway and ensuring its security with a Web
Application Firewall.

As I delved into the documentation, I was quickly overwhelmed by the abundance
of information. Navigating through the documentation became a challenge, and I
struggled to take concrete actions. In this blog post, I will share my findings
and experiences, aiming to assist those who may be facing similar hurdles.

All source code can be found in my [GitHub Repository](https://github.com/tluijken/terraform_aks_example).

## Struggles in Setting Up a Firewall for a Kubernetes Cluster

In my initial attempt to set up a firewall for a Kubernetes cluster, I followed
Microsoft's
[article](https://learn.microsoft.com/en-us/azure/firewall/protect-azure-kubernetes-service)
that described the use of a traditional firewall. I managed to successfully
configure the traditional firewall and convert the provided scripts into a
Terraform setup. Additionally, I implemented a [Hub and Spokes
architecture](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke?tabs=cli)
for improved network management.

Nevertheless, I encountered difficulties when attempting to install an nginx
ingress controller using a helm chart, primarily due to [routing
issues](https://github.com/kubernetes/ingress-nginx/issues/7626#issuecomment-944053390).
Surprisingly, the initial Microsoft article did not mention the need to set up
an nginx ingress controller; it only referred to a load balancer service that
directed traffic directly to a single deployment or replica set.

I explored other [useful
resources](https://denniszielke.medium.com/fully-private-aks-clusters-without-any-public-ips-finally-7f5688411184)
in an attempt to find a solution, but unfortunately, I faced the same recurring
issues. It became evident that my knowledge of networking was insufficient to
resolve these challenges.

Feeling stranded, I reached out to my network for support and sought advice from
DevOps engineers on how they typically secure their Kubernetes clusters. Through
these discussions, I quickly learned that an [Application Gateway Ingress
Controller](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview)
with a [Web Application
Firewall](https://learn.microsoft.com/en-us/azure/web-application-firewall/ag/ag-overview)
would be a more suitable solution for my needs. The Web Application Firewall
offers valuable features such as packet inspection and bot detection, making it
better suited to protect services from malicious requests.

With a sense of relief, I decided to leverage the Application Gateway Ingress
Controller along with the Web Application Firewall to enhance the security of
the Kubernetes cluster.

## Setting up a Application Gateway Ingress Controller and Web Application Firewall.

Initially I just enabled the Application Gateway Ingress Controller by checking
the checkbox under 'Network'. After that I also enabled the Web Application
Firewall through the Azure Portal. So far, super easy setup. However, I found
myself searching for answers when setting up the 'Backend pools', 'SSL
settings', 'Listeners and Rules'. How does this translate to traffic into the
cluster? So many options to review:
![image](../assets/img/agic_terraform/backend_setting.png)
![image](../assets/img/agic_terraform/routing.rules.png)

And what about these 'backend pools', which IP address to I assign? As far as I
could understand from the image, the AGIC would direct traffic directly to the
pods, and those IP's are super dynamic.
![image](../assets/img/agic_terraform/backend_pool.png)

I felt lost again in the Azure ecosystem, and needed to get some more
information. Surely, I'm not the first to set this up, why isn't this straight
forward. Surely, [the documentation on the Terraform
registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/application_gateway)
will indicate quickly how to set this up with easy right?
```terraform
resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "West Europe"
}

resource "azurerm_virtual_network" "example" {
  name                = "example-network"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  address_space       = ["10.254.0.0/16"]
}

resource "azurerm_subnet" "frontend" {
  name                 = "frontend"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = ["10.254.0.0/24"]
}

resource "azurerm_subnet" "backend" {
  name                 = "backend"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = ["10.254.2.0/24"]
}

resource "azurerm_public_ip" "example" {
  name                = "example-pip"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  allocation_method   = "Dynamic"
}

# since these variables are re-used - a locals block makes this more maintainable
locals {
  backend_address_pool_name      = "${azurerm_virtual_network.example.name}-beap"
  frontend_port_name             = "${azurerm_virtual_network.example.name}-feport"
  frontend_ip_configuration_name = "${azurerm_virtual_network.example.name}-feip"
  http_setting_name              = "${azurerm_virtual_network.example.name}-be-htst"
  listener_name                  = "${azurerm_virtual_network.example.name}-httplstn"
  request_routing_rule_name      = "${azurerm_virtual_network.example.name}-rqrt"
  redirect_configuration_name    = "${azurerm_virtual_network.example.name}-rdrcfg"
}

resource "azurerm_application_gateway" "network" {
  name                = "example-appgateway"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location

  sku {
    name     = "Standard_Small"
    tier     = "Standard"
    capacity = 2
  }

  gateway_ip_configuration {
    name      = "my-gateway-ip-configuration"
    subnet_id = azurerm_subnet.frontend.id
  }

  frontend_port {
    name = local.frontend_port_name
    port = 80
  }

  frontend_ip_configuration {
    name                 = local.frontend_ip_configuration_name
    public_ip_address_id = azurerm_public_ip.example.id
  }

  backend_address_pool {
    name = local.backend_address_pool_name
  }

  backend_http_settings {
    name                  = local.http_setting_name
    cookie_based_affinity = "Disabled"
    path                  = "/path1/"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 60
  }

  http_listener {
    name                           = local.listener_name
    frontend_ip_configuration_name = local.frontend_ip_configuration_name
    frontend_port_name             = local.frontend_port_name
    protocol                       = "Http"
  }

  request_routing_rule {
    name                       = local.request_routing_rule_name
    rule_type                  = "Basic"
    http_listener_name         = local.listener_name
    backend_address_pool_name  = local.backend_address_pool_name
    backend_http_settings_name = local.http_setting_name
  }
}
```
...But perhaps not.

## The Hidden Key
Despite extensive digging through [the
documentation](https://learn.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-new),
the elusive piece of the puzzle was yet to be found.

The eureka moment arrived when I delved deeper into Microsoft's provided ingress
configuration. I discovered a tiny discrepancy that would pave the path to
success.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aspnetapp
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          service:
            name: aspnetapp
            port:
              number: 80
        pathType: Exact
```
Crucial in this context is the ingress.class annotation:
`kubernetes.io/ingress.class: azure/application-gateway`.

As I dived deeper, I observed a pod named 'ingress-appgw-deployment-****'
running in the cluster.

```bash
$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
....
ingress-appgw-deployment-85c4dc479-fc6l6   1/1     Running   0          5d20h
...
```
This 'ingress-appgw-deployment' is integrated into the Kubernetes cluster
post-activation of the Application Gateway feature. It effectively manages the
heavy lifting. Depending on the ingress configurations and the
`azure/application-gateway` ingress class, it automates the setup of routing,
backend pools, listeners, and more in the Application Gateway configuration.

With everything now set up correctly, I found myself capable of configuring a
Web Application Firewall, tied to the Application Gateway Ingress Controller.

## Shifting to Terraform

As I sought to configure all elements via Terraform, including firewall rules,
the path forward was not as straightforward as simply enabling the [Application
Gateway Ingress
Controller](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/kubernetes_cluster#ingress_application_gateway)
in the AKS cluster node. Given my intention to integrate the Application
Controller with the Web Application Firewall, I was required to first configure
the Web Application Firewall and Application Gateway and then bind them. This
process inevitably led me to manually set up the VNET/Subnets in Terraform.
Thankfully, my prior experience with the classic Firewall had instilled in me a
strong understanding of networking.

### Resource Group

I began by creating a resource group, which would house all the resources to
configure, except for the ones automatically created for the AKS cluster by
Azure itself.

```terraform
resource "azurerm_resource_group" "aks_appgw_demo" {
  name     = "aks-appgw-demo-rg"
  location = "westeurope"
}
```

### Web Application Firewall

With the resource group established, the next step was to set up the Web
Application Firewall Policy:

```terraform
resource "azurerm_web_application_firewall_policy" "aks_appgw_demo" {
  name                = "web-application-firewall"
  location            = azurerm_resource_group.aks_appgw_demo.location
  resource_group_name = azurerm_resource_group.aks_appgw_demo.name
  depends_on          = [azurerm_resource_group.aks_appgw_demo]

  policy_settings {
    enabled                     = true
    file_upload_limit_in_mb     = 100
    max_request_body_size_in_kb = 128
    mode                        = "Detection"
    request_body_check          = true
  }

  managed_rules {
    managed_rule_set {
      type    = "OWASP"
      version = "3.2"
    }

    managed_rule_set {
      type    = "Microsoft_BotManagerRuleSet"
      version = "1.0"
    }
  }
}
```
This basic configuration defines rudimentary rules for a Web Application
Firewall, including OWASP rules and Bot Detection Rules.

### Networking

Before the Application Gateway can be established, I had to manage some
networking prerequisites. These subnets would eventually be allocated to the
Application Gateway resource and the AKS cluster. To simplify modifications, I
created some local values.

```terraform
locals {
  vnet_address_space          = "10.224.0.0/15" # Ranges 10.224.0.1 till 10.225.255.254
  aks_subnet_address_prefix   = "10.224.0.0/16" # Ranges 10.224.0.1 till 10.224.255.254
  appgw_subnet_address_prefix = "10.225.0.0/16" # Ranges 10.225.0.1 till 10.225.255.254
  appgw_private_ip_address    = "10.225.0.100"  # Should reside in the appgw-subnet range
}
```

> **Note:** The virtual network's address space is meticulously defined to
> prevent overlaps and confusion, especially when operating multiple clusters
> such as DEV, STAGING, and PRODUCTION. I recommend sticking with recognizable
> IP ranges for each environment, as illustrated below:
>
> - DEV: 10.224.0.0/15 (IP range: 10.224.0.1 to 10.225.255.254)
> - STAGING: 10.226.0.0/15 (IP range: 10.226.0.1 to 10.227.255.254)
> - PRODUCTION: 10.228.0.0/15 (IP range: 10.228.0.1 to 10.229.255.254)
>
> This structure not only ensures clarity but also proves beneficial if you wish
> to incorporate a Jumpbox (HUB VNET) into your Azure environment. By peering
> the virtual networks of the AKS clusters with the HUB VNET, you can guarantee
> no overlapping address spaces, thus making it intuitively understandable for
> most engineers.

Next came the implementation of the network configuration.

```terraform
resource "azurerm_virtual_network" "aks_gw_vnet" {
  name                = "aks-appgw-vnet"
  location            = azurerm_resource_group.aks_appgw_demo.location
  resource_group_name = azurerm_resource_group.aks_appgw_demo.name
  depends_on          = [azurerm_resource_group.aks_appgw_demo]
  address_space       = [local.vnet_address_space]
}

resource "azurerm_subnet" "aks" {
  name                 = "aks-subnet"
  resource_group_name  = azurerm_resource_group.aks_appgw_demo.name
  depends_on           = [azurerm_resource_group.aks_appgw_demo, azurerm_virtual_network.aks_gw_vnet]
  virtual_network_name = azurerm_virtual_network.aks_gw_vnet.name
  address_prefixes     = [local.aks_subnet_address_prefix]
}

resource "azurerm_subnet" "application_gateway" {
  name                 = "ingress-appgateway-subnet"
  resource_group_name  = azurerm_resource_group.aks_appgw_demo.name
  depends_on           = [azurerm_resource_group.aks_appgw_demo, azurerm_virtual_network.aks_gw_vnet]
  virtual_network_name = azurerm_virtual_network.aks_gw_vnet.name
  address_prefixes     = [local.appgw_subnet_address_prefix]
}
```

Additionally, I needed a public IP address to assign as the public IP address for the Application Gateway.

```terraform
resource "azurerm_public_ip" "aks_appgw_demo" {
  name                = "appgw_public_ip"
  location            = azurerm_resource_group.aks_appgw_demo.location
  resource_group_name = azurerm_resource_group.aks_appgw_demo.name
  depends_on          = [azurerm_resource_group.aks_appgw_demo]
  allocation_method   = "Static"
  sku                 = "Standard"
}
```
### Application Gateway
Finally, I was ready to set up an Application Gateway. However, this part
introduced some confusion. I was required to define at least one `backend_pool`,
`http_backend_settings`, `http_listener`, and `request_routing_rule`. These
requirements seemed nonsensical since they would be replaced by the `app-gw` pod
during the synchronization process.

I filled them with placeholder data, knowing that they would be deleted and
replaced with relevant configurations later. This brought up another issue: once
the `app-gw` pod starts setting configurations to the Application Gateway, the
configuration no longer matches the state file. Running `terraform apply` at that
point would result in recreating the resource, reverting back to the placeholder
configuration. To avoid this, I included some filters for 'lifecycle
management'.

```terraform
resource "azurerm_application_gateway" "aks_appgw_demo" {
  name                = "application-gateway"
  location            = azurerm_resource_group.aks_appgw_demo.location
  resource_group_name = azurerm_resource_group.aks_appgw_demo.name
  depends_on          = [azurerm_resource_group.aks_appgw_demo, azurerm_web_application_firewall_policy.aks_appgw_demo, azurerm_subnet.application_gateway, azurerm_public_ip.aks_appgw_demo]
  firewall_policy_id  = azurerm_web_application_firewall_policy.aks_appgw_demo.id

  sku {
    name     = "WAF_v2"
    tier     = "WAF_v2"
    capacity = 2
  }

  gateway_ip_configuration {
    name      = "gateway-ip-configuration"
    subnet_id = azurerm_subnet.application_gateway.id
  }


  frontend_port {
    name = "port_80"
    port = 80
  }

  # Assign the public IP to the application gateway
  frontend_ip_configuration {
    name                 = "PublicFrontendIp"
    public_ip_address_id = azurerm_public_ip.aks_appgw_demo.id
  }

  frontend_ip_configuration {
    name                          = "PrivateFrontendIp"
    private_ip_address            = local.appgw_private_ip_address
    private_ip_address_allocation = "Static"
    subnet_id                     = azurerm_subnet.application_gateway.id
  }

  # Initial placeholder configuration for backend, listeners, rules.
  # These are required for setting up the application gateway.
  # In the aks cluster, a synchronization pod will be created,
  # which will update the application gateway's
  # listeners, back-ends, and rules in line with the ingress configuration.
  # This will eventually remove the placeholder items created here.
  backend_address_pool {
    name = "dummyBackend"
  }

  backend_http_settings {
    name                  = "dummyBackendSettings"
    cookie_based_affinity = "Disabled"
    path                  = "/path1/"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 60
  }

  http_listener {
    name                           = "dummyListener"
    frontend_ip_configuration_name = "PrivateFrontendIp"
    frontend_port_name             = "port_80"
    protocol                       = "Http"
  }

  request_routing_rule {
    name                       = "dummyRule"
    rule_type                  = "Basic"
    http_listener_name         = "dummyListener"
    backend_address_pool_name  = "dummyBackend"
    backend_http_settings_name = "dummyBackendSettings"
    priority                   = 100
  }

  # Since the rules, listeners, and backends are dynamically updated at runtime
  # by the application gateway pod, based on the ingress configuration, I've chosen
  # to ignore these changes for lifecycle management.
  lifecycle {
    ignore_changes = [
      backend_address_pool,
      http_listener,
      backend_http_settings,
      request_routing_rule,
      probe,
      redirect_configuration,
      url_path_map,
      ssl_certificate,
      frontend_port,
      tags
    ]
  }
}
```

### AKS Cluster
With the Network, Web Application Firewall, and Application Gateway in place, I can finally add the AKS cluster to the terraform configuration.

This configuration is quite basic, but there are some important settings to note:
* `node_resource_group` - the name I want to assign to the resource group that Azure will create for the AKS resources. This is a personal preference, but I want to have control over this.
* `default_node_pool` > `vnet_subnet_id` - here, I assign our preconfigured subnet to the AKS-cluster node pool.
* `ingress_application_gateway` - this is where I connect the Application Gateway to the AKS cluster.

```terraform
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "app_gw_demo"
  location            = azurerm_resource_group.aks_appgw_demo.location
  resource_group_name = azurerm_resource_group.aks_appgw_demo.name
  depends_on          = [azurerm_resource_group.aks_appgw_demo, azurerm_application_gateway.aks_appgw_demo]
  dns_prefix          = "aks-appgw-demo"

  role_based_access_control_enabled = true
  node_resource_group               = "${azurerm_resource_group.aks_appgw_demo.name}-aks-nodes"

  default_node_pool {
    name            = "default"
    node_count      = 2
    vm_size         = "Standard_DS2_v2"
    os_disk_size_gb = 30
    # the aks cluster can be deployed into the existing virtual network.
    vnet_subnet_id = azurerm_subnet.aks.id
  }

  identity {
    type = "SystemAssigned"
  }

  ingress_application_gateway {
    # bind our aks cluster to the gateway
    gateway_id = azurerm_application_gateway.aks_appgw_demo.id
  }
}
```
# Deploying apps
Before we can deploy applications, we must configure the Kubernetes provider.
This provider draws its kube configuration from the
`azurerm_kubernetes_cluster.aks` resource, which becomes available
post-creation. Although my standard recommendation would be to utilize a
[Multi-Layer
Architecture](https://www.padok.fr/en/blog/terraform-iac-multi-layering) for
this type of setup, for demonstration purposes, I have achieved the desired
outcome using an intermediary data block.

```terraform
# Temporary solution for the multi-layer architecture, utilized solely for demonstration
data "azurerm_kubernetes_cluster" "cluster" {
  name                = azurerm_kubernetes_cluster.aks.name
  resource_group_name = azurerm_kubernetes_cluster.aks.resource_group_name
  depends_on = [ azurerm_kubernetes_cluster.aks ]
}

provider "kubernetes" {
  host                   = data.azurerm_kubernetes_cluster.cluster.kube_config[0].host
  client_certificate     = base64decode(data.azurerm_kubernetes_cluster.cluster.kube_config[0].client_certificate)
  client_key             = base64decode(data.azurerm_kubernetes_cluster.cluster.kube_config[0].client_key)
  cluster_ca_certificate = base64decode(data.azurerm_kubernetes_cluster.cluster.kube_config[0].cluster_ca_certificate)
}
```

This Kubernetes provider can now be leveraged to deploy our applications to the
AKS cluster.

```terraform
resource "kubernetes_deployment_v1" "hello_world" {
  depends_on = [ azurerm_kubernetes_cluster.aks ]
  metadata {
    name = "aks-hello-world"
  }
  spec {
    replicas = 1
    selector {
      match_labels = {
        app = "aks-hello-world"
      }
    }
    template {
      metadata {
        labels = {
          app = "aks-hello-world"
        }
      }
      spec {
        container {
          image             = "mcr.microsoft.com/azuredocs/aks-helloworld:v1"
          name              = "aks-hello-world"
          image_pull_policy = "Always"
          port {
            container_port = 80
          }
          env {
            name  = "TITLE"
            value = "Welcome to Azure Kubernetes Service (AKS)"
          }
        }
      }
    }
  }
}
resource "kubernetes_service_v1" "hello_world" {
  metadata {
    name = "aks-hello-world"
  }
  spec {
    selector = {
      app = kubernetes_deployment_v1.hello_world.spec[0].template[0].metadata[0].labels.app
    }
    type = "NodePort"
    port {
      port        = 80
      target_port = 80
    }
  }
}
```

One aspect I appreciate about using Terraform in this scenario is the ability to
derive certain values, such as the app selector, from other resources. It
supports auto-completion and permits derivation of dependencies, ensuring a
controlled deployment and creation of services within the cluster.

At this point, we have a sample application successfully deployed to the AKS
cluster.

```bash
$ kubectl get all -n default
NAME                                   READY   STATUS    RESTARTS   AGE
pod/aks-hello-world-8485768b75-vdbpx   1/1     Running   0          11m

NAME                      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
service/aks-hello-world   NodePort    10.0.50.9    <none>        80:31330/TCP   11m

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/aks-hello-world   1/1     1            1           11m

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/aks-hello-world-8485768b75   1         1         1       11m
```

## Configuring the Application Gateway Ingress Controller
With the deployment now fully established, we can proceed to set up our Ingress
Controller. Don't forget to utilize the `azure/application-gateway` ingress
class, enabling the `app_gw` deployment to configure our Application Gateway
accordingly.

```terraform
resource "kubernetes_ingress_v1" "hello-world" {
  metadata {
    name = "hello-world-ingress"
    annotations = {
      "kubernetes.io/ingress.class"              = "azure/application-gateway"
    }
  }

  spec {
    rule {
      http {
        path {
          backend {
            service {
              name = kubernetes_service_v1.hello_world.metadata[0].name
              port {
                number = 80
              }
            }
          }
          path = "/"
        }
      }
    }
  }
}
```
This configuration will guide the Ingress Controller to correctly direct
incoming requests.

## Access denied!

After applying the ingress configuration, I was unable to see the content served
by the `aks-hello-world` pod. Moreover, there seemed to be no changes made to my
Application Gateway configuration.

Upon investigating the `ingress_appgw_deployment` pod, I discovered an error:

```bash
E0630 10:16:30.128059       1 client.go:175] Code="ErrorApplicationGatewayForbidden" Message="Unexpected status code '403' while performing a GET on Application Gateway. You can use 'az role assignment create --role Reader --scope /subscriptions/*******/resourceGroups/aks-appgw-demo-rg --assignee ********; az role assignment create --role Contributor --scope /subscriptions/*********/resourceGroups/aks-appgw-demo-rg/providers/Microsoft.Network/applicationGateways/application-gateway --assignee ***********' to assign permissions. AGIC Identity needs at least 'Contributor' access to Application Gateway 'application-gateway' and 'Reader' access to Application Gateway's Resource Group 'aks-appgw-demo-rg'." InnerError="network.ApplicationGatewaysClient#Get: Failure responding to request: StatusCode=403 -- Original Error: autorest/azure: Service returned an error. Status=403 Code="AuthorizationFailed" Message="The client '*******************' with object id '***************' does not have authorization to perform action 'Microsoft.Network/applicationGateways/read' over scope '/subscriptions/******************/resourceGroups/aks-appgw-demo-rg/providers/Microsoft.Network/applicationGateways/application-gateway' or the scope is invalid. If access was recently granted, please refresh your credentials.""
```

To resolve this issue, we need to verify that our `System Assigned Identity` has
the necessary permissions to read and write to the Application Gateway
Configuration.

We can rectify this issue by setting the proper roles:

```terraform
data "azurerm_user_assigned_identity" "aks_agw_uid" {
  resource_group_name = azurerm_kubernetes_cluster.aks.node_resource_group
  name                = "ingressapplicationgateway-${azurerm_kubernetes_cluster.aks.name}"
  depends_on          = [azurerm_kubernetes_cluster.aks]
}

# The application gateway identity should have the Contributor role for the application gateway.
resource "azurerm_role_assignment" "app_gw_contributor" {
  principal_id                     = data.azurerm_user_assigned_identity.aks_agw_uid.principal_id
  role_definition_name             = "Contributor"
  scope                            = azurerm_application_gateway.aks_appgw_demo.id
  skip_service_principal_aad_check = true
  depends_on                       = [data.azurerm_user_assigned_identity.aks_agw_uid]
}

# Make sure the application gateway is able to make changes to the network
resource "azurerm_role_assignment" "network_contributor" {
  principal_id                     = data.azurerm_user_assigned_identity.aks_agw_uid.principal_id
  role_definition_name             = "Network Contributor"
  scope                            = azurerm_resource_group.aks_appgw_demo.id
  skip_service_principal_aad_check = true
  depends_on                       = [data.azurerm_user_assigned_identity.aks_agw_uid]
}

# The application gateway identity should have the Reader role for the target resource group.
resource "azurerm_role_assignment" "resource_group_reader" {
  principal_id                     = data.azurerm_user_assigned_identity.aks_agw_uid.principal_id
  role_definition_name             = "Reader"
  scope                            = azurerm_resource_group.aks_appgw_demo.id
  skip_service_principal_aad_check = true
}
```

Once these roles are set, the managed identity created for the Application
Gateway Ingress Controller should be able to adjust all network settings and the
Application Gateway itself accordingly.
