# Kwinble - Infrastracture as a code project based on ansible and kiwi

**NEWS:  Added support for opensuse Leap 15.3 and SLES 15 SP3!**

This is a project born with the idea of installing VMs in an automated fashion for personal labs use at work on libvirt/KVM platforms, it's experimental and developed with a "learn by doing" attitude.

I am aware that there are excellent tools out there like [terraform]( https://www.terraform.io/) or [pulumi](https://www.pulumi.com/) for instance, which are probably more suited for daily enterprise usage, however, since I am familiar with ansible and kiwi I have decided to use these in order to keep the stack as simple as possible with a minimum amount of dependencies.

The project is constantly under development and subject to changes, the support for Azure is in the pipeline and under evaluation to be added in the future in a form of an Ansible role

Contributors are warmly welcomed :)

## Overview

As mentioned, Kwinble is mainly composed by 2 tools:

- [Kiwi-ng](https://osinside.github.io/kiwi/) a powerful open source appliance builder capable of creating Linux images in a variety of formats such as Qcow2, raw, vmdk/vmx, just to name a few
- [Ansible](https://docs.ansible.com) a popular open source configuration managemenet tool appreciated for its simplicity and its modular design which allows the intergation with tons of technologies from public cloud to baremetal and all in between.

The project consists of a 2 steps process:

1) the build of a GNU/Linux OS image from scratch using kiwi
2) the boot and the automated configuration of 1 or multiple instances using preconfigured Ansible roles 

Once an image has been created can be reused for other Ansible executions, does not have to be built every time.

The idea is to give the user full flexibility and control from the build stage, where he/she can decide how the initial image should look like and what should contain, to the actual final running VM.

Kwinble is essentially a GNU/Linux factory inside the project folder.


## Requirements

I am using openSUSE Leap 15.2 here so the steps shown are referring to this install, however, the same software can be retrieved and easily installed in a similar way using other GNU/Linux distributions.

### Kiwi

kiwi version used for this project: 9.23.49

Kiwi installed by adding the kiwi repo to leap 15.2:

`sudo zypper ar https://download.opensuse.org/repositories/Virtualization:/Appliances:/Builder/openSUSE_Leap_15.2/ kiwi-leap152`

`sudo zypper ref && sudo zypper in python3-kiwi`

### Ansible

A recent Ansible LTSS version:

ansible => 2.9.21 (preferable the one i use)

In my case the one provided by the leap 15.2 repo has been used

`sudo zypper in ansible`

### Libvirt/KVM

on Leap 15.2 install KVM patterns:

`sudo zypper install -t pattern kvm_server kvm_tools`


### Other required config

- Arp-scan is used for IP discovery and is going to be installed automatically if not present by Ansible during the first run.

- Make sure the local user is added to the libvirt group
`usermod -aG libvirt <username>`

- Configure your local or global `ansible.cfg` to skip the ssh fingerprint host check:

`host_key_checking = False`


### Clone the repo:

`git clone https://github.com/gnuninu/kwinble`

`cd kwinble`

## Prepare the image build with kiwi

The aim here is to build QCOW images with a small footprint so Ansible can take over, boot the image and finish the OS/App configuration according to the selected ansible roles defined in the main playbooks.

Kiwi is a very powerful tool, for full info visit kiwi [website](https://opensuse.github.io/kiwi/) and read the detailed [documentation](https://osinside.github.io/kiwi/), the following steps are only meant for a quick start and by no means as a replacement of the official documentation.

### openSUSE Leap 15.2 build

NOTE: The KIWI templates used to build Leap are fetched from openSUSE [JeOS project](https://en.opensuse.org/Portal:JeOS) and then customised: https://build.opensuse.org/package/view_file/openSUSE:Leap:15.2:Images/kiwi-templates-JeOS/JeOS.kiwi?expand=1

`cd jeos_template/opensuse-leap-152`

Edit `JeOS.kiwi` and `config.sh` as needed if you want to modify the default settings

### SLES 15 SP2 build

The package `kiwi-templates-JeOS` provides the templates for SLES and it's availbale in the `sle-module-development-tools` channel module

In order to build SUSE Enterprise Linux a subscription is necessary to access the online repositories through [SCC](https://scc.suse.com/login) and to be able to download a media ISO

Once a media is retrieved, it can be mounted locally, for example:

`mkdir -p /mnt/sles-iso/sles153`

`sudo mount -o loop SLE-15-SP3-Full-x86_64-GM-Media1.iso /mnt/sles-iso/sles153`

Enable the installation server (HTTP) keeping the default settings using yast:

`yast instserver`

Name: `sles153`

Directory: `/srv/install`

then create a symbolic link to the local installation server directory:

`sudo ln -s /mnt/sles-iso/sles153 /srv/install/`

Disable the firewall or make sure `localhost:80` port is open

The resulting locally served directory can be configured in the repository section of kiwi in this way:

`    <repository type="rpm-md" alias="product-sles153" imageinclude="true">
            <source path='http://127.0.0.1/sles153/Product-SLES'/>
     </repository>`

`cd jeos_template/sles153/`

Edit `JeOS.kiwi` and `config.sh` as needed if you want to modify the default config

## Build the image

### openSUSE Leap 15.3

`cd jeos_template/opensuse-leap-152`

`sudo kiwi-ng --profile=kvm-and-xen system build --description . --target-dir .`

### SLES 15 SP3

`cd jeos_template/sles153/`

`sudo kiwi-ng --profile=kvm-and-xen system build --description . --target-dir .`


## Deploy the VMS with Ansible on Libvirt


### Overview

The type of installations are controlled by a variable that needs to be passed to ansible.

The following types are available and mapped to the dictionaries predefined in the libvirt role here: `roles/libvirt/defaults/main.yml`

|   os_install var    | Description |
| --------------------|------------ |
|**sles152multi**     |3 sles 15 SP2 instances
|**sles152**          |1 sles 15 SP2 instances
|**sles153multi**     |3 sles 15 SP3 instances
|**sles153**          |1 sles 15 SP3 instances
|**leap152multi**     | 2 leap 15.2 instances
|**leap152**          | 1 leap 15.2 instance
|**leap153multi**     | 2 leap 15.3 instances
|**leap153**          | 1 leap 15.3 instance

The number of instances can be added or removed by modifying the yaml file and/or reused as templates

IMPORTANT the images will be installed in the **default** storage pool which is configured by default to use the path `/var/lib/libvirt/images` and the NICS will be assigned to the **default** KVM network configured with NAT (192.168.122.0/24).

The libvirt templates are stored in `roles/libvirt/templates/` and they can be customised as needed (for example if you want to add multiple NICs).

### Deployment

NOTE: Before deploying SLES make sure the variable `SCC_RC` is populated with your Registation Code available from SUSE Customer Center, do so by editing the file `variables.yml`

Move into the kwinble root folder.

To deploy on libvirt/kvm we need to populate the `os_install` varibale using the `--extra-vars` option, in this case we choose leap152 (1 Instance):

`ansible-playbook create_kvm_vms.yml --extra-vars "os_install=leap153" -K -k -u $USER`

when prompted type the root password set previously by kiwi during the image build which by default is: **linux**, then your user sudo password

To remove the VMs installation run ansible using the same os_install variable but this time invoking `remove_kvm_vms.yml` playbook:

`ansible-playbook remove_kvm_vms.yml --extra-vars "os_install=leap153" -K -u $USER`

Local storage and static IP DHCP mappings will be deleted as well


## Design diagram and notes about this project


## To do list for future PRs

- users custom role
- Uyuni/SUMA role (multi disks)
- HA Clusters role (multi disks)
- Azure support
- other Linux distro support (Debian, CentOS/Almalinux...)


**References:**
- [Kiwi documentation](https://osinside.github.io/kiwi/)
- [JeOS Project](https://en.opensuse.org/Portal:JeOS) 
- [Ansible docs](https://docs.ansible.com/ansible/2.9/roadmap/ROADMAP_2_9.html)
- [Ansible libvirt modules community project](https://github.com/ansible-collections/community.libvirt)
- [Libvirt docs](https://libvirt.org/docs.html)

