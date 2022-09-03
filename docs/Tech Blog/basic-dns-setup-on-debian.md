# __Basic DNS Setup on Debian using Bind__

There are situations when we want to run a local POC/Demo at our local environment and a DNS server is required. Most of times, a simple change on __/etc/hosts__ file is enough to simulate a network, but other cases will really require you to have a DNS running.

There are many options to handle a DNS server, even running it from a docker container. 

In my case, I have a "utils" Virtual Machine where common services are performed from this host, as well a __DNS Server__.

I also wanted the correct name for a local dns server, then I found [RFC8375](https://www.rfc-editor.org/rfc/rfc8375.html) which contains exactly the information I was looking for. 

So, my local network name is called __home.arpa__ and my local router is setup with a /22 network class, IP range from 192.168.12.1 to 192.168.15.255. In this way, I can have some kind of playground for my local tests. 


This post will contains the modifications required in order to have this "setup" in a local Virtual Machine.


## __Basic Info__

__OS__: Debian 11

__DNS Server__: Bind (a.k.a. Named)

__Local Zone__: home.arpa

__Network__: /22 - 192.168.12.1 to 192.168.15.255

__DNS Server IP__: 192.168.15.205


## __Setup__

The very first step is to install Debian 11 in your virtualization app. 

### Install basic OS Package and Bind9
Bind9 is the DNS Server we'll be using.
```
apt install sudo net-tools mlocate bind9 bind9utils vim
```

### Setup Static IP
Below setup will disable IPv6.
```
vi /etc/network/interfaces

# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
#allow-hotplug enp0s3
#iface enp0s3 inet dhcp

auto enp0s3
iface enp0s3 inet static
 address 192.168.15.205
 netmask 255.255.252.0
 gateway 192.168.15.1
 dns-domain home.arpa
 dns-nameservers 192.168.15.205

# This is an autoconfigured IPv6 interface
# iface enp0s3 inet6 auto
```

Restart the network service and re-connect to the server.
```
systemctl restart networking
```

### Adjust /etc/resolv.conf
In my case, I wanted the "DNS" Server to first search itself and then, if not found, go ahead and search on google dns (8.8.8.8).

```
cat /etc/resolv.conf
nameserver 192.168.15.205
nameserver 8.8.8.8
```

### Create your DNS Local Zone (home.arpa)
To do so, few steps are required since the original folders doesn't exist by default.

```
mkdir -p /var/lib/bind
chown root:bind /var/lib/bind*
touch home.arpa.db                  # this will contain the entries for domain home.arpa
touch 15.168.192.home.arpa.db       # this will contain the reverse DNS entries for home.arpa
```

Once ready, go ahead and add your entries:

```
cat /var/lib/bind/home.arpa.db


$TTL	3600

@	IN	SOA	dns.home.arpa. root.home.arpa. (
			1	    ; serial
			3600	; refresh 1h
			600	    ; retry 10min
			86400	; expire 1day
			600	    ; negative cache ttl 1h
		);

@		IN	NS	dns.home.arpa.

dns		    IN	A	192.168.15.205
k8master	IN	A	192.168.15.210
k8node1		IN	A	192.168.15.211
k8node2		IN	A	192.168.15.212
```

Reverse dns zone:
```
cat 15.168.192.home.arpa.db
@	IN	SOA	dns.home.arpa. root.home.arpa. (
			1	    ; serial
			3600	; refresh 1h
			600	    ; retry 10min
			86400	; expire 1day
			600	    ; negative cache ttl 1h
		);

@	    IN	NS	dns.home.arpa.

205	    IN	PTR	dns.home.arpa.
210	    IN	PTR	k8master.home.arpa.
211	    IN	PTR	k8node1.home.arpa.
212	    IN	PTR	k8node2.home.arpa.
```


### Modify the named.conf files
In debian, the __named.conf__ file comes inside __/etc/bind/__ folder.

```
named.conf
named.conf.local
named.conf.log
named.conf.options
zones.rfc1918
```

__named.conf__

```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.log";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
```

__named.conf.local__

```
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "home.arpa" in {
	type master;
	file "/var/lib/bind/home.arpa.db";
};

zone "15.168.192.in-addr.arpa" in {
	type master;
	file "/var/lib/bind/15.168.192.home.arpa.db";
};
```

__named.conf.options__

```
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "home.arpa" in {
	type master;
	file "/var/lib/bind/home.arpa.db";
};

zone "15.168.192.in-addr.arpa" in {
	type master;
	file "/var/lib/bind/15.168.192.home.arpa.db";
};
root@dns:/etc/bind# cat named.conf.options
options {
	directory "/var/cache/bind";

	// If there is a firewall between you and nameservers you want
	// to talk to, you may need to fix the firewall to allow multiple
	// ports to talk.  See http://www.kb.cert.org/vuls/id/800113

	// If your ISP provided one or more IP addresses for stable
	// nameservers, you probably want to use them as forwarders.
	// Uncomment the following block, and insert the addresses replacing
	// the all-0's placeholder.

	// forwarders {
	// 	0.0.0.0;
	// };

	//========================================================================
	// If BIND logs error messages about the root key being expired,
	// you will need to update your keys.  See https://www.isc.org/bind-keys
	//========================================================================
	dnssec-validation auto;

	listen-on { 127.0.0.1; 192.168.15.205; };
	listen-on-v6 { none; };

	allow-recursion { any; };

	version none;
};
```


__named.conf.log__
This is an additional file in order to record logs from our dns server. 

It will be required to create folders , files and adjust their permissions in the filesystem as well in the __apparmor__ to avoid lack of privileges and security.

```
# create log folder and files
mkdir -p /var/log/bind
touch /var/log/bind.log
touch /var/log/security_info.log
touch /var/log/update_debug.log
chown root:bind /var/log/bind/*
```

Finally, create __named.conf.log__

```
logging {
        channel update_debug {
                file "/var/log/bind/update_debug.log" versions 3 size 100k;
                severity debug;
                print-severity  yes;
                print-time      yes;
        };
        channel security_info {
                file "/var/log/bind/security_info.log" versions 1 size 100k;
                severity info;
                print-severity  yes;
                print-time      yes;
        };
        channel bind_log {
                file "/var/log/bind/bind.log" versions 3 size 1m;
                severity info;
                print-category  yes;
                print-severity  yes;
                print-time      yes;
        };

        category default { bind_log; };
        category lame-servers { null; };
        category update { update_debug; };
        category update-security { update_debug; };
        category security { security_info; };
};
```


__zones.rfc1918__
In this file, we'll need to comment out the entry related to the network 192.168.* .

If that's not your network range, then probably doesn't need to change anything.

```
//zone "168.192.in-addr.arpa" { type master; file "/etc/bind/db.empty"; };
```


### Start services
Once everything is setup, time to start the services and check for logs.

```
systemctl restart named
systemctl status named
```

If any errors occur, you can monitor those errors by running:

```
journalctl |grep named
```


You can validate if your DNS Server is resolving properly:

```
nslookup k8master.home.arpa
nslookup 192.168.15.210
```