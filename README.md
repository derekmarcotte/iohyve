# iohyve v0.7.8
"I'd Rather be in the Delta Quadrant Edition"

FreeBSD bhyve manager utilizing ZFS and other FreeBSD tools.
*Everything is fine.*

iohyve creates, stores, manages, and launches bhyve guests utilizing built in FreeBSD features.
The idea is based on iocage, a jail manager utilizing some of the same principles.


DO YOU EVEN MAN PAGE?
````
man iohyve			# Installs with 'make install'

cat iohyve.8.txt | less		# Quick and dirty txt file
````

**Pre-Flight Checklist**

As of v0.7 `iohyve` takes care of setting up your machine if you let it.
Once you have created your ZFS pool named 'tank' you can run:
````
iohyve setup pool=tank
````
If you want `iohyve` to take care of networking, so you don't have to set up `rc.conf` you can do the following:
````
iohyve setup net=em0		# 'em0' is the interface I want bridge0 attached to.
````
You can even have `iohyve` load the required kernel modules:
````
iohyve setup kmod=1
````
You can also do all of the above at once:
````
iohyve setup pool=tank kmod=1 net=em0
````
If you want `iohyve` to set up the kernel modules and bridge0 every time you boot, add these lines to `/etc/rc.conf`:
````
iohyve_enable="YES"
iohyve_flags="kmod=1 net=em0"
````
If you want more control over your setup, feel free to read the [handbook](https://www.freebsd.org/doc/en/books/handbook/virtualization-host-bhyve.html).

**GRUB Guests**

In order to boot guests using GRUB, you must install the [sysutils/grub2-bhyve](https://www.freshports.org/sysutils/grub2-bhyve/) port. You can also just run `pkg install grub2-bhyve` if you'd like.

**NOTE**

If you are using [FreeNAS](http://doc.freenas.org/9.10/freenas_jails.html#using-iohyve), you must also have this link so your datasets are called correctly. This should be done by `iohyve setup pool=poolname` but here is the command just in case:
```
ln -s /mnt/iohyve /iohyve
```
You may also want to check out the FreeNAS [tunables](http://doc.freenas.org/9.10/freenas_system.html?highlight=persist#tunables) section of their handbook
so you can add `iohyve_enable="YES"` and `iohyve_flags="kmod=1 net=[iface]"` thus setting up the kernel modules and iohyve networking at boot time on your FreeNAS install.

**Usage**

```
iohyve  

version
setup <pool=poolname> [kmod=0|1] [net=iface]
list [-l]
info [-vsdl]
isolist
fwlist
fetchiso <URL>
cpiso <path>
renameiso <ISO> <newname>
rmiso <ISO>
fetchfw <URL>
cpfw <path>
renamefw <firmware> <newname>
rmfw <firmware>
create <name> <size> [pool]
install <name> <ISO>
load <name> <path/to/bootdisk>
boot <name> [runmode] [pcidevices]
start <name> [-s | -a]
stop <name>
forcekill <name>
scram
destroy <name>
rename <name> <newname>
delete [-f] <name>
set <name> <property=value> ...
get <name> <prop>
rmprop [-f] <name> <property>
getall <name>
add <name> <size>
remove [-f] <name> <diskN>
resize <name> <diskN> <size>
disks <name>
snap <name>@<snap>
roll <name>@<snap>
rmsnap [-f] <name>@<snap>
clone [-r] <name> <clonename>
export <name>
snaplist
taplist
tapadd <name> [iface]
tapdel <name> <tap>
activetaps
conlist
console <name>
conreset
help
```

**General Usage**

List all guests created with:
```
iohyve list
```
You can change guest properties by using set:
```
iohyve set bsdguest ram=512M                 #set ram to 512 Megabytes
iohyve set bsdguest cpu=1                    #set cpus to 1 core
iohyve set bsdguest pcidev:1=passthru,2/0/0  #pass through a pci device
```
You can also set more than one property at once:
```
iohyve set bsdguest tap=tap0 con=nmdm0		#set tap0 and nmdm0
```
You can also set a description that can be a double quoted (") string with no equals sign (=).
At guest creation, the description is the output of `date`
````
iohyve set bsdguest description="This is my string"
````
It's always prudent to `destroy` a guest before changing settings that may affect a running guest.
It's also a good idea to `destroy` a guest after your installation phase has completed.
Destroying a guest does not `delete` a guest from the host, it `destroys` the guest in `VMM`.
```
iohyve destroy bsdguest
```

Get a specific guest property:
```
iohyve get bsdguest ram
```
Get all guest properties:
```
iohyve getall bsdguest
```
Do cool ZFS stuff to a guest:
````
# Take a snapshot of a guest.
iohyve snap bsdguest@beforeupdate  #take snapshot
iohyve snaplist                    #list snapshots
iohyve roll bsdguest@beforeupdate  #rollback to snapshot

# Make an independent clone of a guest
# This is not a zfs clone, but a true copy of a dataset
iohyve clone bsdguest dolly	   #make a clone of bsdguest to dolly
````
**Creating guest templates**

You can lock a guest from being reinstalled, started, renamed, or deleted by making it a template.
To set a guest as a template, you must set the `template` property to `YES`. The `YES` must be in all caps.
EX:
```
iohyve set bsdguest template=YES
```
**Use a custom bhyve path**

If you are testing a bhyve binary that is not in base, you can specify it's full path as a property, and iohyve will use it to launch the guest. This comes in handy when testing new features branches of bhyve. 
```
iohyve set bsdguest bhyve_path=/path/to/custom/bhyve
```
**FreeBSD Guests**

Fetch FreeBSD install ISO for later:
```
iohyve fetchiso ftp://ftp.freebsd.org/.../10.1/FreeBSD-10.1-RELEASE-amd64-bootonly.iso
```
Rename the ISO if you would like:
```
iohyve renameiso FreeBSD-10.1-RELEASE-amd64-bootonly.iso fbsd10.iso
```
Create a new FreeBSD guest named bsdguest with an 8Gigabyte virtual HDD:
```
iohyve create bsdguest 8G
```
List ISO's:
```
iohyve isolist
```
Install the FreeBSD guest bsdguest:
```
iohyve install bsdguest FreeBSD-10.1-RELEASE-amd64-bootonly.iso
```
Console into the installation:
```
iohyve console bsdguest
```
Once installation is done, exit console (~~.) and stop guest:
```
iohyve stop bsdguest
```
Now that the guest is installed, it can be started like usual:
```
iohyve start bsdguest
```
Some guest os's can be gracefully stopped:
```
iohyve stop bsdguest
```
If you are having problems with a guest that is unresponsive you can forcekill it as a last resort.
USE THIS WITH CAUTION, IT WILL KILL ALL PROCESSES THAT MATCH THE NAME OF THE GUEST.
```
iohyve forcekill grubguest
```
**Other BSDs:**

Try out OpenBSD:
````
iohyve set obsdguest loader=grub-bhyve os=openbsd58
iohyve install obsdguest install58.iso
iohyve console obsdguest
````
Try out NetBSD:
````
iohyve set nbsdguest loader=grub-bhyve
iohyve set nbsdguest os=netbsd
iohyve install nbsdguest NetBSD-6.1.5-amd64.iso
iohyve console nbsdguest
````
**Linux flavors:**

Try out Debian or Ubuntu _(note LVM installs should work with os=d8lvm)_:
````
iohyve set debguest loader=grub-bhyve
iohyve set debguest os=debian
iohyve install debguest debian-8.2.0-amd64-i386-netinst.iso
iohyve console debguest
````
Try out ArchLinux:
````
iohyve set archguest loader=grub-bhyve
iohyve set archguest os=arch
iohyve install archguest archlinux-2015.10.01-dual.iso
iohyve console archguest
````
Try out CentOS or RHEL _(note version 6 would use os=centos6)_:

_Note: CentOS7 will no longer work without custom partitioning on the guest. `grub2-bhyve` cannot boot from the new CentOS7 default XFS. Please see the [wiki](https://github.com/pr1ntf/iohyve/wiki/Installing-CentOS-7-on-FreeNAS) for information on how to use custom partitioning in a CentOS kickstart file._
````
iohyve set centosguest loader=grub-bhyve
iohyve set centosguest os=centos7
iohyve install centosguest CentOS-7-x86_64-Everything-1511.iso
iohyve console centosguest
````
##### Use your own custom `grub.cfg` and `device.map` files

If you don't want iohyve to take care of the `grub.cfg` and `device.map` files, you can now "roll your own" and place them in the guests dataset (`/iohyve/guestname/`).
Of course, you must set the guest properties `loader=grub-bhyve` and `os=custom`.
For instance, if you have an OpenBSD guest located in `/iohyve/obsd59/` and an install ISO in `/iohyve/ISO/install59.iso/` and your pool is `zroot`, your files will look like this:

`device.map` file:
```
(hd0) /dev/zvol/zroot/iohyve/obsd59/disk0
(cd0) /iohyve/ISO/install59.iso/install59.iso
```
`grub.cfg` file for installation:
```
kopenbsd -h com0 (cd0)/5.9/amd64/bsd.rd
boot
```
`grub.cfg` file after installation is complete:
```
kopenbsd -h com0 -r sd0a (hd0,openbsd1)/bsd
boot
```
