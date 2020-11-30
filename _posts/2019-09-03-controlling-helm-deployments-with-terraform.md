---
layout: post
title:  "Controlling Helm Deployments with Terraform"
date:   2019-09-03 16:00:00
categories: Technology
tags: [Azure, Terraform, Helm, DevOps]
author: Jonathan
cover-img: '/assets/img/posts/markos-mant-0nKRq0IknHw-unsplash.jpg'
imagecredit_id: '@markos_mant'
imagecredit_name: 'Markos Mant'
share-description: Controlling Helm Deployments with Terraform #Terraform #DevOps #Helm
---

Recently I have been working on a project to compare the performance of an application on Azure VMs and running on the Azure Kubernetes Service. To streamline the infrastructure deployment, the team centralized deployments using [HashiCorp’s Terraform](//www.hashicorp.com/products/terraform). In trying to deploy our application completely via Terraform we ran into some issues that led us to move the deployment to a Helm chart. The Helm deployment via Terraform had a few quirks as well. This article will talk about the journey and how I finally accomplished the application deployment.

> This article assumes the reader has working knowledge of Azure, Terraform, Kubernetes, and Helm. I have tried to link out to areas where the reader may go deeper on each.

## AKS and Persistent Storage
When building out the Terraform modules we needed for the environment I got the AKS cluster set up and configure then moved on to the actual application. I starting building out the application via the [Kubernetes resource provider](//registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/guides/getting-started). The application I am deploying includes a [Zookeeper cluster](//zookeeper.apache.org) to manage leader election.

The Zookeeper deployment includes a StatefulSet deployment that takes advantage of a VolumeClaim. I wanted to use Azure Files for the VolumeClaim. This means creating a Storage Class for Azure files as outlined below. (more info in the [AKS docs](//docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv).)

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000
parameters:
  skuName: Standard_LRS
```

The initial challenge is that the Kubernetes Terraform resource provider does not include an argument for mountOptions. A quick search of the GitHub repo found that this is being tracked in [Issue #510](//github.com/hashicorp/terraform-provider-kubernetes/issues/510). A lack of mount options meant that I could not use the Kubernetes provider to set up the cluster.

## Terraform Provisioners
Anyone who has used Terraform to deploy their applications has probably run into a situation where they can’t quite do all of their deployment via a resource provider and turned to [Terraform Provisioners](https://www.terraform.io/docs/provisioners/index.html) as a solution. This certainly could have been a viable option to include in our deployment pipeline. I had already used one after the creation of the AKS cluster to load the kube_config context on the deployment server for just this case.

Below is the code that I used to add the AKS config information onto my CI/CD system.

```yaml
provisioner "local-exec" {
command = "az aks get-credentials -g ${azurerm_kubernetes_cluster.nifi.resource_group_name} -n ${azurerm_kubernetes_cluster.nifi.name} --overwrite-existing"
}
```

Adding the StorageClass means adding the definition I outlined above to a file and then creating a second provisioner that executes the kubectl apply command. Something like the command below.

```yaml
provisioner "local-exec" {
command = "kubectl apply -f azure-file-sc.yaml"
}
```

While this is certainly a viable option, thinking about the larger scope of my application, I am going to need to deploy quite a few Kubernetes items. Some of these Kubernetes objects already have a YAML manifest. Using only the Kubernetes resource provider would mean breaking each of these up into their individual parts.

There must be a better way. Enter [Helm.sh](//helm.sh).

## Deploying via Helm Chart
This article is not meant to be a primer on Helm, they do that well on their website. The tl;dr version is that I can package my entire application in YAML manifests and have them deploy together. This solves my problem of having to break up each of these into individual Terraform resources. The [Helm Terraform provider](//registry.terraform.io/providers/hashicorp/helm/latest/docs) allows the deployment of the entire application in a single shot.

> As of publication of this article Helm v2 is the production release and what I used for this application. The version is important to note because the issues we faced during the Terraform Helm deployment will change with Helm v3.

Helm is made up of a local client (Helm) and a cluster component (Tiller). As a server side component, Tiller is subject to RBAC and a new service account needs to be created. The code below, outlines how I configured the template.

```yaml
resource "kubernetes_namespace" "tiller" {
  metadata {
    name = "tiller"
  }
}

resource "kubernetes_service_account" "tiller" {
  metadata {
    name      = "tiller"
    namespace = kubernetes_namespace.tiller.metadata.0.name
  }

  automount_service_account_token = true

}


resource "kubernetes_cluster_role_binding" "tiller" {
  metadata {
    name = kubernetes_service_account.tiller.metadata.0.name
  }
  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "ClusterRole"
    name      = "cluster-admin"
  }
  subject {
    kind      = "ServiceAccount"
    name      = kubernetes_service_account.tiller.metadata.0.name
    namespace = kubernetes_namespace.tiller.metadata.0.name
  }
}

resource "helm_release" "my-chart" {
  name         = "my-chart"
  chart        = "../my-chart"
  namespace    = kubernetes_namespace.tiller.metadata.0.name
  timeout      = 3600
  force_update = true

  set {
    name  = "domain"
    value = data.azurerm_kubernetes_cluster.test.addon_profile.0.http_application_routing.0.http_application_routing_zone_name
  }

  depends_on = [kubernetes_cluster_role_binding.tiller]
}
```

There are a few things to note about the code above that address some issue I ran into during the deployment.

### Timeout
While the deployment of most applications is a quick process, because the application I was deploying contained two StatefulSets, both of which used shared shared storage, the deployment takes quite a long time. Setting the `timeout = 3600` argument will ensure that the deployment does not fail causing Terraform to time out.

### Depends On
The creation of the cluster and the deployment of the application happened in the proper order but in a scenario where the cluster needs to be destroyed, Terraform would throw an error that tiller did not have permissions to read the ConfigMap. This happens because the ClusterRoleBinding is destroyed before the release is finished destroying. The `depends_on` will ensure that these are destroyed in the correct order.

## Conclusion
Terraform is a great deployment platform for Infrastructure as Code and is certainly my preferred method for deployment. That said, to make a Helm deployment that takes advantage of Azure Files via Terraform, there are some gotchas. Hope this helps iron out the kinks in your deployment.