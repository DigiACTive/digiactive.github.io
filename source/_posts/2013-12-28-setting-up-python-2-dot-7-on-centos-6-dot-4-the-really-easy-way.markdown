---
layout: post
title: "Setting up Python 2.7 on CentOS 6.4 the (really) easy way"
date: 2013-12-28 16:23:30 +0800
comments: true
categories: python centos
---

Setting up Python 2.7 on CentOS 6.4 is not particularly easy: CentOS ships with 2.6 and yum seems to depend on it.

The traditional way to do this is to build Python from source, and there is even a [Chef cookbook][python_build] to do this.

However, this is slow, and when you want to verify that your box builds correctly so that your team can start work on it, you want to minimise these sorts of delays. So I (Daniel) have been hunting around for the best way to install pre-built RPMs.

The closest I found was this excellent article from Red Hat: [Setting up Django and Python 2.7 on Red Hat Enterprise 6 the easy way](http://developerblog.redhat.com/2013/02/14/setting-up-django-and-python-2-7-on-red-hat-enterprise-6-the-easy-way/).

To summarise the article, RHEL (and CentOS) support 'software containers'. They're sort of like virtualenvs or RVMs for packages - isolated, but more lightweight than full chroots. Python 2.7 is a supported container. Not all the steps are the same for CentOS as they are for RHEL, so here's an update of the article for CentOS:

+ CentOS makes it even easier than RHEL to set up the SCLs. Instead of writing random files to your filesystem, you just need to ```yum install centos-release-SCL```
+ You're now ready to do a ```yum install python27```
+ As with RHEL, you can now pop a shell with Python 2.7 by simply doing ```scl enable python27 bash```
    + If you want to run commands with more than one argument, you need to enclose them in quotes, otherwise `scl` will interpret them as other environments to enable.

**As long as you are happy running `scl enable python27 bash` before work, you're now set.**

# The gotchas

While this is much, much faster than building from source, there are two, interrelated gotchas to bear in mind.

## Virtualenvs no longer shield you from interpreter differences
I naively expected that virtualenvs would 'just work' and would prevent me from having to think about the extra layer of 'magic' I had just added.

I was wrong.

If you create a virtualenv inside scl, and then exit scl and try to run Python, it fails:

```shell
[vagrant@localhost tmp]$ scl enable python27 bash # enable the Python 2.7 environment
[vagrant@localhost tmp]$ virtualenv foo
[vagrant@localhost tmp]$ exit # leave the 2.7 environment, back to 2.6
[vagrant@localhost tmp]$ source foo/bin/activate
(foo)[vagrant@localhost tmp]$ python
python: error while loading shared libraries: libpython2.7.so.1.0: cannot open shared object file: No such file or directory
```

You get the same error when you try to create a virtualenv with the newly installed binary outside of scl.

The correct solution is to just wrap everything in scl: making `scl enable python27 bash` part of your workflow, just like `source env/bin/activate`.

## Not everything is easy to wrap in scl enable

For times when it is impossible or very difficult to use scl enable as designed (for example when you're using someone else's chef cookbook), you can work around it. On my 64 bit system, the library it's looking for is in `/opt/rh/python27/root/usr/lib64`, so you just need to set an environment variable for that:

```shell
env LD_LIBRARY_PATH=/opt/rh/python27/root/usr/lib64 /opt/rh/python27/root/usr/bin/python
```

If you're writing a Chef cookbook, you can give an `execute` block (or similar) an environment parameter:

```ruby
execute "foo" do
  ...
  environment "LD_LIBRARY_PATH" => "/opt/rh/python27/root/usr/lib64/"
end
```

# Conclusion
Using a packaged Python is a "good thing". The most obvious reason is that of security: if you've hardcoded a version of Python to download, build and install, and a security hole is found and a new version is released, you'll have to manually update all your recipes. If you're using distro packages, it'll happen more or less automatically in your regular updates.

The second reason distro packages are better is simply that of speed. Time is money: why spend it waiting Python to build?

Hopefully this makes it a bit easier for you all.

[python_build]: https://github.com/shimizukawa/chef-python-build
