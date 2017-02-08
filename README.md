# Install Xen 4.2.1 with Remus and DRBD on Ubuntu 12.10 (tested using 3.5.0-23-generic)

Startup Menu -> Boot Menu -> Security -> System Security

Virtualization Technology (VTx) -> Enabled

Virtualization Technology Directed I/O (VTd) -> Enabled

## Install a new Ubuntu 12.10 system (PC (Intel x86) desktop image)
Both Servers 
```
Choose 'Use entire disk with LVM'
```

Network setup
```
apt-get install bridge-utils
vi /etc/network/interfaces
#
auto xenbr0
iface xenbr0 inet dhcp
bridge_ports eth0

auto eth0
iface eth0 inet manual
#
```

```
vi /etc/hosts
#
147.8.177.50  cheng-HP-Compaq-Elite-8300-SFF
147.8.179.243 wang-HP-Compaq-Elite-8300-SFF
#
```

## Install Xen
### Install Xen Pre-requisites
Server \#1

Short-cut install for dependencies
```
apt-get build-dep xen
apt-get install libc6-dev libglib2.0-dev libyajl-dev yajl-tools libbz2-dev bison flex zlib1g-dev git-core texinfo debhelper debconf-utils debootstrap fakeroot
```

### Install Xen 4.2.1
Server \#1
```
apt-get install transfig libpixman-1-dev
#latest
apt-get install git-core
git clone git://xenbits.xen.org/xen.git
#release
wget http://bits.xensource.com/oss-xen/release/4.2.1/xen-4.2.1.tar.gz
tar zxvf xen-4.2.1.tar.gz
cd xen-4.2.1
wget http://download.locatrix.com/xen/remus-qdisc-py.patch
patch -p1 < remus-qdisc-py.patch
./configure
vi ./.config
#
PYTHON_PREFIX_ARG=--install-layout=deb
#
make world
make deb

tar zcvf xen-dist.tgz ./xen-4.2.1/dist
# scp to server #2
```
Server \#2
```
tar zxvf xen-dist.tgz
```
Both servers
```
cd /home/eross/xen-4.2.1/dist
./install.sh

update-grub

ls -al /boot/xen*
-rw-r--r-- 1 root root   802303 Jan 30 20:57 /boot/xen-4.2.1.gz
lrwxrwxrwx 1 root root       12 Jan 30 20:57 /boot/xen-4.2.gz -> xen-4.2.1.gz
lrwxrwxrwx 1 root root       12 Jan 30 20:57 /boot/xen-4.gz -> xen-4.2.1.gz
lrwxrwxrwx 1 root root       12 Jan 30 20:57 /boot/xen.gz -> xen-4.2.1.gz
-rw-r--r-- 1 root root 15388780 Jan 30 20:57 /boot/xen-syms-4.2.1

update-rc.d xencommons defaults 19 18
update-rc.d xendomains defaults 21 20
update-rc.d xen-watchdog defaults 22 23

grep Xen /boot/grub/grub.cfg
# You should see "Ubuntu GNU\/Linux, with Xen hypervisor" in there, that's where I got the below from
sed -i 's/GRUB_DEFAULT=.*\+/GRUB_DEFAULT="Ubuntu GNU\/Linux, with Xen hypervisor"/' /etc/default/grub
update-grub

reboot
```

## Install DRBD
Server #1
```
apt-get install autoconf build-essential
wget http://remusha.wikidot.com/local--files/configuring-and-installing-remus/drbd-8.3.11-remus.tar.gz
cd ./drbd-8.3.11
./autogen.sh
./configure --prefix=/usr --localstatedir=/var --sysconfdir=/etc --with-km
make
make install
cd drbd
make clean all
make install
vi /etc/modules
# add drbd
#
tar zcvf drbd.tgz ./drbd-8.3.11
# scp to server #2
```

Server #2
```
tar zxvf ./drbd.tgz
cd ./drbd-8.3.11
make install
vi /etc/modules
# add drbd
#
```

Both Servers
```
lvreduce allows you to reduce the size of a logical volume.
OPTIONS
       -L, --size [-]LogicalVolumeSize[bBsSkKmMgGtTpPeE]
              With the - sign the value will be subtracted  from  the  logical
              volume's actual size.
       -r, --resizefs
              Resize underlying filesystem together with  the  logical  volume using fsadm(8).

lvreduce --resizefs --size -50G /dev/ubuntu/root

cp /home/user/drbd-8.3.11/scripts/global_common.conf.protoD /etc/drbd.d/global_common.conf
cp /home/user/drbd-8.3.11/scripts/testvms_protoD.res /etc/drbd.d/SystemHA_protoD.res
lvcreate -n test -L 10G ubuntu
vi /etc/drbd.d/SystemHA_protoD.res
#
resource drbd-vm {
        device /dev/drbd1;
        disk /dev/ubuntu/test;
        meta-disk internal;
        on cheng-HP-Compaq-Elite-8300-SFF {
                address 147.8.177.50:7791;
        }
        on wang-HP-Compaq-Elite-8300-SFF {
                address 147.8.179.243:7791;
        }
}
#

device name
The name of the block device node of the resource being described.
You must use this device with your application (file system) and you must not use the low level block device which is specified with the disk parameter.

disk name
DRBD uses this block device to actually store and retrieve the data. Never access such a device while DRBD is running on top of it.

#Create the meta-data for the SystemHA-disk and then bring up the resource. Do this on both machines
drbdadm create-md drbd-vm
###answer y or yes for all questions, in the above command
drbdadm up drbd-vm
```
Server #1 - this will override all of the DRBD data on server #2
```
drbdadm -- --overwrite-data-of-peer primary drbd-vm
```
From `/proc/drbd` you can monitor the current status of the DRBD resource.
![drbd](https://github.com/wangchenghku/Remus/blob/master/drbd.png)

Notes:
- When Remus is running you can do 'cat /proc/drbd' and you'll notice the status will be Primary/Primary
- When configuring your VM, use drbd:drbd-vm for the disk. not phy:/… or any other format
- It's one VM per DRBD disk like this, so you have to create a new one for each VM

### Setup Remus
Both servers
```
vi /etc/xen/xend-config.sxp
# Ensure the following are uncommented
(xend-relocation-server yes)
(xend-relocation-port 8002)
(xend-relocation-address '')
(xend-relocation-hosts-allow '')
#

vi /etc/modules
#
sch_plug
sch_prio
sch_ingress
cls_basic
cls_tcindex
cls_u32
act_mirred
ifb
#
```

### Test VM with Remus (PV Guest)
Server #1
```
vi /etc/xen/rt.cfg
#
name = "rt"

memory = 256

disk = [ 'drbd:drbd-vm,xvda,w' ]
vif = [ 'mac=18:66:da:03:15:b1' ]

kernel = "/var/lib/xen/images/ubuntu-netboot/vmlinuz"
ramdisk = "/var/lib/xen/images/ubuntu-netboot/initrd.gz"
extra = "debian-installer/exit/always_halt=true -- console=hvc0"
#

xm create /etc/xen/rt.cfg -c
# run the install

vi  /etc/xen/rt.cfg
# 
bootloader = "/usr/lib/xen/bin/pygrub"
# comment out 'kernel' 'ramdisk' and 'extra'
#

# Press Ctrl+] to exit the console view
xm create /etc/xen/rt.cfg -c

# see it running
xm list

# reconnect
xm console rt

# immediately terminate
xm destroy rt
```
Sometimes, for development or analysis purposes, you dont really want to replicate to a physical machine. You just want to gather up the statistics such as number of pages that changed in a checkpoint, size of data sent, etc. In this case, all you need is a system to continuously checkpoint the VM and replicate it to a sink like `/dev/null`, while still gathering up stats.
```
#remus options:
  -i MS, --interval=MS checkpoint every MS milliseconds
  --blackhole          replicate to /dev/null (no disk checkpoints, only memory & net buffering)
  --no-net             run without net buffering (benchmark option)
#
remus -i 40 --blackhole --no-net rt dummyHost >/var/log/xen/domU-blackhole.log 2>&1 &
```
The VM rt is continuously checkpointed but replicated to `/dev/null`. Gather up all the stats you want and then kill remus
```
pkill -USR1 remus
```
Remus supports most domU guests that can be run in PV, however, you may see this warning when you start Remus: 
```
"WARNING: suspend event channel unavailable, falling back to slow xenstore signalling"
```
This means the kernel in the guest doesn't have "suspend event channel" support, which in turn basically Remus is going to work, but isn't going to perform well.
