---
layout: post
title: "An analysis on dell's dset tool"
date: 2012-10-16 22:26
comments: true
categories: sysadmin dell
---

what is dset?

DSET is "Dell System E-Support Tool", it is available for windows, 32bit linux and 64 bit linux. Essentially dell uses this tool to gather information about your server to help them troubleshoot. I'm only writing about the 64 bit linux version.

why do you care?

If you call dell tech support sooner or later (sooner I'm betting) they are going to ask for a dset. I wanted to dig into, what they are gathering and share with my fellow sysadmins.


#### The story

I had a raid controller battery go bad after 2 years, still under warranty, but dell wanted a dset. Never mind I had a dmidecode for the firmware versions and some megacli64 output to show the battery reports, they wanted a dset.  

Here's how my conversation went:  

dell: There is a firmware bug that falsely reports a bad battery when your battery is still good, we are going to need a dset.  
me: does dset need root?  
dell: yes.  
me: what does it need root for, as in, what exactly is it running?  
dell: I don't know.   
me: okay I'll download it, check it out, and call you back.  

I'm in the medical industry, so installing new/unvetted software on production servers is usually a no-no, and it makes me nervous. So I wasn't about to install it without some testing and analysis (on a blank virtual machine).

In fairness to dell, I could have said "I'm a gold customer with a 4 hour turnaround, ship me the new battery now please" and they would have. but I like to be co-operative with my fellow techies if I can.


# The analysis

I'm not one to trust binaries that need to be run as root very much, so let's take a look at what we are getting ourselves into.

Here's [dell's dset page](http://support.dell.com/support/topics/global.aspx/support/en/dell_system_tool)

at time of writing the latest version is 3.2.0.141_x64_A01.bin

Due to running this on a virtual machine it didn't want to install from running the script, so I extracted it manually via:

    tail -n+20 dell-dset-3.2.0.141_x64_A01.bin | tar -xvz

The tarball doesn't make it's own directory it litters a bit in your current directory (shame shame).


install.sh
----------

The install.sh does a bunch of pre-checks to see if you have supported hardware before collecting it's data. It's fairly simple to disable the check and run it anyway or add your system to the 'support_hw_list' or just copy some exports from the install script and run (more on this later).

Once it decides it will install it will install one/some/all of the following depending on what options you choose:

    rpm -Uhv rpms/srvadmin-hapi* >/dev/null 2>&1
    rpm -Uhv rpms/srvadmin-storelib-sysfs*.rpm >/dev/null 2>&1
	rpm -Uhv rpms/dell-dset-common* --nodeps >/dev/null 2>&1
	rpm -Uhv rpms/dell-dset-collector* --nodeps >/dev/null 2>&1
	rpm -Uhv rpms/dell-dset-provider* --nodeps >/dev/null 2>&1

rhel only:
	rpm -ihv rpms/RHEL/sblim-sfcb*.rpm --nodeps >/dev/null 2>&1
	rpm -Uhv rpms/RHEL/sblim-cmpi-base*.rpm --nodeps >/dev/null 2>&1

sles only:
	rpm -Uhv rpms/SLES/cim-schema*.rpm >/dev/null 2>&1
	rpm -ihv rpms/SLES/sblim-sfcb*.rpm --nodeps >/dev/null 2>&1
	rpm -Uhv rpms/SLES/sblim-indication_helper*.rpm >/dev/null 2>&1
	rpm -Uhv rpms/SLES/sblim-cmpi-base*.rpm --nodeps >/dev/null 2>&1


Okay, I'm not a fan of the --nodeps, but if they are staying in their own sandbox I can forgive them. So let's find out. On the first set of rpms:

    $ rpm -qlp *.rpm |grep -v "^/opt"
    /etc/init.d/dsm_sa_ipmi
    /etc/init.d/instsvcdrv
    /etc/ld.so.conf.d/srvadmin-hapi-x86_64.conf
    /etc/sysconfig/dsm_sa_ipmi
    /usr/lib64/libdchapi.so.5
    /usr/lib64/libdchapi64.so
    /usr/lib64/libdchbas.so.5
    /usr/lib64/libdchbas64.so
    /usr/lib64/libdchcfl.so.5
    /usr/lib64/libdchcfl64.so
    /usr/lib64/libdchesm.so.5
    /usr/lib64/libdchesm64.so
    /usr/lib64/libdchipm.so.5
    /usr/lib64/libdchipm64.so
    /usr/lib64/libdchtvm.so.5
    /usr/lib64/libdchtvm64.so


Alright, not going to install this in the production environment already, but lets throw caution to the wind on this vm.

I'm running centos, so depending on what options I select, the dset installer may install sblim (sublime), with nodeps of course. This could be problematic if you are also using the epel sublime package. 

At least they are no longer using rpm --force --nodeps like they were in previous versions of dset (which they still get you to use if you are using rhel 5.x)).

The install.sh parses out what you want to do and runs the 'collector' program with the various options (more on this later).


Manual Install
--------------

Going to play with the dell-dset*.rpm's first.

For fun I looked at a rpm -ivh dell-dset*.rpm to see what dependencies I was about to ignore. What's weird is they are packaging all of their dependencies, so I'm not sure why they don't just update the spec to do a provides: bla and fix it. Maybe they are trying not to mess with the rpm database, but if that was the case, why are we using rpms at all? Running rpm --nodeps is almost the same as doing a tarball. How much stuff are they doing in their rpm's %pre and %post that they can't do in their install script? I digress.

Let's get this installed.

	rpm --nodeps -ivh dell-dset*.rpm

This will install to /opt/dell/advdiags/dset


The Collector
-------------

	cd /opt/dell/advdiags/dset/bin
	./collector --help
	  File "/usr/lib/python3.1/site-packages/cx_Freeze/initscripts/Console3.py", line 27, in <module>

Now we know it's a python 3.1 script. :)

	$ file collector
	collector: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.4, stripped

But not something we can easily look at without getting a python decompiler or strace involved.

Our paths won't be correct but dell gave us a collector.sh which will setup the correct paths for you (and restore your old path when you are done) that's nice.

	./collector.sh --help

there we go. a nice help file so we can figure out what we want to do. Being that I'm on a vm, the hardware option isn't going to do much for me. So I'll start with the software.

	./collector.sh -d sw

It asks right away for my root password (I'm already running it as root). If I reached this stage in production I would abort, but I'm on a vm, so I'll break out and change root to '1234' and run it again. But not to worry, the dset documentation states:

    NOTE: Root credentials are necessary for the DSET
            Provider to collect inventory or
            configuration information about the system.
            DSET does not store this password. The root
            password must be specified each time a report
            is collected.

We'll find out if this is true or not shortly.

	./collector.sh -d sw -p 1234

huzzah! I have a report.

	./collector.sh -d sw -p 1234 -v yes

Now I have a report with privacy enabled to compare.


The Report
----------

The report gets thrown into a passworded zip file. The password is completely meaningless as, if you unzip it with no password, it unzips a text file which tells you the password is 'dell'. So I unzip'ed again this time with the super secret password. The password is the same in privacy mode or non privacy mode.


### The non privacy report

The non privacy report gathers it's data from many cfg files and logs. It also parses the data into xml/xsl pages which I assume dell has a nice tool to go through quickly to see what's what.  

From looking at the logs directory the collector is gathering the following:

	cat boot/grub/menu.lst
	cat boot/grub/device.map
	ls /boot
	uptime
	cat /proc/meminfo
	lsmod
	cat /etc/modprobe.conf
	cat lib/modules/current/modules.dep
	ifconfig
	cat /etc/resolv.conf
	cat /etc/hosts
	cat /etc/sysconfig/network-scripts/ifcfg-*
	df
	cat /proc/scsi/scsi
	fdisk -l
	free
	hostname
	list of all installed rpms / versions / etc
	iptables dump
	ldconfig
	lspci
	mount
	osversion
	print's environment settings
	ps
	pstree
	route (old school route not ip route ls, shame shame)
	selinux policies
	runs thru the init.d and does a service status on each
	sestatus
	uname

See that ps up there?  

	grep "collector" ps
	root     20589 13.0  1.5 133620 15408 pts/1    S+   23:17   0:00 ./collector -d sw -p 1234

So much for not saving my root password. It also shows up in:

    rawxml/getprocesslist.xml
    xml/processlist.xml

Before anyone gets mad at dell: I'm running this in a non standard way, if I used the install.sh the script would not use the -p option, and would instead prompt me for a password (which would not show up in the processlist). Still not sure why it wants me to type the root password when I am already running it as root, but ok.  

Next the collector has straight up copied my:

	boot/grub/grub.conf
	boot/grub/menu.lst
	etc/aliases
	etc/cron* (crontab, crondirs, etc)
	etc/fstab
	etc/host*
	etc/ld.so.conf
	etc/modprobe.conf
	etc/redhat-release
	etc/resolv.conf
	etc/sysctl.conf
	etc/mail/sendmail.cf
	etc/pam.d/* (WAT?)
	etc/sysconfig/* (entire dir and subdirs)
	etc/X11/XF86Config
	lib/modules/current/modules.dep
	proc/ (many files copied here, 778)
	var/log/dmesg
	var/log/messages

This is WAY to much information to send to dell for any reason. How are you sending it to dell as well? I sure wouldn't email it.


### The privacy report

In the privacy report you get a lot less data. The logs directory is now blank, so everything is in the gui directory only. Which means we get to go through some annoying xml/xls (note: I find all xml/xsl annoying (don't ask)). Allot of the same data is gathered, but now just dumped into xml.

We are gathering:

    /boot/grub/grub.conf
    ls /boot
    ls /boot/grub
	cat boot/grub/device.map
    chassis info
	etc/X11/XF86Config
    lsmod
    cat /etc/modprobe.conf
	cat lib/modules/current/modules.dep
    list all rpms / publisher / size / install date, urlinfo, description
    hardware io ranges
    hardware irq info
    cat /proc/meminfo
    cat /etc/fstab
    ifconfig (with ip info and mac addresses "Omitted by user")
    cat resolv.conf (everything is omitted by user but domain is still listed. WAT?)
	runs thru the init.d and does a service status on each
    storage info (df info)
    os version and kernel
    connected usb devices

I like how the kernel version is omitted in the uname but is in the syssumlist. Heh.


### Other options

The collector script has many different options, I don't have any non-production dell gear right now, so I'm not willing to run the hardware report on a server that has hardware to run it on.

I ran the -lg and -ad options in the collector as well, but there was no difference to the sw logs. I imagine this would be different if I was running on an actual dell machine with actual dell hardware instead of the virtual machine that I'm running this on. :)


# Conclusion

I won't be running the dset tool on any production gear because:

* The package installation could cause issues with your system (not staying in /opt/dell, using --nodeps, conflicting sblim package with epel's)
* It wants your root password to be entered at a prompt of a python program, even if you are currently root
* the non privacy report gathers way to much info about your system, under no circumstances should this be sent to anyone ever.
* the password on the report zip is incredible insecure
* add the domain in the resolv.conf to private information, or just don't parse the resolv.conf at all


# So what happened?

I told the dell tech that I couldn't run the dset tool due to dset doing some bad behaviors; but I had a dmidecode, and some megacli logs to send.

After dell reviewed the logs I emailed they shipped me a new battery.


# tldr version

don't run dset. If you absolutely must use dset, use the privacy option.
