---
author: Vadym Kalsin
date: 2019-03-15
language: en
license: cc by-nc-sa
source: https://www.digitalocean.com/community/tutorials/how-to-gather-infrastructure-metrics-with-metricbeat-on-ubuntu-18-04
---

# How To Gather Infrastructure Metrics with Metricbeat on Ubuntu 18.04

_The author selected the [Computer History Museum](https://www.brightfunds.org/organizations/computer-history-museum) to receive a donation as part of the [Write for DOnations](https://do.co/w4do-cta) program._

## Introduction

[Metricbeat](https://www.elastic.co/products/beats/metricbeat), which is one of several [Beats](https://www.elastic.co/products/beats) that helps send various types of server data to an [Elastic Stack](https://www.elastic.co/products/) server, is a lightweight data shipper that, once installed on your servers, periodically collects system-wide and per-process CPU and memory statistics and sends the data directly to your Elasticsearch deployment. This shipper replaces the earlier [Topbeat](https://www.elastic.co/guide/en/beats/topbeat/current/_overview.html) in version 5.0 of the Elastic Stack.

Other Beats currently available from Elastic are:

- [Filebeat](https://www.elastic.co/products/beats/filebeat): collects and ships log files.
- [Packetbeat](https://www.elastic.co/products/beats/packetbeat): collects and analyzes network data.
- [Winlogbeat](https://www.elastic.co/products/beats/winlogbeat): collects Windows event logs.
- [Auditbeat](https://www.elastic.co/products/beats/auditbeat): collects Linux audit framework data and monitors file integrity.
- [Heartbeat](https://www.elastic.co/products/beats/heartbeat): monitors services for their availability with active probing.

In this tutorial, you will use Metricbeat to forward local system metrics like CPU/memory/disk usage and network utilization from an Ubuntu 18.04 server to another server of the same kind with the Elastic Stack installed. With this shipper, you will gather the basic metrics that you need to get the current state of your server.

## Prerequisites

To follow this tutorial, you will need:

- Two Ubuntu 18.04 servers set up by following the [Initial Server Setup Guide for Ubuntu 18.04](initial-server-setup-with-ubuntu-18-04), including a non-root user with sudo privileges and a firewall configured with `ufw`. On one server, you will download the Elastic Stack; this tutorial will refer to this as the “Elastic Stack server.” The Elastic Stack server will monitor your second server; this second server will be referred to as the “second Ubuntu server.”
- The Elastic Stack installed on the Elastic Stack server by following the tutorial [How To Install Elasticsearch, Logstash, and Kibana (Elastic Stack) on Ubuntu 18.04](how-to-install-elasticsearch-logstash-and-kibana-elastic-stack-on-ubuntu-18-04).

**Note** : When installing the Elastic Stack, you must use the same version across the entire stack. In this tutorial, you will use the latest versions of the entire stack which are, at the time of this writing, Elasticsearch 6.6.2, Kibana 6.6.2, Logstash 6.6.2, and Metricbeat 6.6.2.

## Step 1 — Configuring Elasticsearch to Listen for Traffic on an External IP

The tutorial [How To Install Elasticsearch, Logstash, and Kibana (Elastic Stack) on Ubuntu 18.04](how-to-install-elasticsearch-logstash-and-kibana-elastic-stack-on-ubuntu-18-04) restricted Elasticsearch access to the localhost only. In practice, this is rare, since you will often need to monitor many hosts. In this step, you will configure the Elastic Stack components to interact with the external IP address.

Log in to your Elastic Stack server as your non-root user:

    ssh sammy@Elastic_Stack_server_ip

Use your preferred text editor to edit Elasticsearch’s main configuration file, `elasticsearch.yml`. This tutorial will use `nano`:

    sudo nano /etc/elasticsearch/elasticsearch.yml

Find the following section and modify it so that Elasticsearch listens on all interfaces:

/etc/elasticsearch/elasticsearch.yml

    . . .
    network.host: 0.0.0.0
    . . .

The address `0.0.0.0` is assigned specific meanings in a number of contexts. In this case, `0.0.0.0` means “any IPv4 address at all.”

Save and close `elasticsearch.yml` by pressing `CTRL+X`, followed by `Y` and then `ENTER` if you’re using `nano`. Then, restart the Elasticsearch service with `systemctl` to apply new settings:

    sudo systemctl restart elasticsearch

Now, allow access to the Elasticsearch port from your second Ubuntu server. You will use `ufw` for this:

    sudo ufw allow from second_ubuntu_server_ip/32 to any port 9200

Repeat this command for each of your servers if you have more than two. If your servers are on the same [network](https://en.wikipedia.org/wiki/Subnetwork), you can allow access using one rule for all hosts on the network. To do this, you need to replace the prefix `/32` with a lower value, for example `/24`. You can find more examples of UFW setups in the [UFW Essentials: Common Firewall Rules and Commands](ufw-essentials-common-firewall-rules-and-commands) tutorial.

Next, test the connection. Log in to your second Ubuntu server as your non-root user:

    ssh sammy@second_ubuntu_server_ip

Use the `telnet` command to test the connection to the Elastic Stack server. This command enables communication with another host using the [Telnet](https://en.wikipedia.org/wiki/Telnet) protocol and can check the availability of a port on a remote system.

    telnet Elastic_Stack_server_ip 9200

You’ll receive the following output:

    OutputTrying Elastic_Stack_server_ip...
    Connected to Elastic_Stack_server_ip.
    Escape character is '^]'.

Close the Telnet connection by pressing `CTRL+]`, followed by `CTRL+d`. You can type `quit` and then press `ENTER` to exit the Telnet utility.

Now you are ready to send metrics to your Elastic Stack server.

## Step 2 — Installing and Configuring Metricbeat on the Elastic Stack Server

In the next two steps, you will first install Metricbeat on the Elastic Stack server and import all the needed data, then install and configure the client on the second Ubuntu server.

Log in to your Elastic Stack server as your non-root user:

    ssh sammy@Elastic_Stack_server_ip

Since you previously set up the Elasticsearch repositories in the prerequisite, you only need to install Metricbeat:

    sudo apt install metricbeat

Once Metricbeat is finished installing, load the index template into Elasticsearch. An [_Elasticsearch index_](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html) is a collection of documents that have similar characteristics. Specific names identify each index, which Elasticsearch will use to refer to the indexes when performing various operations. Your Elasticsearch server will automatically apply the index template when you create a new index.

To load the template, use the following command:

    sudo metricbeat setup --template -E 'output.elasticsearch.hosts=["localhost:9200"]'

You will see the following output:

    OutputLoaded index template

Metricbeat comes packaged with example Kibana dashboards, visualizations, and searches for visualizing Metricbeat data in Kibana. Before you can use the dashboards, you need to create the index pattern and load the dashboards into Kibana.

To load the templates, use the following command:

    sudo metricbeat setup -e -E output.elasticsearch.hosts=['localhost:9200'] -E setup.kibana.host=localhost:5601

You will see output that looks like this:

    Output. . .
    2019-02-15T09:51:32.096Z INFO instance/beat.go:281 Setup Beat: metricbeat; Version: 6.6.2
    2019-02-15T09:51:32.136Z INFO add_cloud_metadata/add_cloud_metadata.go:323 add_cloud_metadata: hosting provider type detected as digitalocean, metadata={"instance_id":"133130541","provider":"digitalocean","region":"fra1"}
    2019-02-15T09:51:32.137Z INFO elasticsearch/client.go:165 Elasticsearch url: http://localhost:9200
    2019-02-15T09:51:32.137Z INFO [publisher] pipeline/module.go:110 Beat name: elastic
    2019-02-15T09:51:32.138Z INFO elasticsearch/client.go:165 Elasticsearch url: http://localhost:9200
    2019-02-15T09:51:32.140Z INFO elasticsearch/client.go:721 Connected to Elasticsearch version 6.6.2
    2019-02-15T09:51:32.148Z INFO template/load.go:130 Template already exists and will not be overwritten.
    2019-02-15T09:51:32.148Z INFO instance/beat.go:894 Template successfully loaded.
    Loaded index template
    Loading dashboards (Kibana must be running and reachable)
    2019-02-15T09:51:32.149Z INFO elasticsearch/client.go:165 Elasticsearch url: http://localhost:9200
    2019-02-15T09:51:32.150Z INFO elasticsearch/client.go:721 Connected to Elasticsearch version 6.6.2
    2019-02-15T09:51:32.151Z INFO kibana/client.go:118 Kibana url: http://localhost:5601
    2019-02-15T09:51:56.209Z INFO instance/beat.go:741 Kibana dashboards successfully loaded.
    Loaded dashboards

Now you can start and enable Metricbeat:

    sudo systemctl start metricbeat
    sudo systemctl enable metricbeat

Metricbeat will begin shipping your system stats into Elasticsearch.

To verify that Elasticsearch is indeed receiving this data, query the Metricbeat index with this command:

    curl -XGET 'http://localhost:9200/metricbeat-*/_search?pretty'

You will see an output that looks similar to this:

    Output...
    {
      "took" : 3,
      "timed_out" : false,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : 108,
        "max_score" : 1.0,
        "hits" : [
          {
            "_index" : "metricbeat-6.6.2-2019.02.15",
            "_type" : "doc",
            "_id" : "A4mU8GgBKrpxEYMLjJZt",
            "_score" : 1.0,
            "_source" : {
              "@timestamp" : "2019-02-15T09:54:52.481Z",
              "metricset" : {
                "name" : "network",
                "module" : "system",
                "rtt" : 125
              },
              "event" : {
                "dataset" : "system.network",
                "duration" : 125260
              },
              "system" : {
                "network" : {
                  "in" : {
                    "packets" : 59728,
                    "errors" : 0,
                    "dropped" : 0,
                    "bytes" : 736491211
                  },
                  "out" : {
                    "dropped" : 0,
                    "packets" : 31630,
                    "bytes" : 8283069,
                    "errors" : 0
                  },
                  "name" : "eth0"
                }
              },
              "beat" : {
                "version" : "6.6.2",
                "name" : "elastic",
                "hostname" : "elastic"
              },
    ...

The line `"total" : 108,` indicates that Metricbeat has found 108 search results for this specific metric. If your output shows 0 total hits, you will need to review your setup for errors. If you received the expected output, continue to the next step, in which you will install Metricbeat on the second Ubuntu server.

## Step 3 — Installing and Configuring Metricbeat on the Second Ubuntu Server

Perform this step on all Ubuntu servers from which you want to send metrics to your Elastic Stack server.

Log in to your second Ubuntu server as your non-root user:

    ssh sammy@second_ubuntu_server_ip

The Elastic Stack components are not available in Ubuntu’s default package repositories. However, you can install them with APT after adding Elastic’s package source list.

All of the Elastic Stack’s packages are signed with the Elasticsearch signing key in order to protect your system from package spoofing. Your package manager will trust packages that have been authenticated using the key. In this step, you will import the Elasticsearch public GPG key and add the Elastic package source list in order to install Metricbeat.

To begin, run the following command to import the Elasticsearch public GPG key into APT:

    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

Next, add the Elastic source list to the `sources.list.d` directory, where APT will look for new sources:

    echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list

Next, update your package lists so APT will read the new Elastic source:

    sudo apt update

Then install Metricbeat with this command:

    sudo apt install metricbeat

Once Metricbeat is finished installing, configure it to connect to Elasticsearch. Open its configuration file, `metricbeat.yml`:

    sudo nano /etc/metricbeat/metricbeat.yml

**Note:** Metricbeat’s configuration file is in YAML format, which means that indentation is very important! Be sure that you do not add any extra spaces as you edit this file.

Metricbeat supports numerous outputs, but you’ll usually only send events directly to Elasticsearch or to Logstash for additional processing. Find the following section and update the IP address:

/etc/metricbeat/metricbeat.yml

    #-------------------------- Elasticsearch output ------------------------------
    output.elasticsearch:
      # Array of hosts to connect to.
      hosts: ["Elastic_Stack_server_ip:9200"]
    
    ...

Save and close the file.

You can extend the functionality of Metricbeat with [modules](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-modules.html). In this tutorial, you will use the [`system`](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-module-system.html) module, which allows you to monitor your server’s stats like CPU/memory/disk usage and network utilization.

In this case, the `system` module is enabled by default. You can see a list of enabled and disabled modules by running:

    sudo metricbeat modules list

You will see a list similar to the following:

    OutputEnabled:
    system
    
    Disabled:
    aerospike
    apache
    ceph
    couchbase
    docker
    dropwizard
    elasticsearch
    envoyproxy
    etcd
    golang
    graphite
    haproxy
    http
    jolokia
    kafka
    kibana
    kubernetes
    kvm
    logstash
    memcached
    mongodb
    munin
    mysql
    nginx
    php_fpm
    postgresql
    prometheus
    rabbitmq
    redis
    traefik
    uwsgi
    vsphere
    windows
    zookeeper

You can see the parameters of the module in the `/etc/metricbeat/modules.d/system.yml` configuration file. In the case of this tutorial, you do not need to change anything in the configuration. The default metricsets are `cpu`, `load`, `memory`, `network`, `process`, and `process_summary`. Each module has one or more metricset. A metricset is the part of the module that fetches and structures the data. Rather than collecting each metric as a separate event, metricsets retrieve a list of multiple related metrics in a single request to the remote system.

Now you can start and enable Metricbeat:

    sudo systemctl start metricbeat
    sudo systemctl enable metricbeat

You need to repeat this step on all servers where you want to collect metrics. After that, you can proceed to the next step in which you will see how to navigate through some of Kibana’s dashboards.

## Step 4 — Exploring Kibana Dashboards

In this step, you will take a look at Kibana, the web interface that you installed in the Prerequisites section.

In a web browser, go to the FQDN or public IP address of your Elastic Stack server. After entering the login credentials you defined in Step 2 of [the Elastic Stack tutorial](how-to-install-elasticsearch-logstash-and-kibana-elastic-stack-on-ubuntu-18-04), you will see the Kibana homepage:

![Kibana Homepage](https://raw.githubusercontent.com/opendocs-md/do-tutorials-images/master/img/cart_64850/Metricbeat_Kibana_Landing_Page.png)

Click the **Discover** link in the left-hand navigation bar. On the **Discover** page, select the predefined **meticbeat-** \* index pattern to see Metricbeat data. By default, this will show you all of the log data over the last 15 minutes. You will find a histogram and some metric details:

![Discover page](https://raw.githubusercontent.com/opendocs-md/do-tutorials-images/master/img/cart_64850/Metribeat_Data.png)

Here, you can search and browse through your metrics and also customize your dashboard. At this point, though, there won’t be much in there because you are only gathering system stats from your servers.

Use the left-hand panel to navigate to the **Dashboard** page and search for the **Metricbeat System** dashboard. Once there, you can search for the sample dashboards that come with Metricbeat’s `system` module.

For example, you can view brief information about all your hosts:

![Syslog Dashboard](https://raw.githubusercontent.com/opendocs-md/do-tutorials-images/master/img/cart_64850/Metricbeat_Metrics_Overview.png)

You can also click on the host name and view the detailed information:

![Sudo Dashboard](https://raw.githubusercontent.com/opendocs-md/do-tutorials-images/master/img/cart_64850/Metricbeat_Host_Metrics.png)

Kibana has many other features, such as graphing and filtering, so feel free to explore.

## Conclusion

In this tutorial, you’ve installed Metricbeat and configured the Elastic Stack to collect and analyze system metrics. Metricbeat comes with internal [modules](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-modules.html) that collect metrics from services like Apache, Nginx, Docker, MySQL, PostgreSQL, and more. Now you can collect and analyze the metrics of your applications by simply turning on the modules you need.

If you want to understand more about server monitoring, check out [An Introduction to Metrics, Monitoring, and Alerting](an-introduction-to-metrics-monitoring-and-alerting) and [Putting Monitoring and Alerting into Practice](putting-monitoring-and-alerting-into-practice).
