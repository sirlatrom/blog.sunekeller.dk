---
title: Declarative Docker Enterprise with Packer, Terraform, Ansible and GitLab - part 2
tags: [docker,docker enterprise,packer,terraform,ansible,gitlab,series,enterprise]
description: Part 2
excerpt: Part 2
image: 
uuid: 43e9242c-a205-11e8-a622-6c3be53e3c96
---

# Background

See [part 1]().

# The upgrade process

Once any new nodes have been joined to the cluster, the rolling upgrade can be performed. It starts out with a list of what template was used for each VM currently in the cluster, then swaps out one at a time with the most recent template.

1. Retrieve list of templates for current nodes
1. For each stage (usually in this order: UCP, DTR, other workers):
    1. For each VM in that stage:
        1. Update the template name to the most recent from the catalog (the selection mechanism is likely to be improved to be more granular)
        1. Run terraform plan followed by terraform apply, resulting in:
            1. Drain the node from running tasks
            1. Perform additional tasks for UCP* or DTR** nodes
            1. Having the node leave the swarm and remove it from the node list
            1. Destroying the existing VM in vSphere
            1. Create a new VM in vSphere cloned from the new template
            1. Join the new VM to the Swarm cluster, wait for it to be marked as healthy
            1. Put the VM in the right UCP collection (using node labels) so tasks can be scheduled on it
            1. Perform additional tasks for UCP* or DTR** nodes
        {:.lower_roman_list}
    {:.lower_alpha_list}

\* If the node being upgraded is one of the UCP controllers, when it is to be destroyed, it is demoted such that the other controllers will be made to know about the (temporary) new number of controllers.

\*\* If the node being upgraded is one of the DTR replicas, when it is to be destroyed, it is first removed from the DTR cluster such that the other replicas will know about the (temporary) new number of replicas. When its replacement has been joined to the cluster as a worker node, it is additionally joined to the DTR cluster to become one of the available replicas.

## A quick note on tools and our environment

The setup I describe in this series runs on our on-premises 2 location stretched VMware datacenter. For building VM templates, we use [HashiCorp Packer](https://www.packer.io/) and a customized Ubuntu 16.04 netboot installer ISO. For creating or replacing VMs we use [HashiCorp Terraform](https://www.terraform.io/). To orchestrate the different stages of the pipeline, we use [GitLab](https://www.gitlab.com). We run [Ansible](https://www.ansible.com) playbooks from within Terraform. We use [govc](https://github.com/vmware/govmomi/blob/master/govc/README.md) for several VM related tasks.

## Overview

The overall flow for a new cluster consists of a few high-level steps:

1. Create a base VM template from a custumized Ubuntu installation ISO file
2. Create a Docker + UCP VM template on top of the template made in step 1 with all the necessary packages to install and run Docker Enterprise
3. Create a configured list of VMs for a given cluster environment
4. Configure the created VMs by installing and joining together a Docker UCP cluster, optionally with a DTR cluster and predefined UCP teams and grants

## VM templates

### The base VM template

This is the first part of our pipeline and resides in its own repo in GitLab.

The base VM template is created on a nightly GitLab schedule to always have a template that has the most up to date base Ubuntu packages. To keep concerns separated, and to limit build time for each stage, the base VM template creation sits in its own repo and runs its own build jobs, whereas the Docker + UCP VM template is created in a subsequent repo.

In order to create a VM template from an ISO file, we use HashiCorp Packer, which can assemble VM templates and other artifact formats from various sources.

The ISO file is a PXE booting Ubuntu installer that is preconfigured (in the ISO file) to obtain a Kickstart file (ks.cfg) from a floppy disk created ad-hoc by Packer. It will install a basic set of packages and configure a user which will be used in the following parts of the flow.

Once built, the VM template will exist under a specific folder in VMware vCenter. If successful, the CI pipeline triggers the Docker + UCP repo's CI pipeline in order to create a new Docker + UCP template with the up to date base packages.

### The Docker + UCP VM template

This is the second part of our pipeline and resides in its own repo in GitLab.

The Docker + UCP VM template is built on top of the base VM template. The concern of this part is to install and configure the necessary packages for an instance to be part of the cluster. It uses HashiCorp Packer to clone the base VM template and run Ansible locally on the VM in order to install required packages, apt repos for Docker Enterprise, and configure HTTP proxy and CA certificate trust, among other things.

The list of packages that an operator can expect to find on every instance is listed in a playbook in this repo, and rather than installing additional tools or packages on all the individual nodes after they have been created, it is recommended to add them to this repo and perform a rolling upgrade of the cluster.

Before installing the Docker package, a separate VM disk and partition is created for the `/var/lib/docker` directory using a combination of [`govc vm.disk.create`](https://github.com/vmware/govmomi/blob/master/govc/USAGE.md#vmdiskcreate) and ansible tasks to partition, format and mount the disk, and the systemd service unit for the Docker service is configured to set the wanted options to use for the Docker daemon.

After the VM template has been created, a script cleans up unused, old VM templates to keep things tidy.

## Upgrading a cluster

Once a VM template has been prepared, it can be rolled out by performing a rolling upgrade.

### Procedure

#### Checking the plan

The actions that Terraform will take when the Terraform configuration is applied is visible through the "Plan" part of the pipeline which describes each resource that is to be added, changed or destroyed. Some resources are VMs, others are of different, Terraform-native types that are used to run scripts at particular points in the life-cycle.

#### Ensuring the current state is in order

After checking the plan, it should be applied to join any new nodes before upgrading. Any new nodes added to the Terraform configuration for a cluster will be created and joined to the cluster.

#### Upgrading the nodes one at a time