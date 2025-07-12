title: "Setting Up Nested Proxmox VE iSCSI Configuration with Pure Storage FlashArray"
date: 2025-07-12
author: "Dmitry Gorbatov, Consulting Systems Engineer, Pure Storage"
description: "A guide to configuring nested Proxmox environments with iSCSI storage for evaluation and functionality testing"
tags: [proxmox, iscsi, virtualization, purestorage, flasharray]
# ** Nested Proxmox VE iSCSI Configuration with Pure Storage FlashArray ** 


## **Purpose**

The purpose of this article is to detail the steps of the iSCSI configuration of the Proxmox VE instance nested on Broadcom VMware ESXi for home lab or testing environment.

## **Executive Summary**

In response to the significant licensing changes introduced by Broadcom following its acquisition of VMware, many customers are actively exploring alternatives to VMware for their virtualization infrastructure. Proxmox VE has emerged as a compelling open-source option due to its flexibility, reliability, and growing community support.

However, not all organizations have spare hardware available to evaluate Proxmox in a production-like environment. This article provides a practical guide for running nested Proxmox VE inside a VMware environment, enabling customers to test and validate Proxmox without disrupting their current infrastructure.

The guide walks through the process of deploying Proxmox as a nested VM and connecting it to a Pure Storage FlashArray using in-guest iSCSI with multipath support. This setup simulates a real-world configuration, allowing teams to evaluate Proxmox functionality and storage integration capabilities with enterprise-grade all-flash storage.

Whether you're a storage architect, virtualization admin, or infrastructure decision-maker, this article offers a clear path to hands-on testing of Proxmox VE with Pure Storage—helping accelerate informed decisions around next-generation datacenter platforms.


### **Prepare the nested Proxmox VE Server on VMware**

There are good instructions on how to install Proxmox VE VM nested on VMware

[https://www.virtualizationhowto.com/2022/01/nested-proxmox-vmware-installation-in-esxi/](https://www.virtualizationhowto.com/2022/01/nested-proxmox-vmware-installation-in-esxi/) 

If you are interested in my specific configuration, here it is



![Screenshot My Config](/assets/images/nested-proxmox/image20.png "image_tooltip")




### **Update the Debian repositories for unlicensed Proxmox VE** 



1. Login to Proxmox VE Instance with root privileges 




![Screenshot](/assets/images/nested-proxmox/image3.png "image_tooltip")




2. Ensure that nightly package updates were installed successfully 




![Screenshot](/assets/images/nested-proxmox/image15.png "image_tooltip")


If the updates failed, you will need to add no-subscription repositories by editing the sources.list


```
nano /etc/apt/sources.list
```


![Screenshot](/assets/images/nested-proxmox/image12.png "image_tooltip")


It should look like this:



![Screenshot](/assets/images/nested-proxmox/image27.png "image_tooltip")



### Network Interfaces configuration for nested Proxmox

You will need to have 2 additional network interfaces on the nested VM to configure in-guest iSCSI. They need to be on your iSCSI Network on ESXi.  

On VMware side go to the VM, right-click and go to Edit Settings:



![Screenshot](/assets/images/nested-proxmox/image16.png "image_tooltip")



![Screenshot](/assets/images/nested-proxmox/image6.png "image_tooltip")


On the Proxmox VE server:

List all network interfaces:


```
ip link show 
```



![Screenshot](/assets/images/nested-proxmox/image7.png "image_tooltip")



### Configure IPv4 addresses

Using Proxmox Web GUI

Go to Datacenter → [Your Node] → System → Network

Click Create → Linux Bridge (or Linux Bond if bonding)

Configure the bridge with your desired IP settings

Apply the configuration


![Screenshot](/assets/images/nested-proxmox/image24.png "image_tooltip")


Check that it is configured correctly:




![Screenshot](/assets/images/nested-proxmox/image19.png "image_tooltip")


Remember to hit the “Apply Configuration” button before proceeding.


### Setting MTU to 9000

In /etc/network/interfaces:


```
nano /etc/network/interfaces
```


Add mtu 9000 to each interface:



![Screenshot](/assets/images/nested-proxmox/image9.png "image_tooltip")


**Apply changes:**


```
systemctl restart networking
```


**Verify MTU:**



![Screenshot](/assets/images/nested-proxmox/image2.png "image_tooltip")


### Install and Configure iSCSI Initiator 

**Install required packages:**


```
apt install open-iscsi multipath-tools lsscsi sg3-utils
```



### Configure iSCSI initiator:


```
nano /etc/iscsi/iscsid.conf
```


Recommended settings for Pure Storage:


```
# Authentication (if using CHAP)
node.session.auth.authmethod = CHAP
node.session.auth.username = your_username
node.session.auth.password = your_password

# Timeouts
node.session.timeo.replacement_timeout = 120
node.conn[0].timeo.login_timeout = 15
node.conn[0].timeo.logout_timeout = 15
node.conn[0].timeo.noop_out_interval = 5
node.conn[0].timeo.noop_out_timeout = 5

# Session settings
node.session.iscsi.InitialR2T = No
node.session.iscsi.ImmediateData = Yes
node.session.iscsi.FirstBurstLength = 262144
node.session.iscsi.MaxBurstLength = 16776192
node.conn[0].iscsi.MaxRecvDataSegmentLength = 262144
```


Find out the IQN that is configured in Proxmox. In the shell run the following command:


```
cat /etc/iscsi/initiatorname.iscsi 
```



![Screenshot](/assets/images/nested-proxmox/image25.png "image_tooltip")



### Configure Multipath Tools

**Edit multipath configuration:**


```
nano /etc/multipath.conf
```


By default this file doesn’t exist, so you need to create it.

Pure Storage optimized configuration:


```
defaults {
    user_friendly_names yes
    find_multipaths yes
}

devices {
    device {
        vendor "PURE"
        product "FlashArray"
        path_grouping_policy "group_by_prio"
        path_selector "round-robin 0"
        path_checker "tur"
        hardware_handler "1 alua"
        prio "alua"
        failback immediate
        rr_weight uniform
        no_path_retry 18
        fast_io_fail_tmo 10
    }
}
```


**Enable and start services:**


```
systemctl enable iscsid
systemctl enable open-iscsi
systemctl enable multipath-tools
systemctl start iscsid
systemctl start open-iscsi
systemctl start multipath-tools
```



Expected output:



![Screenshot](/assets/images/nested-proxmox/image23.png "image_tooltip")



### Pure Storage Array Configuration for Host and Volumes

On Pure Storage Array - Configure host and connect volume(s)

On the Pure Storage FlashArray, create a host object with the IQN that you just discovered:

Use this IQN to create host objects in the Pure Storage Purity management interface:



* Go to Storage > Hosts.
* Click the + icon.
* In the pop-up window, give your host a name and under personality select **none**
* Add the Proxmox host IQN under the Host Ports section.



![Screenshot](/assets/images/nested-proxmox/image1.png "image_tooltip")


Create a new volume and connect it to the Proxmox host you just created

Create a volume/LUN for your host:



* Go to Storage > Volumes.
* Click the + icon.
* In the pop-up window, give your volume a name, volume size and add your newly created hosts to this volume/LUN.



![Screenshot](/assets/images/nested-proxmox/image17.png "image_tooltip")




![Screenshot](/assets/images/nested-proxmox/image5.png "image_tooltip")



### Connect to Pure Storage Target


 **Discover targets:**

On Pure Storage find what are your iSCSI target IP Addresses:



![Screenshot](/assets/images/nested-proxmox/image4.png "image_tooltip")


There  is a known limitation with open-iscsi. You can't bind both hwaddress and net_ifacename on the same interface. You need to choose one binding method. I chose the net_ifacename:

**Use net_ifacename (replace with your ens interface ID)**


```
# Delete and recreate with only interface name
iscsiadm -m iface -I ens224 -o delete
iscsiadm -m iface -I ens224 -o new
iscsiadm -m iface -I ens224 -o update -n iface.net_ifacename -v ens224

# Do the same for your second interface
iscsiadm -m iface -I ens256 -o delete
iscsiadm -m iface -I ens256 -o new
iscsiadm -m iface -I ens256 -o update -n iface.net_ifacename -v ens256
```


**Verify the configuration:**


```
iscsiadm -m iface -I ens224 -o show
iscsiadm -m iface -I ens256 -o show
```


**Now proceed with discovery:**

Note that 172.16.50.44 is my home lab IP address for Pure Storage on one of the interfaces. Replace it with yours. 


```
# Discovery using each interface
iscsiadm -m discovery -t st -p 172.16.50.44:3260 -I ens224
iscsiadm -m discovery -t st -p 172.16.50.44:3260 -I ens256
```


**Login to targets:**


```
iscsiadm -m node --login
```


You should see something similar to this output.



![Screenshot](/assets/images/nested-proxmox/image26.png "image_tooltip")


**Verify connections:**


```
iscsiadm -m session -P 3
```


Part of the expected output:




![Screenshot](/assets/images/nested-proxmox/image11.png "image_tooltip")


**Check multipath status:**


```
multipath -ll
```


You should see something like:



![Screenshot](/assets/images/nested-proxmox/image18.png "image_tooltip")


**Verify SCSI Devices**

After installing lsscsi:


```
lsscsi
```


If LUNs still show as “unknown,” try:


```
rescan-scsi-bus
```


Or manually rescan:


```
echo "- - -" | sudo tee /sys/class/scsi_host/host*/scan
```


You need to make the sessions persistent through reboots, otherwise you will need to login `iscsiadm -m node -L all` every time a host reboots.

In order to make the sessions persistent, use the following command on every host:


```
iscsiadm -m node --op update -n node.startup -v automatic
```


**Then login to the targets:**


```
iscsiadm -m node --loginall=automatic
```


**Find out what the Pure LUN S/N is:**



![Screenshot](/assets/images/nested-proxmox/image10.png "image_tooltip")


**In Proxmox Shell:**


```
/lib/udev/scsi_id --whitelisted --device=/dev/sdc
```


![Screenshot](/assets/images/nested-proxmox/image22.png "image_tooltip")


**Compare to Pure Storage GUI:**



![Screenshot](/assets/images/nested-proxmox/image28.png "image_tooltip")



### Test connectivity and performance:

**Test jumbo frames:**


```
ping -M do -s 8972 172.16.50.44
```



![Screenshot](/assets/images/nested-proxmox/image13.png "image_tooltip")


**Basic I/O test:**


```
dd if=/dev/zero of=/dev/mapper/mpatha bs=1M count=100 oflag=direct
```


The multipath device will appear as /dev/mapper/mpatha and can be used like any block device in Proxmox for VM storage.


```
root@proxmox1:~# dd if=/dev/zero of=/dev/mapper/mpatha bs=1M count=100 oflag=direct
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.601568 s, 174 MB/s
```


On Pure Storage FlashArray GUI dashboard you should see the spike corresponding to the test you just ran:



![Screenshot](/assets/images/nested-proxmox/image21.png "image_tooltip")



### **Nest Steps**

Pick one based on your goals:

**If you're testing performance:**

Use `/dev/mapper/mpatha` for any workload benchmarking (e.g., with fio or inside a VM passthrough).

**If you want to create a Proxmox storage volume:**

Option A: LVM Volume Group


```
pvcreate /dev/mapper/mpatha
vgcreate pure-vg /dev/mapper/mpatha
```


Then add it to Proxmox via GUI or /etc/pve/storage.cfg:


```
lvm: pure-lvm
  vgname pure-vg
  content images,rootdir
```


Option B: Format and mount as block device


```
mkfs.xfs /dev/mapper/mpatha
mkdir /mnt/pure-test
mount /dev/mapper/mpatha /mnt/pure-test
```


In my case, I chose a different name:


```
root@proxmox1:~# mkfs.xfs /dev/mapper/mpatha
mkdir /mnt/pure-iscsi-lun
mount /dev/mapper/mpatha /mnt/pure-iscsi-lun
meta-data=/dev/mapper/mpatha     isize=512    agcount=4, agsize=67108864 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=268435456, imaxpct=5
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=131072, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
```


Notes:

Don’t use /dev/sdX paths — stick to /dev/mapper/mpatha

You can give it a friendly name in multipath.conf if you want:


```
multipaths {
  multipath {
    wwid "3624a9370af63fc7f25a84fd70b848412"
    alias pure-data-lun
  }
}
```


Then multipath -r will rename it from mpatha to pure-data-lun

Since I chose to make it a block device, Proxmox won’t show it in the web UI by default — unless you configure it as a custom directory storage backend. 

**Instructions: Mount & Add as Directory Storage**

If you've formatted and mounted the LUN (e.g., to /mnt/pure-iscsi-lun), like I did, use this method for storing ISOs, backups, containers, or VM images.

1. Confirm it's mounted:


```
df -h | grep pure-iscsi-lun
```


2. Edit storage configuration:


```
nano /etc/pve/storage.cfg
```


Add this section:


```
dir: pure-iscsi-lun
    path /mnt/pure-iscsi-lun
    content iso,backup,vztmpl,images
    maxfiles 5
```


Save and exit. Then check the status:


```
pvesm status
```



```
root@proxmox1:~# pvesm status
Name                    Type     Status           Total            Used       Available        %
local                    dir     active        73989044         3156464        67028420    4.27%
local-lvm            lvmthin     active       156962816        37655379       119307436   23.99%
pure-iscsi-lun           dir     active      1073217536         7515784      1065701752    0.70%
pure-storage             dir     active        73989044         3156464        67028420    4.27%
pure-storage-iso         dir     active        73989044         3156464        67028420    4.27%
```


Now visible under Datacenter → Storage → pure-iscsi-lun in the Proxmox UI.



![Screenshot](/assets/images/nested-proxmox/image8.png "image_tooltip")


Congratulations! You are done!
