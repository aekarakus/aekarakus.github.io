---
title: On My Struggles to Setup a Proxy Repository Manager
layout: post
summary: How I managed to set a proxy repository manager on my home lab
image: cephei.jpg
categories:
- Linux
tags:
- nexus
- jfrog
- repository-management
- archiva
- pulpproject
- docker
author: alperenkarakus_
---

> Every test in this post was carried out with Ubuntu 20.04

Recently I decided to set up a repository manager on my homelab, so that whenever I want to spin up a new virtual machine and run, say, "apt upgrade", I don't have to reach out some remote server and download a 700-ish MBs of recent packages all over again. I chose JFrog Artifactory Community Edition, because I was familiar with it.

With Community Edition, the package flavours are limited and if you want to connect a remote repository to it, then you have to go with the *generic repository* option, and if you're lucky then you can download packages.

By that way I was able to set an apt and a yum repository. All my package downloads began one-time-go's as it would store the packages in local storage, once it was downloaded. This way I could save some time and unnecessary data usage, eliminating the network overhead.
### The Docker Mystery
I was happy and settled, until I wanted to set a docker repository.  I can say that either JFrog Artifactory's Community Edition doesn't let you do that with a generic repository or there aren't enough clear documentation on that specific subject.

After that point, I started looking for some other software that were comparable to JFrog Artifactory.
### Apache Archiva
When I first saw Apache Archiva, I was excited. I set it up, only to see it is rather limited to Java technologies such as Maven and ANT. 
### The Pulp Project
I must admit, this one seems like the most promising solution in the near future. If projects like JFrog and Nexus decides to shut down their community editions or limit the capabilities of community editions, I eventually will have to use this as my repository manager.

Pulp comes along with several installation options, Ansible and PyPI. I've failed both. PyPI installation is harder, because it's all manual from the scratch. You will have to do a seperate database installation yourself, but I couldn't even pass the first steps because of incompatible Python libraries.

I also tried the Ansible installer. This is the most infuriating option IMHO. Instructions is very clear, but even if I switched between Centos and Fedora, there is a certain point in the playbook that sometimes fail, and is guarenteed to fail if you run the playbook multiple times. I checked for that failure on several platforms including Reddit, and saw people struggling on the same point. The playbook was not idempotent, I hope they fix it in a near future, because this was the reason I gave up with their solution.
### Nexus Repository Manager - OSS
I know my research seems desperate, but I had one more solution to try, Nexus Repository Manager OSS.

In  OSS edition, it still had the capability to connect to Docker remote repositories, and again it is one-time-go, once it downloads the packages, they are stored in the local storage. Then if you have to download the package again, it happens in your network with of-course, higher speeds.

You can download the binary packages from [here](https://www.sonatype.com/products/repository-oss-download). All you have to do is to fill up the form. 

The only prequisite is to have jre8 installed on your system. Installation process is rather simple, extract the tar.gz archive.

<div class="message" style="color:red">
Move your extracted directories(there are two directories) to /opt, I only managed to get this software run by this way.
</div>

Then you are good to go. Here are the commands you might run for a basic setup:

```bash
sudo apt update
sudo apt-get install openjdk8-jre
tar -xzvf your_nexus_archive.tar.gz
mv nexus-someversionnumber/ nexus/
sudo mv nexus/ sonatype-work/ /opt/
cd /opt/nexus/bin
./nexus run
```

This  will run a webserver at http://localhost:8081. Connect to that url, and from right top click *sign-in*. This will provide you with the path at where you can find your password for *admin* user.

After you set up your credentials, you can start configuring your repositories at *cogwheel icon-> repository*.

### Setting up a Docker Repository
#### On Nexus
Setting up a proxy Docker repository was my aim all along. First, at *cogwheel icon-> repository->repositories* menu, click *create repository*, then select *docker(proxy)* option. Then do the following:

* Pick a name
* If you want to do a quick test, check HTTP and pick a port._I picked 9000_
* Check *Enable Docker V1 API*
* Enter *https://registry-1.docker.io* as remote url
* Check *Use Docker Hub* as an index

#### On Your Client
On a client running docker, do the following:
* Create the file */etc/docker/daemon.json*
* Write down the following *{ "insecure-registries":["your_nexus_ip:9000"] } *
* Add the line *DOCKER_OPTS="--config-file=/etc/docker/daemon.json"* to */etc/default/docker*
* Stop and start docker service.

Here are the commands:

```bash
echo '{ "insecure-registries":["192.168.57.10:9000"] }' | sudo tee -a /etc/docker/daemon.json
echo 'DOCKER_OPTS="--config-file=/etc/docker/daemon.json"' | sudo tee -a /etc/default/docker
sudo systemctl stop docker && sudo systemctl start docker
```

After this point we are ready to connect to our Nexus proxy docker repository.

```bash
docker login http://your_nexus_ip:9000
# provide the credentials you changed at the nexus installation

docker pull your_nexus_ip:9000/ubuntu:18.04
```

This ubuntu image will be stored in your Nexus instance forever.


![cockpit](https://i.imgur.com/TCZuMse.png)

As a quick test, I removed the image from both my client and Nexus and timed the pull operation. At first time it lasted around 13 seconds, and following pulls took around 2,5 seconds, achieving 5x speed.
