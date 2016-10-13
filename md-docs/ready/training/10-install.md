---
id: install
title: Install
permalink: ready/training/deploy/install/index.html
---

In this lesson you'll learn how to install Sync Gateway and Couchbase Server, our NoSQL database server.

[//]: # "COMMON ACROSS LESSONS"

#### Requirements

Two instances with the following:

- Ubuntu >= 12.04, =< 14.04
- RAM >= 2GB

#### Getting Started

This lesson contains some deployment scripts to automically deploy and configure Sync Gateway with Couchbase Server. Download those scripts on each VM using wget.

```
wget https://cl.ly/3E2u2j0c1k0y/deploy.zip
sudo apt-get install unzip
unzip deploy.zip
```

Throughout this lesson, you will use different scripts located in the **deploy** folder.

[//]: # "COMMON ACROSS LESSONS"

## Architecture

The server-side architecture will be comprised of 1 node of Sync Gateway and 1 node of Couchbase Server. Both nodes will run on the same Ubuntu instance. Whether hosting on-premise or in the cloud you will want to have your Sync Gateway and Couchbase Server sit closely to each other for optimal performance between these two systems.

![](img/image74.png)

## Install Couchbase Server

To deploy Couchbase Mobile to production you must first get familiar Couchbase Server. It can deployed on a whole host of [operating systems](http://www.couchbase.com/nosql-databases/downloads) and can scale horizontally with multiple nodes or vertically by increasing the VM specs.

### Try it out

1. Log on terminal console of VM1.
2. Download Couchbase Server 4.1 using `wget`.

    ```bash
    wget http://packages.couchbase.com/releases/4.1.0/couchbase-server-community_4.1.0-ubuntu14.04_amd64.deb
    ```

2. Install Couchbase Server 4.1 using `dpkg`.

    ```bash
    dpkg -i couchbase-server-community_4.1.0-ubuntu14.04_amd64.deb
    ```

3. Initialize the cluster and a new user (**Administrator/password**).

    ```bash
    /opt/couchbase/bin/couchbase-cli cluster-init -c 127.0.0.1 --cluster-init-username=Administrator --cluster-init-password=password --cluster-init-ramsize=600 -u admin -p password
    ```

4. Initialize a new bucket called **todo**.

    ```bash
    /opt/couchbase/bin/couchbase-cli bucket-create -c 127.0.0.1:8091 --bucket=todo --bucket-type=couchbase --bucket-port=11211 --bucket-ramsize=600 --bucket-replica=1 -u Administrator -p password
    ```

5. Log in the Couchbase Server Admin Console on [http://NODE_IP:8091](http://NODE_IP:8091) with the user credentials that were created above (**Administrator/password**).

    ![](https://cl.ly/2v400A2s0I2v/image68.gif)

## Install Sync Gateway

Sync Gateway is the middleman server that exposes a database API for Couchbase Lite databases to replicate to and from. It connects internally to a Couchbase Server bucket to persist the documents.

In the production, the configuration file should point to a Couchbase Server URL and bucket as shown below.

```javascript
{
  "interface":":4984",
  "log": ["HTTP", "Auth"],
  "databases": {
    "todo": {
      "server": "http://localhost:8091",
      "bucket": "todo",
      "users": {
        "user1": {"password": "pass"},
        "user2": {"password": "pass"}
      },
      ...
    }
  }
}
```

### Try it out 

1. Log on the terminal console of VM2.
2. Download Sync Gateway 1.3.1

    ```bash
    wget http://latestbuilds.hq.couchbase.com/couchbase-sync-gateway/1.3.1/1.3.1-16/couchbase-sync-gateway-community_1.3.1-16_x86_64.deb
    ```

2. Install Sync Gateway using `dpkg`. This will install Sync Gateway as a service and start it automatically.

    ```bash
    dpkg -i couchbase-sync-gateway-community_1.3.1-16_x86_64.deb
    ```

3. By default, the config file is located at `/home/sync_gateway/sync_gateway.json`. You must upload the configuration file in the project file to this location on the VM. scp is a good utility for such operations.

    ```bash
    scp sync-gateway-config.json root@NODE_IP:/home/sync_gateway/sync_gateway.json
    ```

4. Restart the `sync_gateway` service.

    ```bash
    service sync_gateway restart
    ```

5. The database is now accessible at [http://NODE_IP:4984/todo](http://NODE_IP:4984/todo).

    ![](https://cl.ly/3r3J3S3b2G1Q/image69.gif)

    > **Note:** You can now run the Todo mobile app with the Sync Gateway URL pointing to your publicly accessible Sync Gateway database.

6. Repeat the same process for VM3.

## Using a reverse proxy

With two Sync Gateway nodes you can now configure the reverse proxy and update the sync endpoint in the mobile app to start replications pointing to the reverse proxy instead of an individual Sync Gateway instance.

### Try it out

1. Install Nginx.

    ```bash
    sudo apt-get install nginx
    ```

2. Insert the following in a new file **/etc/nginx/sites-available/sync_gateway**.

    ```bash
    upstream sync_gateway {
        server 139.59.178.239:4984;
        server 139.59.162.112:4984;
    }
    # HTTP server
    #
    server {
        listen 80;
        client_max_body_size 20m;
        location / {
            proxy_pass              http://sync_gateway;
            proxy_pass_header       Accept;
            proxy_pass_header       Server;
            proxy_http_version      1.1;
            keepalive_requests      1000;
            keepalive_timeout       360s;
            proxy_read_timeout      360s;
        }
    }
    ```

3. Enable this configuration by creating a symlink to a file with the same name in the folder **/etc/nginx/sites-enabled**.

    ```bash
    ln -s /etc/nginx/sites-available/sync_gateway /etc/nginx/sites-enabled/sync_gateway
    ```

4. Restart nginx.

    ```bash
    sudo service nginx restart
    ```

> **Note:** You can now run the Todo mobile app with the URL pointing to your nginx server publicly accessible.

## Conclusion

Well done! You've completed this lesson on installing Sync Gateway and Couchbase Server. In the next lesson you'll learn how to perform an upgrade on Sync Gateway. Feel free to share your feedback, findings or ask any questions on the forums.