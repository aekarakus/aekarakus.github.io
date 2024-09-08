---
title: TCP Load Balancing with NGINX
layout: post
---

Since version 1.9.0 nginx supports TCP load balancing. I decided to give it a try while I was working on a high available WireGuard setup. For learning purposes I didn't use WireGuard at first, I am still working on it, and I think even though nginx proved successful in basic TCP load balancing, I am not quite sure that it will work for a WireGuard setup, cause WireGuard works with UDP, and then there are inter-peer key sharing mechanism, which is another story and might not get along with a load balancing layer well. So, let's dive into basic TCP load balancing setup.
### A Basic Setup
#### Environment
- 3 Ubuntu 22.04 servers
- 1 server running nginx
- 3 server running PostgreSQL
- proper entries in hosts file or a DNS server

Here's the **/etc/hosts** file on all hosts.

```bash
127.0.0.1	localhost

# The following lines are desirable for IPv6 capable hosts
::1	ip6-localhost	ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
ff02::3	ip6-allhosts
127.0.1.1	ubuntu-jammy	ubuntu-jammy

192.168.77.81 lb01.example.com lb01
192.168.77.82 postgresql01.example.com postgresql01
192.168.77.83 postgresql02.example.com postgresql02
```

First, let's assume that we can load balance raw TCP request without any real backend. NGINX does this with a *stream* directive. Inside it, we declare classic upstream group and server definitions. This configuration goes into **/etc/nginx/nginx.conf**. In this configuration file I will basically say, "Forward any TCP request to port 8080, to port 3000 of the two backend servers".

```bash
stream {
	upstream backend {
		server 192.168.77.82:3000;
		server 192.168.77.83:3000;
	}
	server {
		listen 8080;
		proxy_pass backend;
	}
}
// the rest of config
```

The default load balancing algorithm of NGINX is *round-robin*. So, two consecutive TCP request to port 8080 on machine *lb1* should hit *postgresql01* and *postgresql02*, or *postgresql01* and *postgresql02*, in order.
#### Proof of Concept
We need to open port 3000 on both backend servers. We can utilize *nc* command to achieve this goal. This will open a port for TCP requests and when a client connects to it, they will interact with each other via stdin. So when a machine will send a string via stdin, the other should see it and vice versa. We can facilitate piping an *ls* command to print the content of the current working directory to the client's stdout. Let's create two different files in each backend servers and start listening on port 3000.

```bash
# postgresql01
touch server01
ls | nc -l 3000

# postgresql01
touch server02
ls | nc -l 3000
```

We will hit port 8080 on localhost two times and should see different results of files on current working directory on both server. Let's finally prove the concept.
![]({{ 'demo 1.gif' | relative_url }})
### A Not So Basic Setup
*nc* is fun and all but let's actually prove the point with a realistic example. All of our servers are already running PostgreSQL. We can change our config such that connections on localhost will hit the two database systems in order. Let's first change our config. **Do not forget to restart nginx service after changing this configuration file**.
```bash
stream {
	upstream backend {
		server 192.168.77.82:5432;
		server 192.168.77.83:5432;
	}
	server {
		listen 8080;
		proxy_pass backend;
	}
}
// the rest of config
```
#### Preparations
We will use user *postgres* and default schema of *postgres*. In order to get access to the database systems, first we have to set a password for *postgres* user. Execute the following on all hosts, including lb01.
```bash
sudo -u postgres psql
postgres=# ALTER USER postgres PASSWORD 'password';
```
Also, we have to change **pg_hba.conf** file so that the connection can be established from our external IP range. 
```bash
host	all		all		192.168.77.1/24    md5
```
Additionally, we have to specify the NICs that the PostgreSQL service will listen on. The default is the loop back adapter. Instead, if we want to have connection from remote addresses, we have to listen on additional NICs. Here I chose all NICs. Add the following to **postgresql.conf**. **Do not forget to restart postgresql service after changing this configuration file**.
```bash
listen_addresses = '*'
```
Now we can create a table called *users* and insert distinguishable records on them on two different database systems.
```bash
#postgresql01
postgres=# CREATE TABLE users (id int8);
postgres=# INSERT INTO TABLE users VALUES(1);

#postgresql02
postgres=# CREATE TABLE users (id int8);
postgres=# INSERT INTO TABLE users VALUES(5);
```

Now we need two different PostgreSQL clients. Because running o the same client didn't give reliable results on  my tests, for they manage sessions somewhat sticky. So, in my opinion, the easiest way to simulate two genuine database handles is to use different clients. I chose *pgAdmin* and *dbeaver*. First add the first client you choose the connection *jdbc:postgresql://192.168.77.81:8080/postgres*. Issue a select on *users* table. Here's my results

![]({{ '2024-09-08_23-56.png' | relative_url }})

Then *close this connection*, and add the same connection to the second client. Issue the same query, and you should see a different result, proving that NGINX is load balancing between different PostgreSQL servers.
![]({{ '2024-09-09_00-02.png' | relative_url }})
