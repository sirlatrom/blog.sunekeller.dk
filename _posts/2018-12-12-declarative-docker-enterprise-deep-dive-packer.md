---
title: 'Deep Dive: Using Packer and Ansible to create a golden VMware image for Docker Enterprise - part 1'
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

### Kickstart

#### Configuring your install using a Kickstart script

The detailed list of things we configure using in our Kickstart script is as follows:

* System language: `lang en_US`
* Language modules to install (in our case: Danish): `langsupport da_DK`
* System keyboard (Danish layout): `keyboard dk`
* Timezone (`Europe/Copenhagen`): `timezone --utc Europe/Copenhagen`
* Disable password login for `root`: `rootpw --disabled`
* Initial username and encrypted password (provided as an environment variable by GitLab, substituted in before the Kickstart config file is put on the virtual floppy disk): `user automationuser --fullname "Automation user" --iscrypted --password ${AUTOMATION_USER_CRYPTED_PASSWORD}`
* Install instead of upgrade: `install`
* Use text mode install: `text`
* Reboot after install: `reboot`
* Use web installation: `url --url http://dk.archive.ubuntu.com/ubuntu/`
* Set hardware clock to UTC: `preseed clock-setup/utc boolean true`
* Set preseed time zone: `preseed time/zone string Europe/Copenhagen`
* Set NTP server: `preseed clock-setup/ntp-server {%raw%}<redacted>{%endraw%}`
* Use MBR bootloader: `bootloader --location=mbr`
* Zero out the MBR: `zerombr yes`
* Several options for partitioning:

```
preseed partman-auto/disk string /dev/sda
preseed partman-auto/method string lvm
preseed partman-lvm/device_remove_lvm boolean true
preseed partman-md/device_remove_md boolean true
preseed partman-lvm/confirm boolean true
preseed partman-lvm/confirm_nooverwrite boolean true
preseed partman-auto/choose_recipe select atomic
preseed partman-partitioning/confirm_write_new_label boolean true
preseed partman/choose_partition select finish
preseed partman/confirm boolean true
preseed partman/confirm_nooverwrite boolean true
```

* Enable shadow file and password hashing: `auth --useshadow --enablemd5`
* Disable firewall for Docker compatibility and because we run an external one: `firewall --disabled`
* Skip configuring the X Window System: `skipx`

Then follows the list of packages we install initially (some are redacted out):

```
%packages
build-essential
curl
dnsmasq
dnsmasq-base
dnsmasq-utils
dnsutils
htop
man
nfs-common
ntp
open-vm-tools
rng-tools
software-properties-common
ssh
unzip
vim
wget
curl
ca-certificates
```

#### The Kickstart post script

The last touch is the post script, which will run after the installation is complete, but noteably before the automation user has been added. That's the reason why we create the `rc.install` script, which will run once the installation is done and the VM has rebooted. It copies the SSH public key of the automation user such that tools in the later phases (both Packer and Ansible) can SSH in as the automation user.

```sh
%post --nochroot
touch /target/etc/installdate

umask 077
mkdir -p /media
mount /dev/fd0 /media
cp /media/files/ntp.conf /target/etc/ntp.conf
mkdir -p /target/usr/local/share/ca-certificates/
cp /media/files/almbrand-corporate-pki-root-ca.crt /target/usr/local/share/ca-certificates/almbrand-corporate-pki-root-ca.crt

#### Configure sudoers
cat > /target/etc/sudoers.d/defaults << EOF
Defaults  secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults        env_keep="http_proxy https_proxy"
EOF
cat << EOF > /target/etc/sudoers.d/automationuser_nopasswd
automationuser ALL=(ALL) NOPASSWD:ALL
EOF

#### Keep a safe copy /etc/rc.local for later
mv /target/etc/rc.local /target/etc/rc.local.dist

cat > /target/etc/rc.local << EOF
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

if [ -x /etc/rc.install ]
then
    /etc/rc.install && mv /etc/rc.install /etc/rc.install.1
else
    echo /etc/rc.install does not exist >> /root/rc.install.log
fi

exit 0
EOF

chmod +x /target/etc/rc.local

#### Create /etc/rc.install ####
cat > /target/etc/rc.install << EOF
#!/bin/bash

# rc.install
#

#### automationuser SSH public key
mkdir -p /home/automationuser/.ssh
chmod 0700 /home/automationuser/.ssh
chmod 0600 /home/automationuser/.ssh/authorized_keys
mount /dev/fd0 /media
cp /media/files/authorized_keys /home/automationuser/.ssh/authorized_keys
umount /media
chown -R automationuser. /home/automationuser/.ssh

#### Corporate PKI Root CA certificate
(
  update-ca-certificates
) 2>&1 | tee /tmp/certinst.log
EOF

chmod +x /target/etc/rc.install
```

#### Making a Kickstart script available to the installer

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

Another aspect of keeping iterations fast is to avoid installing *too* many packages in this first phase, since the cost in terms of waiting time is larger than in the next phase.

## SSH access to the VM template during provisioning

This step is not really needed as long as you're _only_ using Kickstart to run your provisioning, but you may easily end up in a situation where more advanced provisioning will require a tool such as Ansible, or to execute commands on the created VM template before it is considered done. For such purposes (and, indeed, for the configuration to be accepted by the vSphere builder plugin), you have to tell what SSH user and private key file will be used to access the VM once provisioned. In our case, we make sure to put that user's _public_ key file on the VM as described in [The Kickstart post script](#the-kickstart-post-script) section above, specifically these lines:

```bash
mount /dev/fd0 /media
cp /media/files/authorized_keys /home/automationuser/.ssh/authorized_keys
umount /media
```

You'll want to set the following options and make sure your SSH private key is accessible from `.ssh/id_rsa` inside your pipeline's working directory (or, if you're running this from a laptop, whichever directory you're running it from then). You can modify the `"ssh_private_key_file"` and `"ssh_username"` values to suit your needs.

```jinja
json
{% raw %}{
  "builders": [
    {
      ...
      "ssh_private_key_file": "{{ template_dir }}/.ssh/id_rsa",
      "ssh_username": "automationuser",
      ...
    }
  ]
}
{% endraw %}```

## The final config

As you can probably see from the config, we parameterize almost all the values. This is because we have multiple vCenters, and we need to build the templates in all of them. In our repo, this file is saved as `from-iso.json`.

```jinja
{% raw %}{
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
      "iso_paths": [
        "[{{ user `iso_datastore` }}] {{ user `iso_path` }}"
      ],
      "network": "/{{ user `datacenter` }}/network/{{ user `network` }}",
      "network_card": "vmxnet3",
      "password": "{{ user `vsphere_password` }}",
      "resource_pool": "{{ user `resource_pool` }}",
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
    "resource_pool": "{{ env `RESOURCE_POOL` }}",
    "vm_name": "{{ env `BASE_VM_TEMPLATE_NAME` }}",
    "vsphere_password": "{{ env `VSPHERE_PASSWORD` }}",
    "vsphere_server": "{{ env `VSPHERE_SERVER` }}",
    "vsphere_user": "{{ env `VSPHERE_USER` }}"
  }
}
{% endraw %}```

# Creating the Docker VM template

We perform the VM template creation from within our GitLab CI pipeline, but there isn't much to it other than setting the relevant environment variables (all listed in the `"variables"` key in the config above). In order to avoid having to install Packer on our GitLab runner, we run Packer from a Docker image thusly:

```console
{% raw %}$ docker run \
  --rm \
  --volume ${PWD}:/data --workdir /data \
  --tmpfs /tmp \
  --env BASE_VM_TEMPLATE_NAME \
  --env CLUSTER \
  --env DATACENTER \
  --env DATASTORE \
  --env FOLDER \
  --env ISO_DATASTORE \
  --env ISO_PATH \
  --env NETWORK \
  --env VSPHERE_SERVER \
  --env VSPHERE_USER \
  --env VSPHERE_PASSWORD \
  $packer_image \
  build from-iso.json
{% endraw %}```

As you can see, the variables are all defined outside of the command, meaning Docker will take the values of the variables in the environment and pass them on to the container.

Additionally, we make sure to make the working directory available to Packer by mounting the current working directory on `/data` inside the container and set it as the working directory of the container with `--workdir /data`. 

Another important detail is that `$packer_image` bit - we build our own Packer image for Docker in order to add a few more tools in there, namely `govc` and `jq`, as well as the two Packer vSphere builders released by JetBrains. In fact, since the next phase will use Ansible to provision Docker Enterprise on the next template, we base our final image off of [William Yeh](https://github.com/William-Yeh)'s [Ansible image on Docker Hub](https://hub.docker.com/r/williamyeh/ansible/). The Dockerfile for that image looks like this:

```dockerfile
{% raw %}ARG         packer_version=1.3.3@sha256:e65fb210abc027b7d66187d34eb095fffa2fd8401e7032196f760d7866c6484c
FROM        hashicorp/packer:${packer_version} AS packer

FROM        williamyeh/ansible:alpine3@sha256:8072eb5536523728d4e4adc5e75af314c5dc3989e3160ec4f347fc0155175ddf

# Copy in corporate certificates
COPY        *.crt /usr/local/share/ca-certificates/
RUN         update-ca-certificates

# Add utilities used inside later Packer builds
RUN         apk add --update bash jq wget curl
COPY --from=packer /bin/packer /bin/packer

# Download the Packer vSphere builders from the JetBrains infra repo
ARG         vsphere_packer_builders_version=2.1
ADD         https://github.com/jetbrains-infra/packer-builder-vsphere/releases/download/${vsphere_packer_builders_version}/packer-builder-vsphere-clone.linux /bin/packer-builder-vsphere-clone.linux
ADD         https://github.com/jetbrains-infra/packer-builder-vsphere/releases/download/${vsphere_packer_builders_version}/packer-builder-vsphere-iso.linux /bin/packer-builder-vsphere-iso.linux

# Install GOVC
ARG         govmomi_version=v0.19.0
ADD         https://github.com/vmware/govmomi/releases/download/${govmomi_version}/govc_linux_386.gz /bin/govc.gz
RUN         cd /bin && gunzip govc.gz && chmod +x /bin/packer-builder-vsphere-iso.linux /bin/packer-builder-vsphere-clone.linux govc

# Add a default ansible.cfg
COPY        ansible.cfg /etc/ansible/ansible.cfg

# Add an automation user
RUN         adduser -u 1000 -D automationuser
USER        automationuser

# Set the entrypoint to run Packer by default
ENTRYPOINT  [ "/bin/packer" ]
{% endraw %}```

You then build and push that image to your registry (Docker Hub, DTR or whichever you prefer) and use that to run Packer.

# Conclusion

Building a golden image VM template with Packer and vSphere is perfectly doable, the tools are all there, readily available and well documented.

The next part of this deep dive will deal with actually installing Docker Enterprise in a subsequent VM template.