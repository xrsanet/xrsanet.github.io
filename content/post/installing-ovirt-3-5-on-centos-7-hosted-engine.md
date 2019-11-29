---
title: "Installing Ovirt 3 5 on Centos 7 Hosted Engine"
date: 2015-02-04T19:30:13Z
draft: false
categories: ["Virtualization"]
---
Below are simple step by step instructions for installing the node and getting it configured for the hosted engine using CentOS 7.

# Installation Requirements

 - Installing Hosted Engine on CentOS 7 requires oVirt 3.5.1
 - Both the node and engine will be running CentOS 7 (Minimal Installation)
 - Ensure the host is fully updated via `yum update` and rebooted before proceeding

# Prerequisites

**DNS**

Ensure you have set up hostnames for the host and engine. If you do not have a DNS server configured and you are only testing oVirt on a single server, you can use /etc/hosts instead. I have the following:

Engine: Hostname: engine.xrsa.net, IP Address: 192.168.122.101/24
<br>
Host: Hostname: ovirt01.xrsa.net, IP Address: 192.168.122.100/24

**NFS**

Ensure you have set up NFS mount points for the engine and virtual machines. If you do not have a shared NFS server and you are only testing oVirt on a single server, you can configure NFS locally on the host instead as shown below.
<br><br>
*Please Note:* UID/GID of 36:36 is vdsm:kvm which is created when you install oVirt. However, you do not require this user to be created for NFS to work properly.

{{< highlight bash >}}
------------
-[ (Host) ]-
------------
# yum install -y nfs-utils
 
# mkdir /home/{engineha,vms} && chown 36:36 /home/{engineha,vms}
# cat > /etc/exports << EOF 
/home/engineha  192.168.122.0/24(rw,anonuid=36,anongid=36,all_squash)
/home/vms       192.168.122.0/24(rw,anonuid=36,anongid=36,all_squash)
EOF
#
 
# systemctl start rpcbind.service && systemctl enable rpcbind.service
# systemctl start nfs-server.service && systemctl enable nfs-server.service
{{< /highlight >}}

Verify you can see and properly mount the correct mount points.

{{< highlight bash >}}
----------
-[ Host ]- 
----------
# showmount -e ovirt01.xrsa.net
Export list for ovirt01.xrsa.net:
/home/engineha 192.168.122.0/24
/home/vms      192.168.122.0/24
# mount ovirt01.xrsa.net:/home/engineha /mnt && umount /mnt
 
###
Please note: The above mount command is to verify you can mount
/home/engineha as a test only, you do not need to mount this yourself
as oVirt will ensure its mounted when required. 
 
If you get access denied please run the below command
and rerun the mount test, if it still fails verify your NFS server.
####
 
# systemctl restart nfs-server.service
{{< /highlight >}}

# Installation

**NTP**

This not a requirement, but it is recommended that you keep your servers time in sync:

{{< highlight bash >}}
------------
-[ (Host) ]-
------------
# yum install -y ntp
# systemctl start ntpd && systemctl enable ntpd
#
 
Verify you can reach the NTP servers:
 
# ntpq -p
{{< /highlight >}}

You may put your own NTP servers in /etc/ntp.conf if required.

Once you have verified DNS and NFS, install the required repositories and packages.

{{< highlight bash >}}
------------
-[ (Host) ]-
------------
# yum -y install epel-release
# yum localinstall -y http://resources.ovirt.org/pub/yum-repo/ovirt-release35.rpm
# yum install -y ovirt-hosted-engine-setup bind-utils screen
{{< /highlight >}}

We will need an ISO for the hosted engine installation.

{{< highlight bash >}}
-----------
-[ Host ] -
-----------
# mkdir /home/tmpengineiso && cd /home/tmpengineiso
# curl -O http://mirror.ukhost4u.com/centos/7.1.1503/isos/x86_64/CentOS-7-x86_64-Minimal-1503-01.iso
# chown -R 36:36 /home/tmpengineiso
{{< /highlight >}}

*Please Note:* UID/GID of 36:36 is vdsm:kvm which is created when you install oVirt.

Now all the prerequisites are in place, verify DNS and then go through the hosted-engine wizard.

When running the wizard make sure you select the Boot type of CDROM, else you will get asked for an OVA file instead.

{{< highlight bash >}}
-----------
-[ Host ] -
-----------
# host engine.xrsa.net; host ovirt01.xrsa.net
# screen
# hosted-engine --deploy
...
--== CONFIGURATION PREVIEW ==--
         
Bridge interface                   : eth0
Engine FQDN                        : engine.xrsa.net
Bridge name                        : ovirtmgmt
SSH daemon port                    : 22
Firewall manager                   : iptables
Gateway address                    : 192.168.122.1
Host name for web application      : ovirt01.xrsa.net
Host ID                            : 1
Image alias                        : hosted_engine
Image size GB                      : 25
Storage connection                 : nfs01.xrsa.net:/home/engineha
Console type                       : vnc
Memory size MB                     : 4096
MAC address                        : 00:16:3e:71:de:6d
Boot type                          : cdrom
Number of CPUs                     : 2
ISO image (for cdrom boot)         : /home/tmpengineiso/CentOS-7-x86_64-Minimal-1503-01.iso
CPU Type                           : model_Westmere
         
Please confirm installation settings (Yes, No)[Yes]:
{{< /highlight >}}

The hosted-engine wizard will give you VNC details so you can connect to the hosted engine virtual machine and install CentOS 7.

*Please note:* You never VNC to the virtual machines themselves, you always VNC to the oVirt Host that particular virtual machine is running on from your PC/Laptop. Each running virtual machine on the host will get assigned a dedicated port which will give you Video / Keyboard via VNC.

Getting console access to your virtual machines is much easier once you get to the oVirt Interface, but as we don’t have that yet the first virtual machine running on a host will be assigned port 5900. So in order for us to connect to the hosted engine to install CentOS 7, we would connect to our host on that port as shown below:

{{< highlight bash >}}
$ vncviewer -quality 2 ovirt01.xrsa.net:5900
{{< /highlight >}}

Once installed choose option (1) on the hosted-engine wizard, it will wait until you have rebooted the hosted engine virtual machine. The wizard will give you another set of VNC details to connect to if you need it. However, if you configured networking during the install you should be able to SSH instead.

Once you have connected to the hosted engine, download the repositories, configure NTP and run through the ovirt-engine wizard. Please make sure the admin password matches up with the password set in the hosted-engine wizard.

**Note:**

> DNS must be set properly so the engine can resolve itself and the host, or the installation will fail!
<br><br>
> While in the engine setup wizard below ensure you put a proper ACL for the “NFS export ACL” option. If you do not you will not be able to activate the ISO_DOMAIN later.

{{< highlight bash >}}
--------------
-[ (Engine) ]-
--------------
# yum -y update
# yum -y install epel-release
# yum localinstall -y http://resources.ovirt.org/pub/yum-repo/ovirt-release35.rpm
# yum install -y ovirt-engine bind-utils screen ntp
# host engine.xrsa.net; host ovirt01.xrsa.net
# systemctl start ntpd && systemctl enable ntpd
# ntpq -p
# screen
# engine-setup
...
--== CONFIGURATION PREVIEW ==--
         
Application mode                        : both
Firewall manager                        : firewalld
Update Firewall                         : True
Host FQDN                               : engine.xrsa.net
Engine database name                    : engine
Engine database secured connection      : False
Engine database host                    : localhost
Engine database user name               : engine
Engine database host name validation    : False
Engine database port                    : 5432
Engine installation                     : True
NFS setup                               : True
PKI organization                        : xrsa.net
NFS mount point                         : /var/lib/exports/iso
NFS export ACL                          : 192.168.122.0/24(rw)
Configure local Engine database         : True
Set application as default page         : True
Configure Apache SSL                    : True
Configure WebSocket Proxy               : True
Engine Host FQDN                        : engine.xrsa.net
         
Please confirm installation settings (OK, Cancel) [OK]:
{{< /highlight >}}

Once finished go back to the hosted-engine wizard and finish off the installation by choosing option (1). It will ask you one final time for the hosted engine to be shutdown, wait a few minutes and it will come back up automatically.

After around a minute you can verify the state of the hosted engine virtual machine by using the following command:

{{< highlight bash >}}
-----------
-[ Host ] -
-----------
# hosted-engine --vm-status
 
--== Host 1 status ==--
 
Status up-to-date                  : True
Hostname                           : ovirt01.xrsa.net
Host ID                            : 1
Engine status                      : {"reason": "bad vm status", "health": "bad", "vm": "up", "detail": "powering up"}
Score                              : 2400
Local maintenance                  : False
...
        host-id=1
        score=2400
        maintenance=False
        state=EngineStarting
#
 
Please wait for around five / ten minutes for the hosted engine virtual
machine to come back up properly.
 
# hosted-engine --vm-status
 
--== Host 1 status ==--
 
Status up-to-date                  : True
Hostname                           : ovirt01.xrsa.net
Host ID                            : 1
Engine status                      : {"health": "good", "vm": "up", "detail": "up"}
Score                              : 2400
Local maintenance                  : False
...
        host-id=1
        score=2400
        maintenance=False
        state=EngineUp
# 
{{< /highlight >}}

# Data Domain and ISO_Domain Setup

Before you can create virtual machines in oVirt you need to create a Data Domain and ensure the ISO_DOMAIN is attached to the Default cluster.

Navigate to https://engine.xrsa.net and login with admin.

Create a new Data / NFS Domain by going to “System -> Storage -> New Domain”:

![ovirt_create_datadomain](/images/ovirt_create_datadomain.png)

You must wait until the NFS01 Data Domain is in an active state.

![ovirt_datadomain_activeng](/images/ovirt_datadomain_active.png)

Once activated attach the ISO_DOMAIN to the Default Data Center:

![iso_domain_attach](/images/iso_domain_attach.png)

*Please note:* If you are having issues attaching ISO_DOMAIN to the cluster you might have forgot to add a proper ACL on the “NFS export ACL” option during the engine wizard. You can check this as follows:

{{< highlight bash >}}
--------------
-[ (Engine) ]-
--------------
# cat /etc/exports.d/ovirt-engine-iso-domain.exports               
/var/lib/exports/iso    engine.xrsa.net(rw)
#
 
This is incorrect as the hosts are also mounting this NFS share not just the engine.
You can fix this by changing to the subnet the hosts are using.
 
# sed -i "s#engine.xrsa.net#192.168.122.0/24#" /etc/exports.d/ovirt-engine-iso-domain.exports
# cat /etc/exports.d/ovirt-engine-iso-domain.exports               
/var/lib/exports/iso    192.168.122.0/24(rw)
# systemctl restart nfs-server
{{< /highlight >}}

If everything went as expected you should see both the NFS01 and ISO_DOMAIN in an up and active state:

![iso_nfs_domain_up](/images/iso_nfs_domain_up.png)

# Uploading ISO Images

There is no GUI based ISO upload tool during this time, so to upload ISO images you must login to the engine first and run the following commands:

{{< highlight bash >}}
--------------
-[ (Engine) ]-
--------------
# curl -O http://mirror.ukhost4u.com/centos/7.1.1503/isos/x86_64/CentOS-7-x86_64-Minimal-1503-01.iso
# ovirt-iso-uploader upload -i ISO_DOMAIN CentOS-7.0-1406-x86_64-Minimal.iso
# rm CentOS-7.0-1406-x86_64-Minimal.iso
{{< /highlight >}}

# Using oVirt

At this point everything should be up and running for you to start creating virtual machines. For more information please read the oVirt Documentation: [http://www.ovirt.org/Documentation](http://www.ovirt.org/Documentation)