---
id: os-level-tuning
title: OS Level Tuning
permalink: ready/guides/sync-gateway/os-level-tuning/index.html
---

To get the most out of Sync Gateway, it may be necessary to tune a few parameters of the OS.

## Tuning the max no. of file descriptors

Raising the maximum number of file descriptors available to Sync Gateway is important because it directly affects the maximum number of **sockets** the Sync Gateway can have open, and therefore the maximum number of endpoints that the Sync Gateway can support.

### Linux Instructions (CentOS)

The following instructions are geared towards CentOS.

Increase the max number of file descriptors available to **all processes**. To specify the number of system wide file descriptors allowed, open up the `/etc/sysctl.conf` file and add the following line.

```bash
fs.file-max = 500000
```

Apply the changes and persist them (this will last across reboots) by running the following command.

```bash
$ sysctl -p
```

Increase the **ulimit** setting for max number of file descriptors available to a single process. For example, setting it to 250K will allow the Sync Gateway to have 250K connections open at any given time, and leave 250K remaining file descriptors available for the rest of the processes on the machine. These settings are just an example, you will probably want to tune them for your own particular use case.

```bash
$ ulimit -n 250000
```

In order to persist the ulimit change across reboots, add the following lines to `/etc/security/limits.conf`.

```bash
* soft nofile 250000
* hard nofile 250000
```

Verify your changes by running the following commands.

```bash
$ cat /proc/sys/fs/file-max
$ ulimit -n 
```

The value of both commands above should be `250000`.

### References

- [Increasing ulimit and file descriptors limit on Linux](https://glassonionblog.wordpress.com/2013/01/27/increase-ulimit-and-file-descriptors-limit/)

## Tuning the TCP Keepalive parameters

If you have already raised the maximum number of file descriptors available to Sync Gateway, but you are still seeing "too many open files" errors, you may need to tune the TCP Keepalive parameters.

### Understanding the problem

Mobile endpoints tend to abruptly disconnect from the network without closing their side of the connection, as described in [Section 2.3. (Checking for dead peers)](http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html) of the TCP-Keepalive-HOWTO.

By default, these connections will hang around for approximately 7200 seconds (2 hours) before they are detected to be dead and cleaned up by the tcp/ip stack of the Sync Gateway process. If enough of these connections accumulate, you can end up seeing "too many open files" errors on Sync Gateway.

If you are seeing "too many open files" errors, you can count the number of established connections coming into your sync gateway with the following command:

```bash
$ lsof -p <sync_gw_pid> | grep -i established | wc -l
```

If the value returned is near your max file descriptor limit, then you can either try increasing the max file descriptor limit even higher, or tuning the TCP Keepalive parameters to reduce the amount of time that dead peers will cause a socket to be held open on their behalf.

### Linux Instructions (CentOS)

Tuning the TCP Keepalive settings is not without its downsides -- it will increase the amount of overall network traffic on your system, because the tcp/ip stack will be sending more frequent Keepalive packets in order to detect dead peers faster.

The following settings will reduce the amount of time that dead peer connections hang around from approximately 2 hours down to approximately 30 minutes. Add the following lines to your `/etc/sysctl.conf` file:

```bash
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 9
```

To reduce the amount of time even further, you can reduce the `tcp_retries2` value. Add the following line to your `/etc/sysctl.conf` file:

```bash
net.ipv4.tcp_retries2 = 8
```

To activate the changes and persist them across reboots, run:

```bash
$ sysctl -p
```

### References

- [TCP Keepalive HOWTO](http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html)
- [Application control of TCP retransmission on Linux](http://stackoverflow.com/questions/5907527/application-control-of-tcp-retransmission-on-linux)
- [Proactively closing longpoll connections for endpoints that disappear from the network](https://groups.google.com/forum/#!msg/golang-nuts/rRu6ibLNdeI/0bjSmO5fN_8J)
- [TCP man page](http://linux.die.net/man/7/tcp)
- [Sync Gateway Issue 742](https://github.com/couchbase/sync_gateway/issues/742)