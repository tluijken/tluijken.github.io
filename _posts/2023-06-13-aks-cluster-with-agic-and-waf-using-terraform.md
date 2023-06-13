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

## The struggles
