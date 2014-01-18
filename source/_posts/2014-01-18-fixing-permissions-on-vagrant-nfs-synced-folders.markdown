---
layout: post
title: "Vagrant, NFS synced folders, permissions, and bindfs on CentOS 6"
date: 2014-01-18 23:59:00 +1100
comments: true
categories: vagrant nfs open-source bindfs
---
*WARNING: highly technical post ahead!*

A tale of two synced folder backend implementations
===================================================

Here at DigiACTive, we've made much use of [Vagrant][vagrant] to help us manage and deploy a consistent development environment across all our development machines. For the uninitiated, Vagrant essentially allows developers to create a standard configuration -- operating system, software packages, configuration files, and so on -- which can be automagically deployed (usually using a single command!) into a variety of environments, such as a [VirtualBox][virtualbox] virtual machine. Our development team uses both Linux and Mac OS X, and our software has a large number of dependencies, so having a standard development environment has proven very useful.

Our Vagrant configuration uses Vagrant's [synced folders][vagrant_synced_folders] to share the developer's working copy of our Git repository with the test server. Depending on the provider onto which the Vagrant setup is deployed, there are a number of different backend implementations used for synced folders. By default, VirtualBox deployments will use [VirtualBox shared folders][virtualbox_shared_folders] -- which, apart from being notoriously unreliable, have limited support for some important POSIX file system features, such as hard links.

To get around this, one can enable the alternative [NFS backend][vagrant_nfs], which doesn't have these limitations -- and can conveniently be enabled with a single line in the Vagrantfile. It all sounds great, right?

*Not quite.*

Daniel fixed up the last details of our Vagrantfile, gave it a final test, and pushed it up. Ben pulled it down on to his MacBook, ran `vagrant up`, and after a couple of hours of watching our [Chef][chef] scripts download many, many dependencies, it all worked! (Well, mostly. Anyway.)

Meanwhile, on my Debian box, I (Andrew) was getting very, very close to tearing my hair out. After fixing up a number of other problems to get Vagrant running properly, I tried to provision the VM -- and every time I tried, it complained that the provisioning scripts didn't have appropriate permissions when working with our shared folders.


NFS and permissions
===================
After some investigation, it became apparent what the issue was.

Our Chef scripts were attempting to change the owner of the configuration files they were modifying to `vagrant`, the default user set up inside the VM. However, as the synced folders are mounted using NFS, changing the ownership of a file on the remote client (the VM) means changing the ownership on the server (the host system). On the host system, [root squashing][root_squash] means that the client didn't have the root privileges necessary to do that.

However, it soon became apparent that there was another issue here:

```sh
[vagrant@localhost vagrant]$ ls -la
total 84
...
drwxr-xr-x   2 1000 1000 4096 Dec  7 05:46 scope
drwxr-xr-x   3 1000 1000 4096 Jan 16 12:13 src
drwxr-xr-x   4 1000 1000 4096 Nov 19 00:51 vagrant
```

It turns out that NFS shares UIDs between the server and clients -- which is fine if all systems use the same authentication backend. This isn't the case in a Vagrant system, obviously. NFS does not provide a built-in way to map between different server/client users -- I believe this can be accomplished with a proper directory service, but we weren't going to set that up just for a simple development VM.

This also explained why Daniel and Ben had no problems provisioning on their Macs. On my Debian host system, standard UIDs start at 1000, while in the CentOS guest system, UIDs start at 501. Coincidentally, Mac OS X also starts UIDs at 501 -- so on their systems, the files were already owned by `vagrant`.

I searched around for a few too many hours trying to find a solution, playing around with random NFS mounting options in an effort to make it work...

bindfs and vagrant-bindfs
=========================
Enter [bindfs][bindfs].

[Bind mounts][bind_mounts] have existed for a while in various \*nixes, allowing users to mount already-mounted filesystems to other locations.

bindfs takes this concept further, though. Using the wonderful powers of [FUSE][fuse], bindfs allows you to virtually alter ownership and permission bits -- which is exactly what I needed! As a FUSE filesystem, bindfs has some [unfortunate performance issues][bindfs_performance], but hey, at least it'd work!

Even better, there's a Vagrant plugin, [vagrant-bindfs][vagrant_bindfs], that allows easy configuration of bindfs mounts straight from the Vagrantfile!

I thought I'd found my solution, and proceeded to try and grab the bindfs RPM for CentOS 6...

...which didn't exist. It used to be built as part of the [EPEL][epel] repository, but alas, no more!

bindfs, vagrant-bindfs, and RPMs
================================
After much searching around for information about building RPMs, I ended up downloading the RPM spec file from the old EPEL package, updating it to bindfs 1.12.3, and compiling it myself. After learning more than I ever wanted to know about RPM building, I managed to compile a CentOS 6-compatible bindfs RPM.

However, there was still the small issue of installing this RPM during the Vagrant configuration process. For vagrant-bindfs to work, bindfs needs to be installed in the boot-up configuration phase, not the provisioning phase, which meant that adding it to our Chef scripts wasn't going to work.

Conveniently, vagrant-bindfs makes use of Vagrant's [guest capabilities][guest_capabilities] framework to detect when the guest doesn't have bindfs installed and trigger an installation capability to download the appropriate package. The installation capability needs to be implemented separately for each type of guest, and at present only Debian is supported.

I implemented a quick-and-dirty hack to extend vagrant-bindfs's installation capability for CentOS. Because bindfs has a number of dependencies which mean we can't just use `rpm -i` to install it automatically, I decided to create a YUM repository (which is actually really easy!), which makes the rest really easy.

So, after many, many hours of trying, I finally got bindfs working, and finally managed to provision my testing VM!

If you're trying to get bindfs working with Vagrant and CentOS 6, and don't want to go through the same pain I went through, I give you...

Downloads and Links!
====================
**DigiACTive provides these downloads as-is, and takes no responsibility for any issues! Use at your own risk!**

 * [bindfs-1.12.3-1.el6.x86_64.rpm](http://digiactive.com.au/digiactive-repo/centos/6/x86_64/bindfs-1.12.3-1.el6.x86_64.rpm)
 * [bindfs-debuginfo-1.12.3-1.el6.x86_64.rpm](http://digiactive.com.au/digiactive-repo/centos/6/x86_64/bindfs-debuginfo-1.12.3-1.el6.x86_64.rpm)
 * [bindfs-1.12.3-1.el6.src.rpm](http://digiactive.com.au/digiactive-repo/centos/6/SRPMS/bindfs-1.12.3-1.el6.src.rpm)
 * [vagrant-bindfs-0.2.4.digiactive2.gem](http://digiactive.com.au/digiactive-repo/gems/vagrant-bindfs-0.2.4.digiactive2.gem) (install with `vagrant plugin install`)
 * [our version of vagrant-bindfs on GitHub](https://github.com/DigiACTive/vagrant-bindfs)
 * [DigiACTive YUM Repository Configuration File](http://digiactive.com.au/digiactive-repo/DigiACTive.CentOS.repo) (put in `/etc/yum.repos.d`)
 * [DigiACTive PGP Signing Key](http://digiactive.com.au/digiactive-repo/RPM-GPG-KEY.asc)



[vagrant]: http://www.vagrantup.com/
[virtualbox]: https://www.virtualbox.org/
[vagrant_synced_folders]: http://docs.vagrantup.com/v2/synced-folders/
[virtualbox_shared_folders]: https://www.virtualbox.org/manual/ch04.html#sharedfolders
[vagrant_nfs]: http://docs.vagrantup.com/v2/synced-folders/nfs.html
[chef]: http://www.getchef.com/chef/
[root_squash]: http://en.wikipedia.org/wiki/Unix_security#Root_squash
[bindfs]: http://bindfs.org/
[bind_mounts]: http://docs.1h.com/Bind_mounts
[fuse]: http://fuse.sourceforge.net/
[bindfs_performance]: http://www.redbottledesign.com/blog/mirroring-files-different-places-links-bind-mounts-and-bindfs
[vagrant_bindfs]: https://github.com/gael-ian/vagrant-bindfs
[epel]: https://fedoraproject.org/wiki/EPEL
[guest_capabilities]: http://docs.vagrantup.com/v2/plugins/guest-capabilities.html