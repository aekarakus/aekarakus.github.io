---
title: Pretty Hostnames
layout: post
categories:
- Linux
tags:
- hostname
- rhel
- pretty-hostname
---

> A shame; the ones who died without a name.
>
> <cite><a href="https://en.wikipedia.org/wiki/Holiday_(Green_Day_song)">Greenday, Holiday(2004)</a></cite>

#### The Need For Names, Why Bother?

A computer is generally known by it's IP address. Here's the IP address of one of the servers of GitHub.

```bash
root@system:~# nslookup github.com
Server:         192.168.233.2
Address:        192.168.233.2#53

Non-authoritative answer:
Name:   github.com
Address: 140.82.121.3

```

That server's IP is 140.82.121.3.  And it's hostname is github.com. We can use either of them to reach that particular server. 


``` bash
root@system:~# ping 140.82.121.3
PING 140.82.121.3 (140.82.121.3) 56(84) bytes of data.
64 bytes from 140.82.121.3: icmp_seq=1 ttl=37 time=47.7 ms
64 bytes from 140.82.121.3: icmp_seq=2 ttl=37 time=65.2 ms
64 bytes from 140.82.121.3: icmp_seq=3 ttl=37 time=51.0 ms
64 bytes from 140.82.121.3: icmp_seq=4 ttl=37 time=50.2 ms
^C
--- 140.82.121.3 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 47.782/53.607/65.290/6.857 ms


root@system:~# ping github.com
PING github.com (140.82.121.3) 56(84) bytes of data.
64 bytes from 140.82.121.3 (140.82.121.3): icmp_seq=1 ttl=37 time=50.6 ms
64 bytes from 140.82.121.3 (140.82.121.3): icmp_seq=2 ttl=37 time=52.0 ms
64 bytes from 140.82.121.3 (140.82.121.3): icmp_seq=3 ttl=37 time=51.3 ms
64 bytes from 140.82.121.3 (140.82.121.3): icmp_seq=4 ttl=37 time=51.4 ms
^C
--- github.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
rtt min/avg/max/mdev = 50.608/51.353/52.012/0.524 ms

```

We can clearly see, while providing uniqueness, the IP addresses are hard to memorize. They are more machine-related, and difficult for us to remember.  Imagine calling your friends by their social security number, instead of their names. You'd be a lone-wolf without a single friend eventually.

For that reason, we need hostnames. They helps us build specific domains, which defines an organization or just the kind of work a bunch server does. (like host1.myorg.io or worker9.lab)

By this way, we reach them by their hostnames easily, without having to know all the IP addresses.

#### Getting and Setting a Hostname

Starting with RHEL 7, hostnamectl command is introduced. We can reach detailed info about our server and we can set hostname through this command. Here's it in action.

``` bash
[root@system /]# hostnamectl
   Static hostname: system.example.com
         Icon name: computer-vm
           Chassis: vm
        Machine ID: ba239dbb284441f18345412810ccf798
           Boot ID: d0d48f7735e542d18cf3a863a5b7b0ba
    Virtualization: vmware
  Operating System: CentOS Linux 8 (Core)
       CPE OS Name: cpe:/o:centos:centos:8
            Kernel: Linux 4.18.0-193.el8.x86_64
      Architecture: x86-64
```

Along with general information of the system, we can see my machine has the hostname "system.example.com". To change this hostname, with command *set-hosname*.

``` bash
[root@system /]# hostnamectl set-hostname worker.example.com
[root@system /]# hostnamectl
   Static hostname: worker.example.com
         Icon name: computer-vm
           Chassis: vm
        Machine ID: ba239dbb284441f18345412810ccf798
           Boot ID: d0d48f7735e542d18cf3a863a5b7b0ba
    Virtualization: vmware
  Operating System: CentOS Linux 8 (Core)
       CPE OS Name: cpe:/o:centos:centos:8
            Kernel: Linux 4.18.0-193.el8.x86_64
      Architecture: x86-64
```

This command automatically changes */etc/hostname* file on action.

We can check for our hostname with *hostname* command also.

``` bash
[root@system /]# hostname
worker.example.com
```

#### Setting a Pretty Hostname

Searching through the man page of *hostnamectl*, I've found an interesting  feature called *pretty hostname*. It's a kind of high-level hostname, used by some applications. It's only available to humans and not the machine. Any UTF-8 string is valid. Let's set a pretty hostname with a fancy graduation cap.

``` bash 
[root@system /]# hostnamectl --pretty set-hostname "$(perl -CO -le 'print "\x{1f393}"')"
[root@system /]# hostnamectl
   Static hostname: worker.example.com
   Pretty hostname: 🎓
         Icon name: computer-vm
           Chassis: vm
        Machine ID: ba239dbb284441f18345412810ccf798
           Boot ID: d0d48f7735e542d18cf3a863a5b7b0ba
    Virtualization: vmware
  Operating System: CentOS Linux 8 (Core)
       CPE OS Name: cpe:/o:centos:centos:8
            Kernel: Linux 4.18.0-193.el8.x86_64
      Architecture: x86-64
```

Let's see an application using it.  This is my server's cockpit page. Now, isn't this pretty?

![cockpit](https://i.ibb.co/2sY0yND/cockpit.png)

#### Sources

[Stackoverflow](https://unix.stackexchange.com/questions/357920/what-is-different-between-static-hostname-and-icon-name-and-pretty-host-name-in)
