---
id: listener
title: Listener
permalink: ready/guides/couchbase-lite/native-api/attachment/index.html
---

Once the IP address of another device is known you can start replicating data to or from that peer. However, there are some good practice guidelines to follow in order to replicate the changes as they are persisted to a particular node.

## Filter functions

It may be desirable to use filter functions to replicate only the documents of interest to another peer. Filter functions in a peer-to-peer context are executed when the start method on the replication object is called. This is a major difference with the Sync Functionavailable on Sync Gateway that builds the access rules when documents are saved to the Sync Gateway database.

### Pull replication

### Push replication

## Port allocation

If the port number passed to the Listener is hardcoded, there is a small chance that another application may already be using it. To avoid this scenario, specifying a value of 0 for the port in the Listener constructor will let the TCP stack pick a random available port.

## Remote UUID

The replication algorithm keeps track of what was last synchronized with a particular remote database. To identify a remote, it stores a hash of the remote URL http://hostname:port/dbname and other properties such as filters, filter params etc. In the context of peer-to-peer, the IP address will frequently change which will result in a replication starting from scratch and sending over every single document although they may have already been replicated in the past. You can override the method of identifying a remote database using the remoteUUID property of the replicator. If specified, it will be used in place of the remote URL for calculating the remote checkpoint in the replication process.