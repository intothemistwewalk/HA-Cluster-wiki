# This guide is for RHEL/CentOS machines only

## What is a DRBD Cluster?
A DRBD cluster (Distributed Replicated Block Device) is a cluster. A cluster being an instance of more than one server to achieve the same objective.
In a case where real-time synchronized storage is required between block device storage there is the existence of DRBD. This works in the format of having two nodes which require a real-time rate of high availabilty. DRBD is mostly used within data center disaster recovery situations. The DRBD component allows two nodes to be synced meaning if one node within a data center location was destroyed, then with regards to DRBD; the data which is managed is still accessible by one of the remaining nodes thanks to the presence of DRBD.

### Resources to consult:
https://www.learnitguide.net/2016/07/how-to-install-and-configure-drbd-on-linux.html
https://unix.stackexchange.com/questions/29999/why-are-my-two-virtual-machines-getting-the-same-ip-address

# Troubleshooting network issues in Virtualbox

Virtualbox is known to not share it's IP addresses and instead allocates a default IP of 10.0.2.15, a quick fix if you're using CentOS machines is to go into file -> preferences -> Network. From this point you need to select a NatNetwork, going forward you should then go into the settings of the VBOX machine and assign the machine to a NatNetwork. (Do this with both machines)
Once the machine is configured you should then be able to have an individual IP address for each machine involved 

#### Useful commands in CentOS7 glossary
ip addr show - Will show the IP address, ifconfig is not the default command

ssh username@ip-address (Format of accessing the machines via SSH)

## Configuration and Installation of DRBD Cluster
1. Install required packages and update both nodes. 
	sudo yum -y update 
	
	yum -y install gcc make automake autoconf libxslt libxslt-devel flex rpm-build kernel-devel
	
	rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
	
	rpm -Uvh
2. Install DRBD repository and import GPG key  
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
3. Install drbd
yum install drbd90-utils kmod-drbd90

Ensuring the modules at boot, ensure you can access your root account and enter the following command: echo drbd > /etc/modules-load.d/drbd.conf 

(Do this for both nodes)
A seperate disk will be needed for DRDB as it is an entire block device which is what we need
In virtual box do the following:
1. Power off the VM 
2. Access the storage section and add a new hard disk titled by /dev/sdb for clarity
	We have added a second disk which will act as the block device for the DRDB to be configured upon
	
3. Once the installation is done issue the next command to ensure the configuration is picked up by the system kernel: lsmod | grep -i drbd

4. If that does not work and nothing is found then issue the following command: sudo find / -name drbd.ko
	The use of this command is to search the entire filesystem from root directory level to locate the location of the drbd.ko file 	which is the kernel file associated to drbd.ko. Once the file is found, use the command insmod to add the appropriate kernel 		files.
	
5. Once the first four steps are done it's now time for the steps of configuration. Change your directory to /etc and you should find drbd.conf, this is the sample configuration for you to fill in; in addition there is a directory called drbd.d. At this point you should now copy the drbd.conf file into the drbd.d directory.

6. Open the file global_common.conf, create a backup of this file before editing it. And then insert the following into the file
global {
  usage-count yes;
}
common {
  net {
    protocol C;
  }
}

#global_common.conf file troubleshooting
Ensure the file path is exact to your files such as "/etc/drbd.d/global_common.conf" do not have a file path which is incomplete as this will result in a failure to load the file up 

Protocol C is one of three protocols which DRBD can use. In our case the use of protocol C is important because it is asynchronous data transfer.

7. Open the .res file which can have any name of your choice on the start of it for example: name.res. You should then add the following data:

resource (name of resource) {
device (name of device);
disk (name of disk used);
meta-disk internal;
protocol C;
	on (hostname of your first VM) {
	address: (YOUR IP ADDRESS);
	meta-disk internal;
	}
	on (hostname of your first VM) {
	address: (YOUR IP ADDRESS);
	meta-disk internal;
	}
}

8. Once the configuration is done and all troubleshooting is finished, it is then time to create the metadata for the associated device before you can start the drbd service using the following command: (Use sudo if you're not in root) (This must be done on both nodes)

sudo drbdaddm create-md (name of resource) - in my case it will be drbd0 

It is then time process the command to start the service again on BOTH NODES: sudo systemctl start drbd and to ensure that the access to drbd will be constant after reboot ensure you enter: sudo systemctl enable drbd. This will create a symbolic link for you to access the drbd service easier. 

9. Now that everything has been created and started it's now time to setup your primary server. Enter the following command to firstly ensure that the status of your drbd is active: drbdadm up (name of resource)

10. Add the firewall rules to ensure the port number is open for your two nodes to communicate over the network: 

### firewall rule: 
firewall-cmd --permanent --add-rich-rules='rule family="ipv4" source address ="(Your IP address) port port="7788" protocol="tcp" accept

# BEWARE
Your DRBD process will not remain in the next boot-up if you do not add the following:
 echo drbd > /etc/modules-load.d/drbd.conf
 
Ensure this line is added so that you're able to keep your kernel file included every time you boot up, if you don't include the above command you will have issues to keep the drbd in a constant working state. 

After every set of changes make sure to update the drbd service

If resource is open and you can't reset metadata to reconfigure the VMs then detach the block device using the command drbdadm detach (name of server)

#### If the server has another volume occupied and you want to reconfigure to assign as primary, do the following for a clean restart:

1. Detach the server and bring it down using drbdadm

2. Create the metadata for the block device e.g. drbdadm create-md drbd0

3. Bring the block device up using drbd

4. Assign as primary. If there's a fault then use the --force switch

# IMPORTANT - FOR USERS OF VERSION 9
The /proc/drbd file has been dropped in the DRBD 9.x releases. You should instead use drbdadm status to get the current state of the DRBD resources.

If you really insist on the old /proc/drbd output you can find this within the sysfs now. Note that this is per connection only. It is located at:

/sys/kernel/debug/drbd/resources/${resource_name}/connections/${hostname}/0/proc_drb

### Final manual failover test 

1. Mount the block device on the primary node on the mount table.
	Make a filesystem for the block device if you have not done so yet. 

2. List the contents of the /mnt table (It should be the contents of the block device)

3. Unmount from the current primary and change the current primary device into a secondary.

4. Change the previous secondary device into a primary and mount the contents of the drbd block device.

If all is successful then you have successfully implemented drbd with a manual failover.

Reference from: https://unix.stackexchange.com/questions/441313/drbd-no-output-of-cat-proc-drbd
## Resources used
http://prolinuxhub.com/building-simple-drbd-cluster-on-linux-centos-6-5/

https://www.atlantic.net/hipaa-compliant-database-hosting/how-to-configure-lvm-drbd/

https://www.digitalocean.com/community/tutorials/how-to-use-lvm-to-manage-storage-devices-on-ubuntu-16-04

https://www.techrepublic.com/article/how-to-add-new-drives-to-a-virtualbox-virtual-machine/

https://serverstack.wordpress.com/2017/05/31/install-and-configure-drbd-cluster-on-rhel7-centos7/

https://docs.linbit.com/man/v9/drbd-conf-5/

https://linuxhandbook.com/install-drbd-linux/

http://yallalabs.com/linux/how-to-install-and-configure-drbd-cluster-on-rhel7-centos7/

https://www.quora.com/What-is-a-Linux-block-device

https://www.youtube.com/watch?time_continue=337&v=oGIpo1SoSdY

https://serverfault.com/questions/452693/drbd-status-uptodate-diskless

https://lists.gt.net/drbd/users/27942

https://serverfault.com/questions/804217/drbd-on-raw-disk-block-device

https://hk.saowen.com/a/4863a03aade28300f5646ef42eb1c04e7ec107620941030757dbfb5a6c8ca541

https://serverfault.com/questions/870213/how-to-get-drbd-nodes-out-of-connection-state-standalone-and-wfconnection

https://stackoverflow.com/questions/45998076/an-explanation-of-drbd-protocol-c?rq=1
#### Troubleshooting connections:

Since there are ports involved, check the port availability and if the port is closed then allow it through the firewall

#### To further understand LVMs consult this guide:
https://www.howtoforge.com/linux_lvm

# Adding pacemaker into HA cluster
Pacemaker is a cluster resource manager used within HA, pacemaker uses scripts and health checks to monitor the progress and health of the nodes which are used. 

Pacemaker does not work by itself, it works alongside other services such as corosync which is the messaging service that works between nodes. 

1. Install the required cluster software and tools:

yum install -y pacemaker pcs psmisc policycoreutils-python

(If applicable to your setup then disable the firewall)

2. systemctl start pcsd.service && systemctl enable pcsd.service - This will allow the nodes including the pcs daemon to start up on boot. The use of this daemon works with the pcs cli to synchronize the corosync config across all nodes 

3. Login to the machines using the command: pcs cluster auth (node 1 name) (node 2 name)
The corosync creates a default account called hacluster, ensure to setup a password for it before trying to login to anything. 

4. You will be prompted

### Troubleshooting
Check if the associated process is running using: ps -ef grep pscd

You should see output of a ruby process. 

If your login brings out errors then use the --debug option to be able to see the problem. From my experience of a connection failure was due to a lack of connectivity. The hostnames entered could not see each other due to lack of entries in /etc/hosts. 


### References
https://www.suse.com/documentation/sle-ha-12/singlehtml/book_sleha_techguides/book_sleha_techguides.html#sec_ha_quick_nfs_usagescenario

http://jensd.be/186/linux/use-drbd-in-a-cluster-with-corosync-and-pacemaker-on-centos-7

https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/2.0/pdf/Clusters_from_Scratch/Pacemaker-2.0-Clusters_from_Scratch-en-US.pdf
