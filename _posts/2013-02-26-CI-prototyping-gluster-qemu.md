---
layout: post 
title: "Converged Infrastructure hacking" 
date: 2013-02-26
---
I just wrapped up my presentation at the [Gluster Workshop at CERN][gluster-cern] where I discussed Open Source advantages in tackling converged infrastructure challenges.  Here is my [slidedeck]({{ site.url }}/assets/CI_presentation_26Feb2013.pdf).  Just a quick heads up, there's some animation that's lost in the pdf export as well as color commentary during almost every slide.

During the presentation I demo'ed out the new QEMU/GlusterFS native integration leveraging libgfapi.  For those of you wondering what that means, in short, there's no need for FUSE anymore and QEMU leverages GlusterFS natively on the back end.  Awesome.

So for my demo I needed two boxes running QEMU/KVM/GlusterFS.  This would provide the compute and storage hypervisor layers.  As I only have a single laptop to tour Europe with, I obviously needed a nested KVM environment.  

If you're got enough hardware feel free to skip the Enable Nested virtualization section and skip ahead to the Base OS installation.

This wasn't as easy envionment to get up and running, this is alpha code boys and girls so expect to roll your sleeves up.  Ok with that out of the way, I'd like to walk through the steps I did in order to get my demo envionment up and running.  These installation assumes you have Fedora 18 installed and updated with virt-manager and KVM installed.

# Enable Nested Virtualization
Since we're going to want to install an OS on our VM running on our Gluster/QEMU cluster that we're building, we'll need to enable Nested Virtualization. Let's first check and see if nested virtualization is enabled.  If it responds with an N then No.  If yes, skip this section to the install.

{% highlight rub  %}
$ cat /sys/module/kvm_intel/parameters/nested
N
{% endhighlight %}

If it's not we'll need to load a KVM specific module with the nested option loaded.  The easist way to change this is using the modprobe configuration files:

{% highlight rub  %}
$ echo “options kvm-intel nested=1″ | sudo tee /etc/modprobe.d/kvm-intel.conf
{% endhighlight %}

Reboot your machine once the changes have been made and check again to see if the feature is enabled:

{% highlight rub  %}
$ cat /sys/module/kvm_intel/parameters/nested
Y
{% endhighlight %}

That's it we're done with prepping the host.

# Install VMs OS
Starting with my base Fedora laptop, I've installed virt-manager for VM management.  I wanted to use Boxes, but it's not designed for this type of configuration. So. Create your new VM, I selected the "Fedora http install" as I didn't have an iso laying around. Also http install=awesome.

![multisite](/assets/gluster01-150x150.png){:class="img-responsive"}

To do this, select the http install option and enter the nearest available location.  

![multisite](/assets/gluster02-150x150.png){:class="img-responsive"}

For me this was the Masaryk University, Brno (where I happened to be sitting during [Dev Days 2013][dev-days-2013]

{% highlight rub  %}
http://ftp.fi.muni.cz/pub/linux/fedora/linux/releases/18/Fedora/x86_64/os/
{% endhighlight %}

I went with an 8 gig base disk to start (we'll add another one in a bit), gave the VM 1G of ram and a default vCPU.  Start the VM build and install.

![multisite](/assets/gluster03-150x150.png){:class="img-responsive"}

The install will take a bit longer as it's downloading the install files during the intial boot.

![multisite](/assets/gluster04-150x150.png){:class="img-responsive"}

Select the language you want to use and continue to the installation summary screen.  Here we'll want to change the software selection option.

![multisite](/assets/gluster05-150x150.png){:class="img-responsive"}

and select the minimal install:

![multisite](/assets/gluster06-150x150.png){:class="img-responsive"}

during the installation, go ahead and set the root password:

![multisite](/assets/gluster07-150x150.png){:class="img-responsive"}

Once the installation is complete, the VM will reboot.  Once done, power it down.  Although we've enable Nested Virtualization, we need to pass the CPU flags onto the VM.

In the virt-manager window right click on the VM, and select open. In the VM window, select view > details.  Rather than guessing the cpu architecture, select the copy from host and select Ok.

![multisite](/assets/gluster08-150x150.png){:class="img-responsive"}

While you're here go ahead and add an additional 20 gig virtual drive.  Make sure you select virtio for the drive type!

![multisite](/assets/gluster09-150x150.png){:class="img-responsive"}

Boot your VM up and let's get started.

# Base installation components

You'll need to install some base components before you get started installing GlusterFS or QEMU.

After logging in as root,

{% highlight rub  %}
yum update
yum install nettools wget xfsprogs binutils 
{% endhighlight %}

Now we're going to create the mount point and format the additional drive we just installed.

{% highlight rub  %}
mkdir -p /export/brick1
mkfs.xfs -i size=512 /dev/vdb
{% endhighlight %}

We'll need to edit our fstab and add this as well, so that it will remain persistent going forward after any reboots.
add the following line to /etc/fstab

{% highlight rub  %}
/dev/vdb /export/brick1 xfs defaults 1 2 
{% endhighlight %}

Once you're done with this, let's go ahead and mount the drive.

{% highlight rub  %}
mount -a && mount
{% endhighlight %}

# Firewalls.  YMMV
it may be just me (I'm sure it is) but I struggled getting gluster to work with firewalld on fedora 18.  This is not reccomeneded in production envionments, but for our all in VM on a laptop deployment, I just disabled and removed firewalld.

{% highlight rub  %}
yum remove firewalld
{% endhighlight %}

# Gluster 3.4.0 Alpha Installation
First thing we'll need to do on our VM is configure and enable the gluster repo.

{% highlight rub  %}
wget http://download.gluster.org/pub/gluster/glusterfs/qa-releases/3.4.0alpha/Fedora/glusterfs-alpha-fedora.repo
{% endhighlight %}

and move it to /etc/yum.repos.d/

{% highlight rub  %}
mv glusterfs-alpha-fedora.repo /etc/yum.repos.d/
{% endhighlight %}

Now we enable the repo and install glusterfs:

{% highlight rub  %}
yum update
yum install glusterfs-server glusterfs-devel
{% endhighlight %}

Important to note here we need the gluster-devel package for the QEMU integration we'll be testing.  Once done we'll start the  glusterd service and verify that it's working.

# break break 2nd VM
Ok folks, if you've made it here, get a coffee and do the install again on a 2nd VM.  You'll need the 2nd replication VM target before you proceed.
{% highlight rub  %}
</end coffee break>
{% endhighlight %}

# break break Network Prepping both VMs
As we're on the private nat'd network on our laptop that virt-manager is managing, we'll need to update our VMs we create and assign static addresses, as well as editing the /etc/hosts file to add both servers with thier addresses.  We're not proud here people, this is a test envionment, if you want to use proper DNS, I won't judge if you don't.

1) change both VMs to using static addresses in the nat range.
2) change VMs hostnames
3) update both VMs /etc/hosts to include both nodes.  This is hacky but expedient.

# back to Gluster

start and verify the gluster services on both VMs.

{% highlight rub  %}
service glusterd start
service glusterd status
{% endhighlight %}

On either host, we'll need to create the gluster volume and set it for replication.
{% highlight rub  %}
gluster volume create vmstor replica 2 ci01.local:/export/brick1 ci02.local:/export/brick1
{% endhighlight %}

Now we'll start the volume we just created
{% highlight rub  %}
gluster volume start vmstor
{% endhighlight %}

Verify that everything is good, if this returns fine, you're up and running with GlusterFS!
{% highlight rub  %}
gluster volume info
{% endhighlight %}

# building QEMU dependancies
let's get some prereqs for getting the latest qemu up and running

{% highlight rub  %}
yum install lvm2-devel git gcc-c++ make glib2-devel pixman-devel
{% endhighlight %}

Now we'll download QEMU:

{% highlight rub  %}
git clone git://git.qemu-project.org/qemu.git
{% endhighlight %}

The rest is pretty standard compiling from source.  you'll start with configuring your build.  I'll trim the target list to save time as I know I'm not going to use many of the QEMU supported architectures.
{% highlight rub  %}
./configure --enable-glusterfs --target-list=i386-softmmu,x86_64-softmmu,x86_64-linux-user,i386-linux-user
{% endhighlight %} 

With that done everything on this host is done, and we're ready to start building VMs using GlusterFS natively bypassing fuse and leveraging thin provisioning. W00!

# Creating Virtual Disks on GlusterFS
{% highlight rub  %}
qemu-img create gluster://ci01:0/vmstor/test01?transport=socket 5G
{% endhighlight %}

Breaking this down, we're using qemu-img to create a disk natively on GlusterFS that's five gigs in size.  I'm looking for some more information about what the transport socket is, expect an answer soonish.

# Build a VM and install an OS onto the GlusterFS mounted disk image

At this point you'll want something to actually install on your image.  I went with TinyCore because as it is I'm already pushing up against the limitations of this laptop with nested virtualization.  You can download [TinyCore Linux here][tiny-core].	

{% highlight rub  %}
qemu-system-x86_64 --enable-kvm -m 1024 -smp 4 -drive file=gluster://ci01/vmstor/test01,if=virtio -vnc 192.168.122.209:1 --cdrom /home/theron/CorePlus-current.iso
{% endhighlight %}

This is the quickest way to get this moving, I skipped using Virsh for the demo, and am assigning the VNC IP and port manually.  Once the VM starts up you should be able to connect to the VM from your external host and start the install process.

![multisite](/assets/gluster10-150x150.png){:class="img-responsive"}

To get the install going, select the harddrive that was build with qemu-img and follow the OS install procedures.

# finished!
At this point you're done and you can start testing and submitting bugs!
I'd expect to see some interesting things with OpenStack in this space as well as tighter oVirt integration moving forward.
Let me know what you think about this guide and if it was useful.

# side note
Also, something completely related.  I'm pleased to announce that I've joined the Open Source and Standards team at [Redhat][redhat] working to promote and assist making upstream projects wildly successful.  If you're unsure what that means or you're wondering why Red Hat cares about upstream projects, PLEASE reach out and say hello.

# References:
* [nested KVM][nested-kvm]
* [KVM VNC][kvm-vnc]
* [Using QEMU to boot VM on GlusterFS][qemu-gluster-boot]
* [QEMU downloads][qemu-download]
* [QEMU Gluster native file system integration][qemu-glusterfs]




[gluster-cern]: http://www.gluster.org/community/documentation/index.php/Planning/CERN_Workshop
[tiny-core]: http://distro.ibiblio.org/tinycorelinux
[nested-kvm]: http://www.rdoxenham.com/?p=275
[kvm-vnc]: http://www.cyberciti.biz/faq/linux-kvm-vnc-for-guest-machine
[qemu-gluster-boot]: http://www.youtube.com/watch?v=JG3kF_djclg
[qemu-download]: http://qemu-project.org/Download
[qemu-glusterfs]: http://raobharata.wordpress.com/2012/10/29/qemu-glusterfs-native-integration
[redhat]: http://www.redhat.com
[dev-days-2013]: http://www.devconf.cz/
