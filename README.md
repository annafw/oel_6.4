oel_6.4
=======

Setup Oracle Enterprise Linux 6.4 Vagrant Base Box

Original steps from http://thornelabs.net/2013/11/11/create-a-centos-64-vagrant-base-box-from-scratch-using-virtualbox.html

## Prepare the CentOS 6.4 Virtual Machine

- Download the CentOS 6.4 x86_64 DVD.
- Open VirtualBox and click New.
- Give the virtual machine a Name: centos-6.4-x86_64.
- From the Type dropdown menu choose Linux.
- From the Version dropdown menu choose Red Hat (64 bit) and click Continue.
- Change RAM to 2048MB (Vagrant can change this on-the-fly later) and click Continue.
- Select Create a virtual hard drive now and click Create.
- Select VDI (VirtualBox Disk Image) and click Continue.
- Select Dynamically allocated and click Continue.
- Change the disk size to 40.00 GB and click Create.

The virtual machine definition has now been created. Click the virtual machine name and click Settings.

Go to the Storage tab, click Empty just under Controller: IDE, then on the right hand side of the window click the CD icon, and select Choose a virtual CD/DVD disk file….

Navigate to where the CentOS 6.4 x86_64 DVD ISO was downloaded, select it, and click Open.

Go to the Audio tab and uncheck Enable Audio.

Go to the Ports tab, then go to the USB tab, and uncheck Enable USB Controller.

Click Ok to close the Settings menu.

Finally, start up the virtual machine to begin installation.

## Install CentOS 6.4
Install CentOS 6.4 however you like, but I use this Kickstart Profile to automate the build (add link).

At the end of the install, restart the virtual machine.

Log in as root, and from VirtualBox insert guest additions CD.

	ifup eth0
	yum -y install gcc kernel-devel kernel-uek-devel make vim emacs
	mkdir /media/vboxadd
	mount /dev/sr0 /media/vboxadd
	sh /media/vboxadd/VBoxLinuxAdditions.run

You will see an error that the X Window System drivers aren't installed if you don't have X installed - that's OK.
Shutdown the virtual machine, and open Settings again for the virtual machine.

Go to the Storage tab, select Controller: IDE, and click the green square with red minus icon in the lower right hand corner of the Storage Tree section of the Storage tab.

Click OK to close the Settings menu.

If you used the Kickstart Profile above to automate the installation skip to the Create the Vagrant Box section.

## Configure CentOS 6.4
Once CentOS has finished installing and booted, perform the following steps to make it work with Vagrant.

Login as the root user (password is vagrant).

By default, eth0 is not brought up, so bring it up:

	ifup eth0

Enable ntpd and set the time:

	yum install ntp
	chkconfig ntpd on
	service ntpd stop
	ntpdate time.nist.gov
	service ntpd start

Enable ssh service start on boot:

	chkconfig sshd on

Disable iptables service start on boot:

	chkconfig iptables off

Set SELinux to permissive:

	sed -i -e 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config

Add vagrant user:

	useradd vagrant

Create vagrant user’s .ssh folder:

	mkdir -m 0700 -p /home/vagrant/.ssh

If you want to use your own SSH public/private key then create an SSH public/private key on your workstation (you may already have), and copy the public key to /home/vagrant/.ssh/authorized_keys on the virtual machine.

Otherwise, if you want to use the SSH public/private key provided by Vagrant, run the following command:

	curl https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub >> /home/vagrant/.ssh/authorized_keys

Change permissions on authorized_keys files to be more restrictive:

	chmod 600 /home/vagrant/.ssh/authorized_keys

Make sure vagrant user and group owns the .ssh folder and its contents:

	chown -R vagrant:vagrant /home/vagrant/.ssh

Comment out requiretty in /etc/sudoers. This change is important because it allows ssh to send remote commands using sudo. Without this change vagrant will be unable to apply changes (such as configuring additional NICs) at startup:

	sed -i 's/^.*requiretty/#Defaults requiretty/' /etc/sudoers

Allow user vagrant to use sudo without entering a password:

	echo "vagrant ALL=NOPASSWD: ALL" >> /etc/sudoers
	
Open /etc/sysconfig/network-scripts/ifcfg-eth0 and make it look exactly like the following:

	DEVICE=eth0
	TYPE=Ethernet
	ONBOOT=yes
	NM_CONTROLLED=no
	BOOTPROTO=dhcp

Remove the udev persistent net rules file:

	rm -f /etc/udev/rules.d/70-persistent-net.rules

Install some additional repository packages:

	yum install -y emacs vim man openssh-clients git wget

Clean up yum:

	yum clean all

Clean up history:

	history -c

Shutdown the virtual machine:

	shutdown -h now


## Create the Vagrant Box
Make sure the –base parameter matches the name of the virtual machine in VirtualBox:

	vagrant package --output centos-6.4-x86_64.box --base centos-6.4-x86_64

Add the Vagrant Box:

	vagrant box add centos-6.4-x86_64 centos-6.4-x86_64.box

### Create a Vagrant Project and Configure Vagrantfile
You can have as many vagrant projects as you want. Each will contain different Vagrantfiles and different virtual machines. So, create a directory somewhere to house your Vagrantfile and associative virtual machines:

	mkdir -p ~/Development/vagrant-test
	cd ~/Development/vagrant-test

Create the Vagrantfile:

	vagrant init centos-6.4-x86_64

You now have a Vagrantfile that points to the centos-6.4-x86_64 Base Box you just created.

If you are using your own SSH private/public key, and not the SSH private/public key provided by Vagrant, you need to tell Vagrantfile where to find your SSH private key, so add the following to your Vagrantfile:

	config.ssh.private_key_path = "~/.ssh/id_rsa"

Lastly, if you do not want Shared Folders setup between your workstation and virtual machine, disable it by adding the following to your Vagrantfile:

	config.vm.synced_folder ".", "/vagrant", id: "vagrant-root", disabled: true

## Start Using Vagrant
Spin up your first virtual machine:

	vagrant up

If everything spins up properly you can see the status of your virtual machine using vagrant status, you can SSH into your virtual machine using vagrant ssh, or you can destroy your virtual machine using vagrant destroy.
