# Install Xen 4.2.1 with Remus and DRBD on Ubuntu 12.10
tested using 3.5.0-23-generic

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
scp ./xen-dist.tgz eross@10.0.1.50:.
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

# xen-tools gives you some handy scripts for creating guests
# http://packages.ubuntu.com/quantal/all/xen-tools/filelist
apt-get install libtext-template-perl libconfig-inifiles-perl libfile-slurp-perl liblist-moreutils-perl
wget http://mirror.pnl.gov/ubuntu/pool/universe/x/xen-tools/xen-tools_4.3.1-1_all.deb
dpkg -i xen-tools_4.3.1-1_all.deb

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
cd drbd
make install
vi /etc/modules
# add drbd
#

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
lvcreate -n drbdtest -L 10G ubuntu
vi /etc/drbd.d/SystemHA_protoD.res
#
resource drbd-vm {
        device /dev/drbd1;
        disk /dev/ubuntu/drbdtest;
        meta-disk internal;
        on cheng-HP-Compaq-Elite-8300-SFF {
                address 147.8.177.50:7791;
        }
        on wang-HP-Compaq-Elite-8300-SFF {
                address 147.8.179.243:7791;
        }
}
#

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
![drbd](https://github.com/wangchenghku/Remus/blob/master/.resources/drbd.png)

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
mkdir -p /var/lib/xen/images/ubuntu-netboot
cd /var/lib/xen/images/ubuntu-netboot
wget http://ubuntu.c3sl.ufpr.br/ubuntu/dists/precise/main/installer-amd64/current/images/netboot/xen/initrd.gz
wget http://ubuntu.c3sl.ufpr.br/ubuntu/dists/precise/main/installer-amd64/current/images/netboot/xen/vmlinuz

vi /etc/xen/SystemHA.cfg
#
name = "SystemHA"

memory = 256

disk = [ 'drbd:drbd-vm,xvda,w' ]
vif = [ 'mac=18:66:da:03:15:b1,bridge=xenbr0' ]

kernel = "/var/lib/xen/images/ubuntu-netboot/vmlinuz"
ramdisk = "/var/lib/xen/images/ubuntu-netboot/initrd.gz"
extra = "debian-installer/exit/always_halt=true -- console=hvc0"
#

xm create /etc/xen/SystemHA.cfg -c
# run the install

vi  /etc/xen/SystemHA.cfg
# 
bootloader = "/usr/lib/xen-4.1/bin/pygrub"
# comment out 'kernel' 'ramdisk' and 'extra'
#

xm create /etc/xen/SystemHA.cfg -c
```
Sometimes, for development or analysis purposes, you dont really want to replicate to a physical machine. You just want to gather up the statistics such as number of pages that changed in a checkpoint, size of data sent, etc. In this case, all you need is a system to continuously checkpoint the VM and replicate it to a sink like `/dev/null`, while still gathering up stats.
```
#remus options:
  -i MS, --interval=MS checkpoint every MS milliseconds
  --blackhole          replicate to /dev/null (no disk checkpoints, only memory & net buffering)
  --no-net             run without net buffering (benchmark option)
#
remus -i 40 --blackhole --no-net SystemHA dummyHost >/var/log/xen/domU-blackhole.log 2>&1 &
```
The VM SystemHA is continuously checkpointed but replicated to `/dev/null`. Gather up all the stats you want and then kill remus
```
pkill -USR1 remus
```

### Manually installing an HVM Guest VM

```
sudo lvcreate -L 4G -n ubuntu-hvm /dev/ubuntu
```
Create a guest config file `/etc/xen/ubuntu.cfg`
```
builder = "hvm"
name = "ubuntu"
memory = "512"
vcpus = 1
vif = ['mac=18:66:da:03:15:b1,bridge=xenbr0']
disk = ['phy:/dev/ubuntu/ubuntu-hvm,hda,w','file:/home/cheng/ubuntu-12.04-desktop-amd64.iso,hdc:cdrom,r']
vnc = 1
boot="dc"
```

```
xm create /etc/xen/ubuntu.cfg
vncviewer localhost:0 
```

Once you have installed by formatting the disk and by following the prompts the domain will restart - however this time we want to prevent it booting from DVD so destroy the domain with
```
xm destroy ubuntu
```
Then change the boot line in the config file to read `boot="c"` restart the domain with
```
xm create /etc/xen/ubuntu.cfg
```
