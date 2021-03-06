---
author: Mitchell Anicas
date: 2015-04-29
language: en
license: cc by-nc-sa
source: https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-centos-7
---

# How To Configure BIND as a Private Network DNS Server on CentOS 7

## Introduction

An important part of managing server configuration and infrastructure includes maintaining an easy way to look up network interfaces and IP addresses by name, by setting up a proper Domain Name System (DNS). Using fully qualified domain names (FQDNs), instead of IP addresses, to specify network addresses eases the configuration of services and applications, and increases the maintainability of configuration files. Setting up your own DNS for your private network is a great way to improve the management of your servers.

In this tutorial, we will go over how to set up an internal DNS server, using the BIND name server software (BIND9) on CentOS 7, that can be used by your Virtual Private Servers (VPS) to resolve private host names and private IP addresses. This provides a central way to manage your internal hostnames and private IP addresses, which is indispensable when your environment expands to more than a few hosts.

The Ubuntu version of this tutorial can be found [here](how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-14-04).

## Prerequisites

To complete this tutorial, you will need the following:

- Some servers that are running in the same datacenter and have [private networking enabled](how-to-set-up-and-use-digitalocean-private-networking)
- A new VPS to serve as the Primary DNS server, _ns1_
- Optional: A new VPS to serve as a Secondary DNS server, _ns2_
- Root access to all of the above ([steps 1-4 here](initial-server-setup-with-centos-7))

If you are unfamiliar with DNS concepts, it is recommended that you read at least the first three parts of our [Introduction to Managing DNS](https://www.digitalocean.com/community/tutorial_series/an-introduction-to-managing-dns).

### Example Hosts

For example purposes, we will assume the following:

- We have two existing VPS called “host1” and “host2”
- Both VPS exist in the nyc3 datacenter
- Both VPS have private networking enabled (and are on the 10.128.0.0/16 subnet)
- Both VPS are somehow related to our web application that runs on “example.com”

With these assumptions, we decide that it makes sense to use a naming scheme that uses “nyc3.example.com” to refer to our private subnet or zone. Therefore, _host1_’s private Fully-Qualified Domain Name (FQDN) will be “host1.nyc3.example.com”. Refer to the following table the relevant details:

| Host | Role | Private FQDN | Private IP Address |
| --- | --- | --- | --- |
| host1 | Generic Host 1 | host1.nyc3.example.com | 10.128.100.101 |
| host2 | Generic Host 2 | host2.nyc3.example.com | 10.128.200.102 |

**Note:** Your existing setup will be different, but the example names and IP addresses will be used to demonstrate how to configure a DNS server to provide a functioning internal DNS. You should be able to easily adapt this setup to your own environment by replacing the host names and private IP addresses with your own. It is not necessary to use the region name of the datacenter in your naming scheme, but we use it here to denote that these hosts belong to a particular datacenter’s private network. If you utilize multiple datacenters, you can set up an internal DNS within each respective datacenter.

## Our Goal

By the end of this tutorial, we will have a primary DNS server, _ns1_, and optionally a secondary DNS server, _ns2_, which will serve as a backup.

Here is a table with example names and IP addresses:

| Host | Role | Private FQDN | Private IP Address |
| --- | --- | --- | --- |
| ns1 | Primary DNS Server | ns1.nyc3.example.com | 10.128.10.11 |
| ns2 | Secondary DNS Server | ns2.nyc3.example.com | 10.128.20.12 |

Let’s get started by installing our Primary DNS server, ns1.

## Install BIND on DNS Servers

**Note:** Text that is highlighted in red is important! It will often be used to denote something that needs to be replaced with your own settings or that it should be modified or added to a configuration file. For example, if you see something like host1.nyc3.example.com, replace it with the FQDN of your own server. Likewise, if you see host1\_private\_IP, replace it with the private IP address of your own server.

On both DNS servers, _ns1_ and _ns2_, install BIND with yum:

    sudo yum install bind bind-utils

Confirm the prompt by entering `y`.

Now that BIND is installed, let’s configure the primary DNS server.

## Configure Primary DNS Server

BIND’s configuration consists of multiple files, which are included from the main configuration file, `named.conf`. These filenames begin with “named” because that is the name of the process that BIND runs. We will start with configuring the options file.

### Configure Bind

BIND’s process is known as **named**. As such, many of the files refer to “named” instead of “BIND”.

On _ns1_, open the `named.conf` file for editing:

    sudo vi /etc/named.conf

Above the existing `options` block, create a new ACL block called “trusted”. This is where we will define list of clients that we will allow recursive DNS queries from (i.e. your servers that are in the same datacenter as ns1). Using our example private IP addresses, we will add _ns1_, _ns2_, _host1_, and _host2_ to our list of trusted clients:

/etc/named.conf — 1 of 4

    acl "trusted" {
            10.128.10.11; # ns1 - can be set to localhost
            10.128.20.12; # ns2
            10.128.100.101; # host1
            10.128.200.102; # host2
    };

Now that we have our list of trusted DNS clients, we will want to edit the `options` block. Add the private IP address of ns1 to the `listen-on port 53` directive, and comment out the `listen-on-v6` line:

/etc/named.conf — 2 of 4

    options {
            listen-on port 53 { 127.0.0.1; 10.128.10.11; };
    # listen-on-v6 port 53 { ::1; };
    ...

Below those entries, change the `allow-transfer` directive to from “none” to **ns2** ’s private IP address. Also, change `allow-query` directive from “localhost” to “trusted”:

/etc/named.conf — 3 of 4

    ...
    options {
    ...
            allow-transfer { 10.128.20.12; }; # disable zone transfers by default
    ...
            allow-query { trusted; }; # allows queries from "trusted" clients
    ...

At the end of the file, add the following line:

/etc/named.conf — 4 of 4

    include "/etc/named/named.conf.local";

Now save and exit `named.conf`. The above configuration specifies that only your own servers (the “trusted” ones) will be able to query your DNS server.

Next, we will configure the local file, to specify our DNS zones.

### Configure Local File

On _ns1_, open the `named.conf.local` file for editing:

    sudo vi /etc/named/named.conf.local

The file should be empty. Here, we will specify our forward and reverse zones.

Add the forward zone with the following lines (substitute the zone name with your own):

/etc/named/named.conf.local — 1 of 2

    zone "nyc3.example.com" {
        type master;
        file "/etc/named/zones/db.nyc3.example.com"; # zone file path
    };

Assuming that our private subnet is _10.128.0.0/16_, add the reverse zone by with the following lines (note that our reverse zone name starts with “128.10” which is the octet reversal of “10.128”):

/etc/named/named.conf.local — 2 of 2

    zone "128.10.in-addr.arpa" {
        type master;
        file "/etc/named/zones/db.10.128"; # 10.128.0.0/16 subnet
        };

If your servers span multiple private subnets but are in the same datacenter, be sure to specify an additional zone and zone file for each distinct subnet. When you are finished adding all of your desired zones, save and exit the `named.conf.local` file.

Now that our zones are specified in BIND, we need to create the corresponding forward and reverse zone files.

### Create Forward Zone File

The forward zone file is where we define DNS records for forward DNS lookups. That is, when the DNS receives a name query, “host1.nyc3.example.com” for example, it will look in the forward zone file to resolve _host1_’s corresponding private IP address.

Let’s create the directory where our zone files will reside. According to our _named.conf.local_ configuration, that location should be `/etc/named/zones`:

    sudo chmod 755 /etc/named
    sudo mkdir /etc/named/zones

Now let’s edit our forward zone file:

    sudo vi /etc/named/zones/db.nyc3.example.com

First, you will want to add the SOA record. Replace the highlighted ns1 FQDN with your own FQDN, then replace the second “nyc3.example.com” with your own domain. Every time you edit a zone file, you should increment the _serial_ value before you restart the `named` process–we will increment it to “3”. It should look something like this:

/etc/named/zones/db.nyc3.example.com — 1 of 3

    @ IN SOA ns1.nyc3.example.com. admin.nyc3.example.com. (
                                  3 ; Serial
                 604800 ; Refresh
                  86400 ; Retry
                2419200 ; Expire
                 604800 ) ; Negative Cache TTL

After that, add your nameserver records with the following lines (replace the names with your own). Note that the second column specifies that these are “NS” records:

/etc/named/zones/db.nyc3.example.com — 2 of 3

    ; name servers - NS records
        IN NS ns1.nyc3.example.com.
        IN NS ns2.nyc3.example.com.

Then add the A records for your hosts that belong in this zone. This includes any server whose name we want to end with “.nyc3.example.com” (substitute the names and private IP addresses). Using our example names and private IP addresses, we will add A records for _ns1_, _ns2_, _host1_, and _host2_ like so:

/etc/named/zones/db.nyc3.example.com — 3 of 3

    ; name servers - A records
    ns1.nyc3.example.com. IN A 10.128.10.11
    ns2.nyc3.example.com. IN A 10.128.20.12
    
    ; 10.128.0.0/16 - A records
    host1.nyc3.example.com. IN A 10.128.100.101
    host2.nyc3.example.com. IN A 10.128.200.102

Save and exit the `db.nyc3.example.com` file.

Our final example forward zone file looks like the following:

/etc/named/zones/db.nyc3.example.com — complete

    $TTL 604800
    @ IN SOA ns1.nyc3.example.com. admin.nyc3.example.com. (
                      3 ; Serial
                 604800 ; Refresh
                  86400 ; Retry
                2419200 ; Expire
                 604800 ) ; Negative Cache TTL
    ;
    ; name servers - NS records
         IN NS ns1.nyc3.example.com.
         IN NS ns2.nyc3.example.com.
    
    ; name servers - A records
    ns1.nyc3.example.com. IN A 10.128.10.11
    ns2.nyc3.example.com. IN A 10.128.20.12
    
    ; 10.128.0.0/16 - A records
    host1.nyc3.example.com. IN A 10.128.100.101
    host2.nyc3.example.com. IN A 10.128.200.102

Now let’s move onto the reverse zone file(s).

### Create Reverse Zone File(s)

Reverse zone file are where we define DNS PTR records for reverse DNS lookups. That is, when the DNS receives a query by IP address, “10.128.100.101” for example, it will look in the reverse zone file(s) to resolve the corresponding FQDN, “host1.nyc3.example.com” in this case.

On _ns1_, for each reverse zone specified in the `named.conf.local` file, create a reverse zone file.

Edit the reverse zone file that corresponds to the reverse zone(s) defined in `named.conf.local`:

    sudo vi /etc/named/zones/db.10.128

In the same manner as the forward zone file, replace the highlighted ns1 FQDN with your own FQDN, then replace the second “nyc3.example.com” with your own domain. Every time you edit a zone file, you should increment the _serial_ value before you restart the `named` process–we will increment it to “3”. It should look something like this:

/etc/named/zones/db.10.128 — 1 of 3

    @ IN SOA ns1.nyc3.example.com. admin.nyc3.example.com. (
                                  3 ; Serial
                             604800 ; Refresh
                              86400 ; Retry
                            2419200 ; Expire
                             604800 ) ; Negative Cache TTL

After that, add your nameserver records with the following lines (replace the names with your own). Note that the second column specifies that these are “NS” records:

/etc/named/zones/db.10.128 — 2 of 3

    ; name servers - NS records
          IN NS ns1.nyc3.example.com.
          IN NS ns2.nyc3.example.com.

Then add `PTR` records for all of your servers whose IP addresses are on the subnet of the zone file that you are editing. In our example, this includes all of our hosts because they are all on the 10.128.0.0/16 subnet. Note that the first column consists of the last two octets of your servers’ private IP addresses in reversed order. Be sure to substitute names and private IP addresses to match your servers:

/etc/named/zones/db.10.128 — 3 of 3

    ; PTR Records
    11.10 IN PTR ns1.nyc3.example.com. ; 10.128.10.11
    12.20 IN PTR ns2.nyc3.example.com. ; 10.128.20.12
    101.100 IN PTR host1.nyc3.example.com. ; 10.128.100.101
    102.200 IN PTR host2.nyc3.example.com. ; 10.128.200.102

Save and exit the reverse zone file (repeat this section if you need to add more reverse zone files).

Our final example reverse zone file looks like the following:

/etc/named/zones/db.10.128 — complete

    $TTL 604800
    @ IN SOA nyc3.example.com. admin.nyc3.example.com. (
                                  3 ; Serial
                             604800 ; Refresh
                              86400 ; Retry
                            2419200 ; Expire
                             604800 ) ; Negative Cache TTL
    ; name servers
          IN NS ns1.nyc3.example.com.
          IN NS ns2.nyc3.example.com.
    
    ; PTR Records
    11.10 IN PTR ns1.nyc3.example.com. ; 10.128.10.11
    12.20 IN PTR ns2.nyc3.example.com. ; 10.128.20.12
    101.100 IN PTR host1.nyc3.example.com. ; 10.128.100.101
    102.200 IN PTR host2.nyc3.example.com. ; 10.128.200.102

### Check BIND Configuration Syntax

Run the following command to check the syntax of the `named.conf*` files:

    sudo named-checkconf

If your named configuration files have no syntax errors, you will return to your shell prompt and see no error messages. If there are problems with your configuration files, review the error message and the [Configure Primary DNS Server](how-to-configure-bind-as-an-private-network-dns-server-on-centos-7#ConfigurePrimaryDNSServer) section, then try `named-checkconf` again.

The `named-checkzone` command can be used to check the correctness of your zone files. Its first argument specifies a zone name, and the second argument specifies the corresponding zone file, which are both defined in `named.conf.local`.

For example, to check the “nyc3.example.com” forward zone configuration, run the following command (change the names to match your forward zone and file):

    sudo named-checkzone nyc3.example.com /etc/named/zones/db.nyc3.example.com

And to check the “128.10.in-addr.arpa” reverse zone configuration, run the following command (change the numbers to match your reverse zone and file):

    sudo named-checkzone 128.10.in-addr.arpa /etc/named/zones/db.10.128

When all of your configuration and zone files have no errors in them, you should be ready to restart the BIND service.

### Start BIND

Start BIND:

    sudo systemctl start named

Now you will want to enable it, so it will start on boot:

    sudo systemctl enable named

Your primary DNS server is now setup and ready to respond to DNS queries. Let’s move on to creating the secondary DNS server.

## Configure Secondary DNS Server

In most environments, it is a good idea to set up a secondary DNS server that will respond to requests if the primary becomes unavailable. Luckily, the secondary DNS server is much easier to configure.

On _ns2_, edit the `named.conf` file:

    sudo vi /etc/named.conf

**Note:** If you prefer to skip these instructions, you can copy _ns1_’s `named.conf` file and modify it to listen on _ns2_’s private IP address, and not allow transfers.

Above the existing `options` block, create a new ACL block called “trusted”. This is where we will define list of clients that we will allow recursive DNS queries from (i.e. your servers that are in the same datacenter as ns1). Using our example private IP addresses, we will add _ns1_, _ns2_, _host1_, and _host2_ to our list of trusted clients:

/etc/named.conf — 1 of 4

    acl "trusted" {
            10.128.10.11; # ns1 - can be set to localhost
            10.128.20.12; # ns2
            10.128.100.101; # host1
            10.128.200.102; # host2
    };

Now that we have our list of trusted DNS clients, we will want to edit the `options` block. Add the private IP address of ns1 to the `listen-on port 53` directive, and comment out the `listen-on-v6` line:

/etc/named.conf — 2 of 4

    options {
            listen-on port 53 { 127.0.0.1; 10.128.20.12; };
    # listen-on-v6 port 53 { ::1; };
    ...

Change `allow-query` directive from “localhost” to “trusted”:

/etc/named.conf — 3 of 4

    ...
    options {
    ...
            allow-query { trusted; }; # allows queries from "trusted" clients
    ...

At the end of the file, add the following line:

/etc/named.conf — 4 of 4

    include "/etc/named/named.conf.local";

Now save and exit `named.conf`. The above configuration specifies that only your own servers (the “trusted” ones) will be able to query your DNS server.

Next, we will configure the local file, to specify our DNS zones.

Save and exit `named.conf`.

Now edit the `named.conf.local` file:

    sudo chmod 755 /etc/named
    sudo vi /etc/named/named.conf.local

Define slave zones that correspond to the master zones on the primary DNS server. Note that the type is “slave”, the file does not contain a path, and there is a `masters` directive which should be set to the primary DNS server’s private IP. If you defined multiple reverse zones in the primary DNS server, make sure to add them all here:

/etc/named/named.conf.local

    zone "nyc3.example.com" {
        type slave;
        file "slaves/db.nyc3.example.com";
        masters { 10.128.10.11; }; # ns1 private IP
    };
    
    zone "128.10.in-addr.arpa" {
        type slave;
        file "slaves/db.10.128";
        masters { 10.128.10.11; }; # ns1 private IP
    };

Now save and exit `named.conf.local`.

Run the following command to check the validity of your configuration files:

    sudo named-checkconf

Once that checks out, start BIND:

    sudo systemctl start named

Enable BIND to start on boot:

    sudo systemctl enable named

Now you have primary and secondary DNS servers for private network name and IP address resolution. Now you must configure your servers to use your private DNS servers.

## Configure DNS Clients

Before all of your servers in the “trusted” ACL can query your DNS servers, you must configure each of them to use _ns1_ and _ns2_ as nameservers. This process varies depending on OS, but for most Linux distributions it involves adding your name servers to the `/etc/resolv.conf` file.

### CentOS Clients

On CentOS, RedHat, and Fedora Linux VPS, simply edit the `resolv.conf` file:

    sudo vi /etc/resolv.conf

Then add the following lines to the TOP of the file (substitute your private domain, and _ns1_ and _ns2_ private IP addresses):

/etc/resolv.conf

    search nyc3.example.com # your private domain
    nameserver 10.128.10.11 # ns1 private IP address
    nameserver 10.128.20.12 # ns2 private IP address

Now save and exit. Your client is now configured to use your DNS servers.

### Ubuntu Clients

On Ubuntu and Debian Linux VPS, you can edit the `head` file, which is prepended to `resolv.conf` on boot:

    sudo vi /etc/resolvconf/resolv.conf.d/head

Add the following lines to the file (substitute your private domain, and _ns1_ and _ns2_ private IP addresses):

/etc/resolvconf/resolv.conf.d/head

    search nyc3.example.com # your private domain
    nameserver 10.128.10.11 # ns1 private IP address
    nameserver 10.128.20.12 # ns2 private IP address

Now run `resolvconf` to generate a new `resolv.conf` file:

    sudo resolvconf -u

Your client is now configured to use your DNS servers.

## Test Clients

Use `nslookup`—included in the “bind-utils” package—to test if your clients can query your name servers. You should be able to do this on all of the clients that you have configured and are in the “trusted” ACL.

### Forward Lookup

For example, we can perform a forward lookup to retrieve the IP address of _host1.nyc3.example.com_ by running the following command:

    nslookup host1

Querying “host1” expands to “host1.nyc3.example.com because of the `search` option is set to your private subdomain, and DNS queries will attempt to look on that subdomain before looking for the host elsewhere. The output of the command above would look like the following:

    Output:Server: 10.128.10.11
    Address: 10.128.10.11#53
    
    Name: host1.nyc3.example.com
    Address: 10.128.100.101

### Reverse Lookup

To test the reverse lookup, query the DNS server with _host1_’s private IP address:

    nslookup 10.128.100.101

You should see output that looks like the following:

    Output:Server: 10.128.10.11
    Address: 10.128.10.11#53
    
    11.10.128.10.in-addr.arpa name = host1.nyc3.example.com.

If all of the names and IP addresses resolve to the correct values, that means that your zone files are configured properly. If you receive unexpected values, be sure to review the zone files on your primary DNS server (e.g. `db.nyc3.example.com` and `db.10.128`).

Congratulations! Your internal DNS servers are now set up properly! Now we will cover maintaining your zone records.

## Maintaining DNS Records

Now that you have a working internal DNS, you need to maintain your DNS records so they accurately reflect your server environment.

### Adding Host to DNS

Whenever you add a host to your environment (in the same datacenter), you will want to add it to DNS. Here is a list of steps that you need to take:

#### Primary Nameserver

- Forward zone file: Add an "A” record for the new host, increment the value of “Serial”
- Reverse zone file: Add a “PTR” record for the new host, increment the value of “Serial”
- Add your new host’s private IP address to the “trusted” ACL (`named.conf.options`)

Then reload BIND:

    sudo systemctl reload named

#### Secondary Nameserver

- Add your new host’s private IP address to the “trusted” ACL (`named.conf.options`)

Then reload BIND:

    sudo systemctl reload named

#### Configure New Host to Use Your DNS

- Configure resolv.conf to use your DNS servers
- Test using `nslookup`

### Removing Host from DNS

If you remove a host from your environment or want to just take it out of DNS, just remove all the things that were added when you added the server to DNS (i.e. the reverse of the steps above).

## Conclusion

Now you may refer to your servers’ private network interfaces by name, rather than by IP address. This makes configuration of services and applications easier because you no longer have to remember the private IP addresses, and the files will be easier to read and understand. Also, now you can change your configurations to point to a new servers in a single place, your primary DNS server, instead of having to edit a variety of distributed configuration files, which eases maintenance.

Once you have your internal DNS set up, and your configuration files are using private FQDNs to specify network connections, it is **critical** that your DNS servers are properly maintained. If they both become unavailable, your services and applications that rely on them will cease to function properly. This is why it is recommended to set up your DNS with at least one secondary server, and to maintain working backups of all of them.
