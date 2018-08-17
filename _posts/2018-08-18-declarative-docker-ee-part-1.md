---
title: Declarative Docker EE with Packer, Terraform, Ansible and GitLab - part 1
tags: [docker,docker ee,packer,terraform,ansible,gitlab,series,enterprise]
description: At [Alm. Brand](https://www.almbrand.dk), we've been running Docker in production since the first beta of Docker Universal Control Plane (UCP). This is the story about how we moved on to a more automated and declarative approach.
excerpt: At [Alm. Brand](https://www.almbrand.dk), we've been running Docker in production since the first beta of Docker Universal Control Plane (UCP). This is the story about how we moved on to a more automated and declarative approach.
image:
uuid: df2bf47c-a134-11e8-b9ae-6c3be53e3c96
---

# Background

At [Alm. Brand](https://www.almbrand.dk), we've been running Docker in production since the first beta of Docker Universal Control Plane (UCP). This is the story about how we moved on to a more automated and declarative approach.

We started with greenfield services, a simple service discovery, config management and dynamic load balancing setup and accelerated quickly. After having proven the new Docker based platform, interest grew in Dockerizing and migrating the legacy apps running on our organically grown application server infrastructure, which had become painful to manage, and required daily firefighting.

In our [DockerCon Europe 2017 talk](https://www.youtube.com/watch?v=nI9WhhtFmFs), we describe our process and the gains from our migration journey, so I will not dive further into that in this post. Rather, I will describe how we became victims of our own success, and what we did to better the situation.

# The problem

In the summer of 2017, it became clear that co-locating our license constrained legacy apps and our open-source based greenfield services would prevent us from scaling our infrastructure to match the growing number of new services and migrated apps. As we're an Enterprise™, it would become the end of 2017 before we were able to start working on a solution, since, finally, the resources in the cluster were becoming exhausted, causing outages and painful firefighting in both dev/test and production.

The most straightforward solution would be to build a new Docker cluster and migrate the greenfield services to it, thus freeing up resources on the first cluster. To pull that off as fast as possible, that would of course require several key employees to be available within a short amount of time in order to modify external load balancers, create DHCP reservations, open firewall ports, create and provision VMs, etc.

However, we were determined to make better use of our learnings from building and running the first cluster. We temporarily scaled up the first cluster, the bulk cost of which was additional licenses for the proprietary legacy software, and thus bought ourselves time for creating a better solution.

# Our solution

![My helpful screenshot](/assets/images/2018-08-16-ucp-provisioner-pipeline-1.png)

Some of our learnings from the first two years of running a Docker cluster were:

* Keeping OS packages on VMs up to date, even with config management tools, does not completely prevent configuration drift
* Manual work leads to human error, some of which will only be discovered further down the line
* If only a few people work daily with a system, it will become less approachable to newcomers, unless an effort is made to codify and automate it
* As much as possible of the data and process involved in creating or modifying a cluster should be versioned and executable as a pipeline
* When the processes are different for changing different components, it requires more knowledge and context switching. Simplifying processes can save time and reduce errors.

## Creating a golden image

Since we run our own VMware based infrastructure, we have to create VMs from scratch. Fortunately, [HashiCorp Packer](https://www.packer.io/) is a very useful tool in that regard. JetBrains have made a [VMware vSphere plugin for Packer](https://github.com/jetbrains-infra/packer-builder-vsphere) that helps create a new VM or VM template from a given ISO file. The plugin also supports creating a virtual floppy drive to be made available to the installer, which we make use of to supply a Kickstart script. The Kickstart script configures the system account that we use in the following stages, and installs basic packages and configuration such as NTP, internal CA certificates, corporate proxy environment variables, timezone, etc.

This process runs in a GitLab pipeline for a repo aptly named `golden-image-base`. In order to keep iteration times short, we save the resulting VM as a template at this stage and hand off to another repo's pipeline, named `golde-image-docker-ucp`.

In the Docker + UCP repo's pipeline, we clone the base VM template and use Packer's Ansible provisioner to provision it further. Among other things, we use [govc](https://github.com/vmware/govmomi/blob/master/govc/README.md) to add a separate disk for the `/var/lib/docker` partition as recommended in the [CIS CE Benchmark v1.1.0](https://success.docker.com/api/asset/.%2Frefarch%2Fsecurity-best-practices%2FCIS_Docker_Community_Edition_Benchmark_v1.1.0.pdf) section 1.1. We also installed a configurable version of Docker Engine and pre-pull the UCP and DTR images from Docker Hub. To get the list of images to pull for UCP, you can run `docker run --rm docker/ucp images --list`. Beware that the output contains carriage returns (`\r`), which can interfere with automation, so pipe it through `tr -d '\r'`. For DTR, the command is `docker run --rm docker/dtr images`, and it *also* yields carriage returns in the output.

The final VM template is now ready to be used by the pipeline of the next repo in the process, `ucp-provisioner`. That will be described in the next part of the series.
