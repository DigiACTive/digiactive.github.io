---
layout: post
title: "Being part of the open source software ecosystem"
date: 2013-12-21 00:23:08 +0800
comments: true
categories: swift django open-source
---

At DigiACTive, we're building our solution on top of an open source software stack. Practically, we had a number of reasons for chosing open source, including price, ease of access and cross-platform compatibility. More fundamentally, open source software has played a huge role in bringing technological innovations to those who would otherwise not have had access to them, and that's very much in line with how we see the Digital Canberra Challenge.

It's therefore very exciting to be part of advancing the state of open source software in a couple of small ways.

## Improving django-storage-swift ##
We recently submitted a set of patches to [django-storage-swift][dss], a Python module for interfacing between [Django][django], our web framework of choice, and [OpenStack Swift][swift], our object store of choice.

Our main contribution is support for temporary URLs. Temporary URLs provide a way for us to grant users access to their files and their files only in a simple and transparent way. In particular, they are an excellent and simple mechanism for preventing attackers from accessing confidential information by guessing file names. We also improved the out-of-the-box experience for users of django-storage-swift, and made a couple of other minor fixes and improvements.

We've made our changes available in [our fork of the project][ourdss], and contributed them back to the original project with pull requests.

## Updating the clamav chef cookbook ##
We're using [chef-solo][chef-solo] with [Vagrant][vagrant] to deploy repeatable environments for development. We recently tried to install [a cookbook][clamav-cookbook] to support [clamav][clamav] so we can virus scan user-submitted files, but discovered that it's not compatible with the most recent updates to the [yum cookbook][yumcookbookupdates]. We've updated it and submitted the changes back to the original author, and they've been well received and merged in.

## Looking forward ##
We've been making good progress building our framework and we're looking forward to making increasingly rapid progress towards the Digital Canberra Challenge project!

[dss]: https://github.com/wecreatepixels/django-storage-swift
[django]: http://django.org/
[swift]: http://docs.openstack.org/developer/swift/
[ourdss]: https://github.com/DigiACTive/django-storage-swift/tree/digiactive
[chef-solo]: http://docs.opscode.com/chef_solo.html
[vagrant]: http://vagrantup.com/
[clamav-cookbook]: https://github.com/RoboticCheese/clamav
[clamav]: http://www.clamav.net/lang/en/
[yumcookbookupdates]: https://github.com/opscode-cookbooks/yum#notes
