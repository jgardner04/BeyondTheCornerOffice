---
layout: post
title:  "Deploy the Spark Operator on Kubernetes Via Helm 3"
date:   2020-11-30 16:00:00
categories: Technology
tags: [ Spark, Helm, DevOps]
author: Jonathan
cover-img: '/assets/img/posts/dominik-luckmann-SInhLTQouEk-unsplash.jpg'
imagecredit_id: '@exdigy'
imagecredit_name: 'Dominik Lückmann'
share-description: Deploy the Spark Operator on Kubernetes Via Helm 3 #Spark #Helm
---

[Apache Spark](//spark.apache.org) is a high performance batch processing engine who's power draws from the distributed processing of data. A single job is parsed out to multiple worker nodes to compute. [Kubernetes](//kubernetes.io) native scheduler and job processing is driving more and more companies to investigate using the two platforms together. Currently [Kubernetes scheduler support](//spark.apache.org/docs/latest/running-on-kubernetes.html) is available but considered experimental. 

There is lots of work going into making Spark work on Kubernetes. This includes by the Google Cloud Platform team at Google. The [Spark Operator Project](//github.com/GoogleCloudPlatform/spark-on-k8s-operator) they have been working on creates a [Custom Resource Definition](//kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) in Kubernetes allowing Spark jobs to be submitted as native Kubernetes objects. The installation instructions currently take advantage of using Helm for deployment but these instructions rely on Helm v2 and the now deprecated central charts repo. 

I have created [PR #1066](//github.com/GoogleCloudPlatform/spark-on-k8s-operator/pull/1066) that lays the groundwork to ready the repo to install the chart via Helm 3 but the repo owners still have some admin work to do that will finalize ability deploy the operator. You can track the progress of this [Issue #1071](//github.com/GoogleCloudPlatform/spark-on-k8s-operator/issues/1071)

Users looking for a workaround have two options. I have [forked](//github.com/jgardner04/spark-on-k8s-operator) the operator repo and created the branch necessary to allow for easy helm installation. The other option is to fork the repo yourself. In this article I will walk through the two options. 

> Code in this article assumes you have [Helm](//helm.sh/docs/intro/quickstart/) installed. 

## Using my forked repo
This is the quickest option but depending on how fast I can update the repo based on the upstream fork, it may be behind the current project's work.

```bash
helm repo add spark https://jgardner04.github.io/spark-on-k8s-operator/
helm repo update
helm install spark --namespace spark-operator
```

## Create your own fork
> This article assumes you are familiar with [creating a repo fork on GitHub](https://docs.github.com/en/free-pro-team@latest/github/getting-started-with-github/fork-a-repo)

This method will ensure that you have the latest code from the Google team and are not dependent on me to keep my fork updated.

Since [PR #1066](//github.com/GoogleCloudPlatform/spark-on-k8s-operator/pull/1066) is already merged, when you create the fork, the GitHub Action will already be in place to build and publish the chart to GitHub Pages. Clone your forked repo to your development machine. The following code will do the following: 
* create an [orphaned branch](//git-scm.com/docs/git-checkout#Documentation/git-checkout.txt---orphanltnewbranchgt) called `gh-pages` based on the master branch
* Remove all the files in the repo
* Create an `index.html` file with a repo title in it
* Push the branch back to your repo

```bash
git checkout master
git checkout --orphan -b gh-pages
git rm -rf .
rm '.gitignore'
echo "Spark Operator" > index.html
git add index.html
git commit -a -m "Initial Commit"
git push -u origin gh-pages
```

With this new branch created the GitHub Action can be run. From your forked repo follow **Actions->Release Charts->Re-run jobs**. This will initiate the packaging of the chart and building of the index.yaml file that is needed by Helm to deploy the chart. An index.yaml file will created and pushed to the gh-pages branch and a [release](//docs.github.com/en/free-pro-team@latest/github/administering-a-repository/about-releases) created. 

The final configuration step is to ensure that [GitHub Pages](https://pages.github.com) is enabled for the repo.  

In the code below, replace the `SITENAME` variable with the published location in the GitHub Pages section of your repo's settings. 

>The code below assumes you are connected to the Kubernetes where you want the Spark Operator deployed and have the Helm tools installed. 

```bash
SITENAME=<yousitename>
helm repo add spark $SITENAME
helm repo update
helm install spark --namespace spark-operator
```

I hope this article helped you get up to speed with Spark on Kubernetes faster. What has your experience been with Spark on Kubernetes? I'd love to hear your experiences in the comments below. If you found this helpful, please be sure to Upvote the article below and consider following me on social media or RSS. All the relevant links are in the footer. 