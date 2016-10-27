---
id: scale
title: Scale
permalink: ready/training/deploy/scale/index.html
---

In this lesson you'll learn how to scale Sync Gateway and Couchbase Server in real-time with zero downtime.

#### Requirements

Four instances with the following:

- Centos 7
- RAM >= 2GB

## Architecture

In this lesson you will deploy a 3rd Sync Gateway node behind the reverse proxy.

![](img/image80.png)

## Scaling Sync Gateway

Similarly to previous lessons you will first deploy Sync Gateway with the configuration file passing the IP of the VM running Couchbase Server.

There will be 3 Sync Gateway nodes but the reverse proxy is forwarding the load to only 2 of them. To balance the traffic across all 3 you must update the NGINX config file with the IP of VM5.

### Try it out

1. Log on VM5 (sync-gateway).
2. Run the Sync Gateway install script passing the IP of VM1 where Couchbase Server is running.

    ```bash
    sudo ./install_sync_gateway.sh VM1
    ```

3. Log on VM4 (nginx).
4. Run the NGINX install script passing the IP of VM2, VM3 and VM4 where the Sync Gateway instances are running.

    ```bash
    sudo ./configure_nginx.sh VM2 VM3 VM5
    ```

5. Monitor the NGINX operations in real-time.

    ```bash
    tail -f /var/log/nginx/access_log
    ```

6. Send a `/{db}/_all_docs` request with the **user1/password** credentials to http://VM4_IP:8000. The Sync Gateway logs will print this operation.

    ![](https://cl.ly/392N2E2K0J0T/image76.gif)

## Conclusion

Well done! You've completed this lesson on scaling. Feel free to share your feedback, findings or ask any questions on the forums.
