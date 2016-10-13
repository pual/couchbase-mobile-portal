---
id: scale
title: Scale
permalink: ready/training/deploy/scale/index.html
---

In this lesson you'll learn how to scale Sync Gateway and Couchbase Server in real-time with zero downtime.

#### Requirements

Two instances with the following:

- Ubuntu >= 12.04, =< 14.04
- RAM >= 2GB

## Architecture

In the previous lesson you already deployed 1 node of Sync Gateway and 1 node of Couchbase Server on the same Ubuntu instance. In this lesson you will deploy an additional set of Sync Gateway and Couchbase Server to another Ubuntu instance. The diagram describes the architecture. With two Sync Gateway instances you will also deploy a reverse proxy to distribute the load between each one. Both Sync Gateways will use the exact same configuration file.

![](img/image71.png)

## Scaling Sync Gateway

By running identically configured instances of Sync Gateway on each of several machines, and load-balancing them by directing each incoming HTTP request to a random one. Sync Gateway nodes are “shared-nothing,” so they don’t need to coordinate any state or even know about each other.

### Try it out

1. Log on the terminal console of VM2.
2. Run the **deploy/install\_sync\_gateway.sh** script passing the IP of the node running Couchbase Server.

```bash

```

3. Monitor the log file.

    ```bash
    tail -f /home/sync_gateway/logs/sync_gateway_error.log
    ```

4. Send a `/{db}/_all_docs` request with the **user1/password** credentials http://VM4_IP:4984/todo. The Sync Gateway logs will print this operation.

// gif

## Balancing the traffic

There are now 3 Sync Gateway nodes but the reverse proxy is forwarding the load to only 2 of them. To balance the traffic across all 3 you must update the NGINX config file with the IP of VM4.

The following NGINX configuration file balances the traffic between VM2, VM3 and VM4.

```bash
upstream sync_gateway {
    server VM2:4984;
    server VM3:4984;
    server VM4:4984;
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

### Try it out

1. Log on the terminal console of VM2.
2. Run the **deploy/scale.sh** script passing the IPs of the different VMs.

    ```bash
    ./deploy/scale.sh VM2 VM3 VM4
    ```

3. Monitor the NGINX operations in real-time.

```bash

```

4. Send a `/{db}/_all_docs` request with the **user1/password** credentials to http://VM2_IP:8000/todo. The NGINX logs will print this operation.

// gif

## Conclusion

Well done! You've completed this lesson on scaling. Feel free to share your feedback, findings or ask any questions on the forums.