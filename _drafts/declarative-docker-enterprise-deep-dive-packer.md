---
title: 'Deep Dive: Using Packer and Ansible to create a golden VMware image for Docker Enterprise'
tags: [docker,docker enterprise,packer,vmware,vsphere,ansible,gitlab,series,enterprise,deep dive]
description: In this post, I go into detail about how we build the VM template that is the basis of our Docker cluster.
excerpt: In this post, I go into detail about how we build the VM template that is the basis of our Docker cluster.
header:
  image: /assets/images/2018-08-22-golden-image-base-pipeline-1.png
uuid: c2147b10-a617-11e8-b3c1-6c3be53e3c96
---

# Background

In [part 1]({% post_url 2018-08-17-declarative-docker-enterprise-part-1 %}#creating-a-golden-image) of my series on Declarative Docker Enterprise, I describe how we used Packer to create a golden image for the VMs that will later make up our Docker Enterprise cluster. This post will detail how that is done.

**Note:** I will use the terms "Golden Image" and "VM template" interchangeably throughout this post.
{: .notice--info}

# The problem

In order to spin up VMs for a Docker cluster, you can either:

* boot and install each of them individually, with the very likely risk of making manual mistakes along the way, or
* create a VM template first, then clone that whenever you need a new VM.

Obviously, if you go and spin up and provision each VM individually, there are several drawbacks:

* The risk of package versions being different between the VMs
* The risk of manual mistakes along the way, selecting wrong packages or installer options
* Wasting precious time on repetitive tasks

If, on the other hand, you create a single VM *template* that contains all the packages you need, you only need to clone that template whenever you need a new VM.[^tf-plug]

[^tf-plug]: Obviously you'll still need to specify the unique config (e.g. hostname, VM name, possibly IP address) of each VM, but that's what Terraform is for. Read about that in an upcoming blog post.

Assuming you regularly create an upgraded golden image, all you need to do to *upgrade* your cluster is to replace each VM with a new one cloned from an upgraded VM template. As it turns out, this requires some amount of automation work, but yields the benefits of avoiding the drawbacks listed above, and more.

# Creating the VM template

Packer's involvement in our pipeline consists of two phases:

1. Create a plain Ubuntu 16.04 installation
1. Build on top of the first phase to add Docker related packages and configuration

The reason for splitting the process of creating the VM template that will be used to create or upgrade the cluster is to shorten iteration times when making changes to phase 2. More than likely, when you first work on defining what packages go into your Docker VM template, you'll figure out you'll need a few more, and having to run that Ubuntu installer every time is really not worth it.

Packer comes with a VMware plugin, but using it requires running it on a host with VMware Fusion for OS X, VMware Workstation for Linux and Windows, or VMware Player on Linux. We'd rather use the vSphere API, and it turns out JetBrains have created a [Packer plugin for vSphere](https://github.com/jetbrains-infra/packer-builder-vsphere) that supports both creating a new VM from an ISO, and cloning and provisioning a VM and provisioning it using e.g. Ansible.

## Creating the base VM template

Using the `vsphere-iso` value for the `type` option in the Packer config allows you to point to an ISO file stored in your vCenter and create a VM with that ISO mounted in the virtual optical drive of the VM. Now, you're likely to want to automate that install, and for that you can use a Kickstart script to answer the Ubuntu installer's questions. Using a Kickstart script will also allow you to install a few generic packages that are not specific to Docker, but can help in troubleshooting the VM template itself until you get it right.

### Providing the ISO

To boot a VM from an ISO file, you must first upload it to a datastore in your vCenter cluster. We've opted for hand-crafting the ISO due to rigid networking requirements, and as such point to a static location in our config.

We try and strive for having as few static/hardcoded/one-off components in our pipeline, and as such, another option would be to compose the ISO file from a stock Ubuntu netboot installer image and adding in any relevant customizations as part of our pipeline. In order to get that ISO file uploaded, we could use the tool `govc` (maintained by VMware in the [govmomi](https://github.com/vmware/govmomi/tree/master/govc) repo) and its `datastore.upload` command once we've crafted your ISO file. That would certainly make upgrading major Ubuntu releases easier.

The config for pointing to an ISO file is as follows, assuming you parameterize the location in the `ISO_DATASTORE` and `ISO_PATH` environment variables:

```json
{% raw %}{
	"builders": [
		{
			...
			"iso_paths": [
				"[{{ user `iso_datastore` }}] {{ user `iso_path` }}"
			]
			...
		}
	],
	"variables": {
		...
		"iso_datastore": "{{ env `ISO_DATASTORE` }}",
		"iso_path": "{{ env `ISO_PATH` }}",
		...
	}
}
{% endraw %}```

### Making a Kickstart script available to the installer

You have two options:

1. Bake the kickstart script into the ISO file, or
1. Use a neat trick

Here's the neat trick: JetBrains' Packer plugin for vSphere allows you to create and provision a *floppy disk* with select files and attach it to your VM such that the installer can reach the files on it. This way, we're able to keep our Kickstart script in the same repo as our Packer config and avoid having to run a file server to host any files we need during provisioning, thus reducing external dependencies for a successful VM template build.

Specifically, you add these keys in your Packer template file, assuming your Kickstart script file is called `ks.cfg` and your auxillary files are in a `files/` subdirectory relative to your Packer template file:

```json
{% raw %}{
	"builders": [
		{
			...
			"floppy_dirs": [
				"{{ template_dir }}/files"
			],
			"floppy_files": [
				"{{ template_dir }}/ks.cfg"
			],
			...
		}
	]
}
{% endraw %}```

### Speeding up iterations

Set the `create_snapshot` option to `true` to have a snapshot created after the VM has been successfully installed. This will allow you to use the `linked_clone` option in the next phase, which leads to *much*  faster cloning of the VM, increasing your iteration speed.

## The final config

```json
{% raw %}
{
  "builders": [
    {
      "CPU_limit": -1,
      "CPU_reservation": 0,
      "CPUs": "2",
      "RAM": "4096",
      "boot_wait": "2s",
      "cluster": "{{ user `cluster` }}",
      "convert_to_template": true,
      "create_snapshot": true,
      "datacenter": "{{ user `datacenter` }}",
      "datastore": "{{ user `datastore` }}",
      "disk_size": 51200,
      "disk_thin_provisioned": true,
      "floppy_dirs": [
        "{{ template_dir}}/files"
      ],
      "floppy_files": [
        "{{ template_dir }}/ks.cfg"
      ],
      "folder": "{{ user `folder` }}",
      "guest_os_type": "ubuntu64Guest",
      "insecure_connection": "true",
      "iso_paths": [
        "[{{ user `iso_datastore` }}] {{ user `iso_path` }}"
      ],
      "network": "/{{ user `datacenter` }}/network/{{ user `network` }}",
      "network_card": "vmxnet3",
      "password": "{{ user `vsphere_password` }}",
      "ssh_private_key_file": "{{ template_dir }}/.ssh/id_rsa",
      "ssh_username": "automationuser",
      "type": "vsphere-iso",
      "username": "{{ user `vsphere_user` }}",
      "vcenter_server": "{{ user `vsphere_server` }}",
      "vm_name": "{{ user `vm_name` }}"
    }
  ],
  "post-processors": [],
  "variables": {
    "cluster": "{{ env `CLUSTER` }}",
    "datacenter": "{{ env `DATACENTER` }}",
    "datastore": "{{ env `DATASTORE` }}",
    "folder": "{{ env `FOLDER` }}",
    "iso_datastore": "{{ env `ISO_DATASTORE` }}",
    "iso_path": "{{ env `ISO_PATH` }}",
    "network": "{{ env `NETWORK` }}",
    "vm_name": "{{ env `BASE_VM_TEMPLATE_NAME` }}",
    "vsphere_password": "{{ env `VSPHERE_PASSWORD` }}",
    "vsphere_server": "{{ env `VSPHERE_SERVER` }}",
    "vsphere_user": "{{ env `VSPHERE_USER` }}"
  }
}
{% endraw %}```
