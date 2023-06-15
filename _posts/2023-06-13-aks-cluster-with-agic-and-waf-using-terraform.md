---
title: Setting up an AKS cluster with Application Gateway Ingress Controller,Web Application Firewall and Let's Encrypt using Terraform.
author: Thomas Luijken
date: 2023-06-13 21:35:00 +0200
categories: [Azure, DevOps, Infrastructure As Code, Terraform]
tags: [DevOps, Azure, Aks, Kubernetes, Terraform, Let's Encrypt, Firewall]
hidden: true;
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

## Struggles in Setting Up a Firewall for a Kubernetes Cluster

In my initial attempt to set up a firewall for my Kubernetes cluster, I followed
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
these discussions, I quickly learned that an Application Gateway Ingress
Controller with a Web Application Firewall would be a more suitable solution for
my needs. The Web Application Firewall offers valuable features such as packet
inspection and bot detection, making it better suited to protect my services
from malicious requests.

With a sense of relief, I decided to leverage the Application Gateway Ingress
Controller along with the Web Application Firewall to enhance the security of my
Kubernetes cluster.
