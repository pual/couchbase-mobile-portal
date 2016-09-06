---
id: sg
title: Sync Gateway
permalink: ready/installation/sync-gateway/index.html
---

Install Sync Gateway on premise or on a cloud provider. You can download Sync Gateway from the [Couchbase download page](http://www.couchbase.com/nosql-databases/downloads#couchbase-mobile) or download it directly to a Linux system by using the `wget` or `curl` command.

```bash
wget http://latestbuilds.hq.couchbase.com/couchbase-sync-gateway/1.3.0/1.3.0-274/couchbase-sync-gateway-community_1.3.0-274_x86_64.deb
```

All downloads follow the naming convention:

```bash
couchbase-sync-gateway-community_<REL>-<BUILDNUM><ARCH>.deb
```

Where 

- `REL` is the release number.
- `BUILDNUM` is the specific build number of the release.
- `ARCH` is the target architecture of the installer.

## Requirements

|Ubuntu|CentOS/RedHat|Debian|Windows|macOS|
|:-----|:------------|:-----|:------|:----|
|12, 14|5, 6, 7|8|Windows 8, Windows 10, Windows Server 2012|Yosemite, El Capitan|

### Network configuration

Sync Gateway uses specific ports for communication with the outside world, mostly Couchbase Lite databases replicating to and from Sync Gateway. The following table lists the ports used for different types of Sync Gateway network communication:

|Port|Description|
|:---|:----------|
|4984|Public port. External HTTP port used for replication with Couchbase Lite databases and other applications accessing the REST API on the Internet.|
|4985|Admin port. Internal HTTP port for unrestricted access to the database and to run administrative tasks.|

Once you have downloaded Sync Gateway on the distribution of your choice you are ready to install and start it as a service.

## Ubuntu

Install sync_gateway with the dpkg package manager e.g:

```bash
dpkg -i couchbase-sync-gateway-community_1.3.0-274_x86_64.deb
```

When the installation is complete sync_gateway will be running as a service.

```bash
service sync_gateway start
service sync_gateway stop
```

The config file and logs are located in `/home/sync_gateway`.

> **Note:** You can also run the **sync_gateway** binary directly from the command line. The binary is installed at `/opt/couchbase-sync-gateway/bin/sync_gateway`.

## Red Hat/CentOS

Install sync_gateway with the rpm package manager e.g:

```bash
rpm -i couchbase-sync-gateway-community_1.3.0-274_x86_64.rpm
```

When the installation is complete sync_gateway will be running as a service.

On CentOS 5:

```bash
service sync_gateway start
service sync_gateway stop
```

On CentOS 6:

```bash
initctl start sync_gateway
initctl stop sync_gateway
```

On CentOS 7:

```bash
systemctl start sync_gateway
systemctl stop sync_gateway
```

The config file and logs are located in `/home/sync_gateway`.

## Debian

Install sync_gateway with the dpkg package manager e.g:

```bash
dpkg -i couchbase-sync-gateway-community_1.3.0-274_x86_64.deb
```

When the installation is complete sync_gateway will be running as a service.

```bash
systemctl start sync_gateway
systemctl stop sync_gateway
```

The config file and logs are located in `/home/sync_gateway`.

## Windows

Install sync_gateway on Windows by running the .exe file from the desktop.

```bash
couchbase-sync-gateway-community_1.3.0-274_x86_64.exe
```

When the installation is complete sync_gateway will be installed as a service but not running.

Use the **Control Panel --> Admin Tools --> Services** to stop/start the service.

The config file and logs are located in ``.

## macOS

Install sync_gateway by unpacking the tar.gz installer.

```bash
sudo tar --zxvf couchbase-sync-gateway-community_1.3.0-274_x86_64.tar.gz --directory /opt
```

Create the sync_gateway service.

```bash
$ cd /opt/couchbase-sync-gateway/service

$ sudo ./sync_gateway_service_install.sh
```

To restart sync_gateway (it will automatically start again).

```bash
$ sudo launchctl stop sync_gateway
```

To remove the service.

```bash
$ sudo launchctl unload /Library/LaunchDaemons/com.couchbase.mobile.sync_gateway.plist
```

The config file and logs are located in `/Users/sync_gateway`.

## Connecting Sync Gateway to Couchbase Server

After you have installed Sync Gateway, you can optionally connect it to a Couchbase Server instance. By default, Sync Gateway uses a built-in, in-memory server called "Walrus" that can withstand most prototyping use cases, extending support to at most one or two users.

To connect Sync Gateway to Couchbase Server:

- Open the Couchbase Server Admin Console and log on using your administrator credentials.
- In the toolbar, click **Data Buckets**.
- On the Data Buckets page, click **Create New Data Bucket** and create a bucket named `sync_gateway` in the default pool.

See the latest Sync Gateway Release Notes for version compatibility information, available on the [Downloads](http://www.couchbase.com/nosql-databases/downloads#Couchbase_Mobile) page.

> **Note:** Note: You can use any name you want for your bucket, but `sync_gateway` is the default name that Sync Gateway uses if you do not specify a bucket name when you start Sync Gateway. If you use a different name for your bucket, you need to specify the name in the configuration file or via the command-line option `-bucket`.

### Accessing and modifying Sync Gateway's bucket

Sync Gateway is similar to an application server in that it considers itself the owner of its bucket, and stores data in the bucket using its own schema. Even though the documents in the bucket are normal JSON documents, Sync Gateway adds and maintains its own metadata to them to track their sync status and revision history.

> **Note:** Do not add, modify or remove data in the bucket using Couchbase APIs or the admin UI, or you will confuse Sync Gateway. To modify documents, we recommend you use the Sync Gateway's REST API. If you need to operate on the bucket using Couchbase Server APIs, use the Bucket Shadowing feature to create a separate bucket you can modify, which the Sync Gateway will "shadow" with its own bucket.

## Instance from AWS marketplace

1. Browse to the [Sync Gateway AMI](https://aws.amazon.com/marketplace/pp/B013XDNYRG) in the AWS Marketplace.
1. Click Continue.
1. Make sure you choose a key that you have locally.
1. Paste the [user-data.sh](https://raw.githubusercontent.com/couchbase/build/master/scripts/jenkins/mobile/ami/user-data.sh) script contents into the text area in Advanced Details
1. If you want to run a custom Sync Gateway configuration, you should customize the variables in the Customization section of the user-data.sh script you just pasted.  You can set the Sync Gateway config to any public URL and will need to update the Couchbase Server bucket name to match what's in your config.
1. Edit your Security Group to expose port 4984 to Anywhere

### Verify via curl

From your workstation:

```bash
$ curl http://public_ip:4984/sync_gateway/
```
You should get a response like the following:

```bash
{"committed_update_seq":1,"compact_running":false,"db_name":"sync_gateway","disk_format_version":0,"instance_start_time":1446579479331843,"purge_seq":0,"update_seq":1}
```

### Customize configuration

For more advanced Sync Gateway configuration, you will want to create a JSON config file on the EC2 instance itself and pass that to Sync Gateway when you launch it, or host your config JSON on the internet somewhere and pass Sync Gateway the URL to the file.

### View Couchbase Server UI

In order to login to the Couchbase Server UI, go to `http://public_ip:8091` and use the following credentials:

**Username**: Administrator

**Password**: The AWS instance id that can be found on the EC2 Control Panel (eg: i-8a9f8335)
