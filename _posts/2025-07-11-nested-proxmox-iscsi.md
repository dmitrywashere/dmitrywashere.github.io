---

layout: post

title: "Nested Proxmox VE iSCSI Configuration with Pure Storage FlashArray"

date: 2025-07-11

categories: [Proxmox, PureStorage]

---




![](/assets/images/nested-proxmox/image14.png)





































Nested Proxmox VE iSCSI Configuration with Pure Storage FlashArray









# 

# Purpose



The purpose of this article is to detail the steps of the iSCSI configuration of the Proxmox VE instance nested on Broadcom VMware ESXi for home lab or testing environment.

# Executive Summary

In response to the significant licensing changes introduced by Broadcom following its acquisition of VMware, many customers are actively exploring alternatives to VMware for their virtualization infrastructure. Proxmox VE has emerged as a compelling open-source option due to its flexibility, reliability, and growing community support.



However, not all organizations have spare hardware available to evaluate Proxmox in a production-like environment. This article provides a practical guide for running nested Proxmox VE inside a VMware environment, enabling customers to test and validate Proxmox without disrupting their current infrastructure.



The guide walks through the process of deploying Proxmox as a nested VM and connecting it to a Pure Storage FlashArray using in-guest iSCSI with multipath support. This setup simulates a real-world configuration, allowing teams to evaluate Proxmox functionality and storage integration capabilities with enterprise-grade all-flash storage.



Whether you're a storage architect, virtualization admin, or infrastructure decision-maker, this article offers a clear path to hands-on testing of Proxmox VE with Pure Storage—helping accelerate informed decisions around next-generation datacenter platforms.









# Prepare the nested Proxmox VE Server on VMware



There are good instructions on how to install Proxmox VE VM nested on VMware



https://www.virtualizationhowto.com/2022/01/nested-proxmox-vmware-installation-in-esxi/



If you are interested in my specific configuration, here it is





![](/assets/images/nested-proxmox/image20.png)



# 

# Update the Debian repositories for unlicensed Proxmox VE



- Login to Proxmox VE Instance with root privileges





![](/assets/images/nested-proxmox/image3.png)



- Ensure that nightly package updates were installed successfully





![](/assets/images/nested-proxmox/image15.png)



If the updates failed, you will need to add no-subscription repositories by editing the sources.list

nano /etc/apt/sources.list



![](/assets/images/nested-proxmox/image12.png)

It should look like this:



![](/assets/images/nested-proxmox/image27.png)



# 

# Network Interfaces configuration for nested Proxmox

You will need to have 2 additional network interfaces on the nested VM to configure in-guest iSCSI

On VMware side go to the VM, right-click and go to Edit Settings:



![](/assets/images/nested-proxmox/image16.png)



![](/assets/images/nested-proxmox/image6.png)

On the Proxmox VE server:

List all network interfaces:

ip link show



![](/assets/images/nested-proxmox/image7.png)



# Configure IPv4 addresses

Using Proxmox Web GUI

Go to Datacenter → [Your Node] → System → Network

Click Create → Linux Bridge (or Linux Bond if bonding)

Configure the bridge with your desired IP settings

Apply the configuration



![](/assets/images/nested-proxmox/image24.png)



Check that it is configured correctly:



![](/assets/images/nested-proxmox/image19.png)

Remember to hit the “Apply Configuration” button before proceeding.



# Setting MTU to 9000

In /etc/network/interfaces:



nano/etc/network/interfaces

Addmtu 9000to each interface:



![](/assets/images/nested-proxmox/image9.png)

Apply changes:

systemctl restart networking

Verify MTU:



![](/assets/images/nested-proxmox/image2.png)

Install and Configure iSCSI Initiator

Install required packages:

aptinstallopen-iscsimultipath-toolslsscsi sg3-utils



# Configure iSCSI initiator:

nano/etc/iscsi/iscsid.conf

Recommended settings for Pure Storage:

# Authentication (if using CHAP)node.session.auth.authmethod = CHAPnode.session.auth.username = your_usernamenode.session.auth.password = your_password# Timeoutsnode.session.timeo.replacement_timeout =120node.conn[0].timeo.login_timeout =15node.conn[0].timeo.logout_timeout =15node.conn[0].timeo.noop_out_interval =5node.conn[0].timeo.noop_out_timeout =5# Session settingsnode.session.iscsi.InitialR2T = Nonode.session.iscsi.ImmediateData = Yesnode.session.iscsi.FirstBurstLength =262144node.session.iscsi.MaxBurstLength =16776192node.conn[0].iscsi.MaxRecvDataSegmentLength =262144





































Find out the IQN that is configured in Proxmox. In the shell run the following command:



cat /etc/iscsi/initiatorname.iscsi



![](/assets/images/nested-proxmox/image25.png)

# Configure Multipath Tools

Edit multipath configuration:

nano/etc/multipath.conf

By default this file doesn’t exist, so you need to create it.

Pure Storage optimized configuration:

defaults{user_friendly_namesyesfind_multipathsyes}devices {device{vendor"PURE"product"FlashArray"path_grouping_policy"group_by_prio"path_selector"round-robin 0"path_checker"tur"hardware_handler"1 alua"prio"alua"failback immediaterr_weight uniformno_path_retry18fast_io_fail_tmo10}}









































Enable and start services:

systemctlenableiscsidsystemctlenableopen-iscsisystemctlenablemultipath-toolssystemctl start iscsidsystemctl start open-iscsisystemctl start multipath-tools













## Expected output:



![](/assets/images/nested-proxmox/image23.png)



# Pure Storage Array Configuration for Host and Volumes



On Pure Storage Array - Configure host and connect volume(s)

On the Pure Storage FlashArray, create a host object with the IQN that you just discovered:

Use this IQN to create host objects in the Pure Storage Purity management interface:

- Go to Storage > Hosts.

- Click the + icon.

- In the pop-up window, give your host a name and under personality selectnone

- Add the Proxmox host IQN under the Host Ports section.



![](/assets/images/nested-proxmox/image1.png)

Create a new volume and connect it to the Proxmox host you just created

Create a volume/LUN for your host:

- Go to Storage > Volumes.

- Click the + icon.

- In the pop-up window, give your volume a name, volume size and add your newly created hosts to this volume/LUN.



![](/assets/images/nested-proxmox/image17.png)







![](/assets/images/nested-proxmox/image5.png)



## Connect to Pure Storage Target

### Discover targets:



On Pure Storage find what are your iSCSI target IP Addresses:





![](/assets/images/nested-proxmox/image4.png)



There  is a known limitation with open-iscsi. You can't bind both hwaddress and net_ifacename on the same interface. You need to choose one binding method. I chose the net_ifacename:

Use net_ifacename (replace with your ens interface ID)

#Deleteand recreate with onlyinterfacenameiscsiadm -m iface -I ens224 -odeleteiscsiadm -m iface -I ens224 -onewiscsiadm -m iface -I ens224 -o update -n iface.net_ifacename -v ens224#Dothe sameforyour secondinterfaceiscsiadm -m iface -I ens256 -odeleteiscsiadm -m iface -I ens256 -onewiscsiadm -m iface -I ens256 -o update -n iface.net_ifacename -v ens256



















Verify the configuration:

iscsiadm -m iface -I ens224 -oshowiscsiadm -m iface -I ens256 -oshow





Now proceed with discovery:

Note that 172.16.50.44 is my home lab IP address for Pure Storage on one of the interfaces. Replace it with yours.

# Discovery using each interfaceiscsiadm -mdiscovery-t st -p 172.16.50.44:3260 -I ens224iscsiadm -mdiscovery-t st -p 172.16.50.44:3260 -I ens256







Login to targets:

iscsiadm -mnode--login

You should see something similar to this output.

### 

### 

![](/assets/images/nested-proxmox/image26.png)



### 

Verify connections:

iscsiadm -m session -P3

Part of the expected output:



![](/assets/images/nested-proxmox/image11.png)

Check multipath status:

multipath-ll

You should see something like:



![](/assets/images/nested-proxmox/image18.png)

## 

Verify SCSI Devices

After installinglsscsi:



lsscsi



If LUNs still show as “unknown,” try:

rescan-scsi-bus



Or manually rescan:

echo"- - -"| sudo tee/sys/class/scsi_host/host*/scan



You need to make the sessions persistent through reboots, otherwise you will need to loginiscsiadm -mnode-Lallevery time a host reboots.

In order to make the sessions persistent, use the following command on every host:



iscsiadm -mnode--opupdate -nnode.startup-v automatic



Then login to the targets:



iscsiadm -mnode--loginall=automatic



Find out what the Pure LUN S/N is:





![](/assets/images/nested-proxmox/image10.png)











In Proxmox Shell:



/lib/udev/scsi_id--whitelisted --device=/dev/sdc





![](/assets/images/nested-proxmox/image22.png)





Compare to Pure Storage GUI:



![](/assets/images/nested-proxmox/image28.png)



# Test connectivity and performance:

Test jumbo frames:

ping -M do -s8972172.16.50.44

### 

![](/assets/images/nested-proxmox/image13.png)



Basic I/O test:

ddif=/dev/zeroof=/dev/mapper/mpathabs=1Mcount=100oflag=direct



The multipath device will appear as/dev/mapper/mpathaand can be used like any block device in Proxmox for VM storage.



root@proxmox1:~# ddif=/dev/zeroof=/dev/mapper/mpathabs=1Mcount=100oflag=direct100+0 recordsin100+0 records out104857600 bytes (105 MB, 100 MiB) copied, 0.601568 s, 174 MB/s









On Pure Storage FlashArray GUI dashboard you should see the spike corresponding to the test you just ran:





![](/assets/images/nested-proxmox/image21.png)



# Nest Steps

Pick one based on your goals:

If you're testing performance:

Use/dev/mapper/mpathafor any workload benchmarking (e.g., with fio or inside a VM passthrough).



If you want to create a Proxmox storage volume:

Option A: LVM Volume Group

pvcreate /dev/mapper/mpathavgcreate pure-vg /dev/mapper/mpatha



Then add it to Proxmox via GUI or /etc/pve/storage.cfg:

lvm:pure-lvmvgnamepure-vgcontent images,rootdir





Option B: Format and mount as block device

mkfs.xfs /dev/mapper/mpathamkdir /mnt/pure-testmount /dev/mapper/mpatha /mnt/pure-test





In my case, I chose a different name:

root@proxmox1:~# mkfs.xfs /dev/mapper/mpathamkdir /mnt/pure-iscsi-lunmount /dev/mapper/mpatha /mnt/pure-iscsi-lunmeta-data=/dev/mapper/mpathaisize=512agcount=4,agsize=67108864 blks=sectsz=512attr=2,projid32bit=1=crc=1finobt=1,sparse=1,rmapbt=0=reflink=1bigtime=1inobtcount=1nrext64=0data     =bsize=4096blocks=268435456,imaxpct=5=sunit=0swidth=0 blksnaming   =version 2bsize=4096ascii-ci=0,ftype=1log      =internal logbsize=4096blocks=131072,version=2=sectsz=512sunit=0 blks,lazy-count=1realtime =noneextsz=4096blocks=0,rtextents=0Discarding blocks...Done.





























Notes:

Don’t use /dev/sdX paths — stick to /dev/mapper/mpatha

You can give it a friendly name in multipath.conf if you want:

multipaths{multipath{wwid"3624a9370af63fc7f25a84fd70b848412"aliaspure-data-lun}}











Then multipath -r will rename it from mpatha to pure-data-lun

Since I chose to make it a block device, Proxmox won’t show it in the web UI by default — unless you configure it as a custom directory storage backend.

Instructions: Mount & Add as Directory Storage

If you've formatted and mounted the LUN (e.g., to /mnt/pure-iscsi-lun), like I did, use this method for storing ISOs, backups, containers, or VM images.



1. Confirm it's mounted:

df -h| grep pure-iscsi-lun

2. Edit storage configuration:

nano/etc/pve/storage.cfg

Add this section:



dir:pure-iscsi-lunpath /mnt/pure-iscsi-luncontent iso,backup,vztmpl,imagesmaxfiles5







Save and exit. Then check the status:



pvesm status



root@proxmox1:~# pvesm statusName                    Type     Status           Total            Used       Available        %local                    dir     active739890443156464670284204.27%local-lvm            lvmthin     active1569628163765537911930743623.99%pure-iscsi-lun           dir     active1073217536751578410657017520.70%pure-storage             dir     active739890443156464670284204.27%pure-storage-iso         dir     active739890443156464670284204.27%















Now visible under Datacenter → Storage → pure-iscsi-lun in the Proxmox UI.



![](/assets/images/nested-proxmox/image8.png)

Congratulations! You are done!