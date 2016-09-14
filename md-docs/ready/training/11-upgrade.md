---
id: upgrade
title: Upgrade
permalink: ready/training/deploy/upgrade/index.html
---

In this lesson you'll learn how to install upgrades for Sync Gateway with zero downtime.

[//]: # "COMMON ACROSS LESSONS"

#### Requirements

Three instances with the following:

- Ubuntu >= 12.04, =< 14.04
- RAM >= 2GB

#### Getting Started

This lesson contains some scripts to automatically deploy and configure Sync Gateway with Couchbase Server. Download those scripts on each VM using wget.

```bash
wget https://cl.ly/3Z0D2D0l3R0O/deploy.zip
sudo apt-get install unzip
unzip deploy.zip
```

Throughout this lesson, you will use different scripts located in the **deploy** folder.

[//]: # "COMMON ACROSS LESSONS"

## Architecture

To follow this lesson you must first have completed the Install lesson and have 2 Sync Gateway nodes up and running. You have deployed Sync Gateway 1.3 and in this lesson you will deploy Sync Gateway 1.3.1 as a rolling upgrade.

A rolling upgrade means that the nodes are upgraded one at a time. While a node is being upgraded it's taken offline by rebalancing the traffic to other nodes. The diagram below shows this process.

![](img/image79.png)

## Upgrading VM2

First you will need to redirect the traffic to only one Sync Gateway node (VM3). The following script updates the NGINX config with the IP addresses provided as command line arguments. 

```bash
#!/usr/bin/env bash

# Install NGINX
sudo apt-get install nginx

# Update NGINX config with IPs
cp nginx_template.txt tmp.txt
for ip in "$@"
do
	echo "$ip"
	output="$(awk '{print} /sync_gateway_nodes/{print "server '${ip}':4984;"}' tmp.txt)"
	echo "$output" > tmp.txt
done

# Move NGINX config to /etc/nginx/sites-available/sync_gateway_nginx
mv tmp.txt /etc/nginx/sites-available/sync_gateway_nginx

# Enable the configuration file by creating a symlink
ln -s /etc/nginx/sites-available/sync_gateway_nginx /etc/nginx/sites-enabled/sync_gateway_nginx

# Restart NGINX
sudo service nginx restart
```

Once the traffic is successfully redirected you can upgrade Sync Gateway on VM2. The following script uninstalls Sync Gateway 1.3.0 and installs Sync Gateway 1.3.1.

```bash
#!/usr/bin/env bash

# Stop Sync Gateway
service sync_gateway stop

# Uninstall Sync Gateway 1.3.0
dpkg -r couchbase-sync-gateway
dpkg -P couchbase-sync-gateway

# Download and Sync Gateway 1.3.1
wget http://packages.couchbase.com/releases/couchbase-sync-gateway/1.3.1/couchbase-sync-gateway-community_1.3.1-16_x86_64.deb
dpkg -i couchbase-sync-gateway-community_1.3.1-16_x86_64.deb
```

Notice here that we don't have to reload the configuration file. It's already present in **home/sync\_gateway/sync\_gateway.json**.

### Try it out

1. Log on the terminal console of VM2.
2. Run the NGINX install script passing only the IP of VM3.

    ```bash
    bash install_nginx.sh VM3
    ```

3. Run the Sync Gateway upgrade script on VM2.

    ```bash
    bash upgrade_sync_gateway.sh
    ```

4. Run the NGINX install script again this time passing the IP of VM2 and VM3.

    ```bash
    bash install_nginx.sh VM2 VM3
    ```

5. Monitor the NGINX operations in real-time.

    ```bash
    tail -f /var/log/nginx/access_log
    ```

6. Send a `/{db}/` request with the **user1/password** credentials to http://VM2_IP:8000/. The response contains the Sync Gateway version. Notice that it switch between 1.3.0 and 1.3.1 because only one node was upgrade.

    ![](https://cl.ly/3m0g1R0J0w37/image77.gif)

## Upgrading VM3

In this lesson you will perform the same sequence of operations to upgrade the Sync Gateway version running on VM3.

![](img/image78.png)

### Try it out

1. Log on the terminal console of VM2.
2. Run the NGINX install script passing only the IP of VM2.

    ```bash
    bash install_nginx.sh VM2
    ```

3. Log on the terminal console of VM3.
4. Run the Sync Gateway upgrade script on VM3.

    ```bash
    bash upgrade_sync_gateway.sh
    ```

5. Log on the terminal console of VM2.
6. Run the NGINX install script again this time passing the IP of VM2 and VM3.

    ```bash
    bash install_nginx.sh VM2 VM3
    ```

7. Verify that the Sync Gateway version is now 1.3.1

    ```bash
    curl VM2:8000

    {
        "couchdb":"Welcome",
        "vendor":{"name":"Couchbase Sync Gateway","version":1.3},
        "version":"Couchbase Sync Gateway/1.3.1(16;f18e833)"
    }
    ```

## Conclusion

Well done! You've completed this lesson on upgrading the Sync Gateway version. In the next lesson you will learn how to scale Sync Gatway by adding additional nodes. Feel free to share your feedback, findings or ask any questions on the forums.