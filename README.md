# easy-provisioning

*Scripts for putting the words* **fun** *and* **easy** *back in provisioning...*


I'm a big fan of the DevOps attitude of "cattle" versus "pets": machines should be built in a repeatable, automated and consistent way. If there's something wrong, don't be afraid to replace a sick "cow" instead of trying to revive your "pet".
This mindset also helps when preparing for demo's and workshops: Usually I need a number of machines, and what better way than create them by using automation ? For that I'm using the tools Ansible, Packer, Vagrant and VirtualBox - they are all Open Source and can be used on a number of platforms (e.g. Windows, Linux and Mac OS X).

**Packer** creates a machine image by installing an operating system to a multitude of local and cloud platforms, for example VMWare, VirtualBox as well as Docker, Amazon EC2 and DigitalOcean. Packer is licensed under the Mozilla Public License Version 2.0.

**VirtualBox** is a virtualization environment for local use, licensed under the GNU General Public License version 2.

**Vagrant** is a tool for managing virtual machines and is licensed under the MIT license.

**Ansible** is a tool for managing systems and deploying applications, licensed under the GNU General Public License version 3 (my personal favorite).

This repository contains (example) scripts to setup an Ansible server, and Kali Vagrantbox installation, both automatically provisioned using Ansible. All commands are executed on the HOST system, unless specified otherwise.

### Prerequisites
1. Make sure that the following tools are installed, up-to-date and can be executed from the command line.

  **packer** : [https://packer.io/](https://packer.io/) (to build operating systems unattended)

  **Vagrant** : [https://www.vagrantup.com](https://www.vagrantup.com) (to configure and spin up virtual machines)

  **VirtualBox** : [https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads) (the virtualization environment)
2. Start your favorite shell (e.g. zsh or Bash), and verify the tools:

  `packer version && vagrant version && VBoxManage --version && echo "Hey Ho Let's Go"`

3. Clone this repository

  ` git clone https://github.com/PeterMosmans/easy-provisioning && pushd easy-provisioning`

### 1: Bootstrap an Ansible server
End goal: *a Vagrantbox containing Ansible, provisioned using Ansible.*

##### 1a: Build and provision Ansible on Debian
`pushd vagrant/ansible-server && VAGRANT_VAGRANTFILE=Vagrantfile.bootstrap vagrant up`

This will spin up a Debian box named `bootstrap`, install Ansible on it, and provision it for use with VirtualBox. Furthermore it will compact the box, and shut it down afterwards.

Optionally you can add your username, public SSH key and other personal variables in the file `ansible/group_vars/personal.yml`

##### 1b: Package the Ansible server
Use the following command to package it as a new Vagrantbox, and add it to the local catalog, ready to be used as Ansible server:

`VAGRANT_VAGRANTFILE=Vagrantfile.bootstrap vagrant package bootstrap --output ansible-server.box && vagrant box add ansible-server.box --name ansible-server`

Alternatively, you can add it to the local catalog versioned. For this, you need to have `openssl` installed, as it computes a checksum of the boxfile:

`VAGRANT_VAGRANTFILE=Vagrantfile.bootstrap vagrant package bootstrap --output ansible-server.box && sed -i s/CHECKSUM/$(openssl sha1 ansible-server.box|cut -d ' ' -f2)/g ansible-server.json && sed -i s/VERSION/$(date +%Y%m%d.0.0)/g ansible-server.json && vagrant box add ansible-server.json`


That's it! Now, you can spin up the Ansible server box anytime using the command
`vagrant up` in the `vagrant/ansible-server` directory. The directory `/ansible` in the repository is mapped to `/etc/ansible`, so the Ansible configuration (including roles and playbooks) persists across Vagrant 'reboots'.

You can remove the `bootstrap` box using `VAGRANT_VAGRANTFILE=Vagrantfile.bootstrap vagrant destroy`. The `ansible-server.box` can removed as well, when it has been added to the local catalog.

### 2: Install Kali Linux 2016.2 as a Vagrantbox
End goal: *A Vagrantbox containing Kali Linux 2016.2, provisioned using Ansible*

##### 2a: Install Kali
First, let Packer create a VirtualBox installation of Kali 2016.2 that can be imported. You can place the ISO image in the directory `ISO/`, or, if the correct ISO cannot be found in the ISO folder, Packer optionally downloads the latest ISO image of Kali.

`pushd packer && packer build kali-2016.2.json`

The build process will create an OVA file in the directory `output-kali` that can be directly imported into VirtualBox. Note that the root password is `toor`, the SSH server will be enabled on boot and allows root to log in: In other words, it's insecure. That's why it's important to harden it using Ansible.

##### 2b: Run Kali as VirtualBox appliance
Import the box and create a mapping so that port 22 (the SSH server) can be accessed from the Ansible server:

`MYNAME=kali-2016.2 && VBoxManage import "output-kali-2016.2/kali-2016.2.ova" --vsys 0 --vmname "$MYNAME" && VBoxManage modifyvm "$MYNAME" "--natpf1" "guestssh,tcp,,221,,22" && VBoxManage startvm "$MYNAME"`

##### 2c: Connect to Kali from the Ansible server
Spin up an Ansible box (see the first example) and check if you can connect to Kali from Ansible. Note that the server can access Kali on the mapped port (221) of **the gateway address**.
For this, add the host to `/etc/ansible/hosts` file.

First, connect to `ansible-server` by using `vagrant ssh` in

`echo "kali ansible_host=$(sudo route -n|awk '/UG/{print $2}') ansible_port=221 ansible_ssh_extra_args='-o StrictHostKeyChecking=no'">> /etc/ansible/hosts`
The `/etc/ansible` folder is mapped to the `ansible` folder of the repository and is therefore persistent across Vagrant reboots/restarts.
First make sure that the hostkey of kali is added to the vagrant server
The following command tests whether Kali can be reached from `ansible-server`:


`ansible kali -m ping -u root --ask-pass`

If this succeeds, the system is all set for provisioning.

##### 2d: Provision Kali

On `ansible-server`, install the necessary roles on Ansible:

`ansible-galaxy install -r /etc/ansible/requirements.yml`

While on `ansible-server`, run the playbook

`pushd /playbooks && ansible-playbook kali.yml -u root --ask-pass`

This will provision kali.


##### 2e: Package Kali as Vagrant box

On the host, compact the virtual machine, package it as versioned box and import it:
`MYNAME=kali-2016.2 && ssh root@127.0.0.1 -p 221 "/usr/bin/compact_box.sh" ; vagrant package --base ${NAME} --output ${MYNAME}.box && sed -i s/CHECKSUM/$(openssl sha1 ${MYNAME}.box|cut -d ' ' -f2)/g ${MYNAME}.json && sed -i s/VERSION/$(date +%Y%m%d.0.0)/g ${MYNAME}.json && vagrant box add ${MYNAME}.json`

That's it! Now, you can spin up the Ansible server box anytime using the command
`vagrant up` in the `vagrant/kali` directory (similar to the Ansible server).

You can remove the `.box` files: as soon as they are added to Vagrant they are stored 'internally', on a different location.

# Packer templates and preseed files
## Kali 2016.2
The template `kali-2016.2` will install the latest version of Kali 2016.2, and prepare it for use with other provisioning tools. It uses the `kali-packer.cfg` preseed file, which is the vanilla preseed file for Kali 2016.2, with a minimal addition in order to use it with Packer.


