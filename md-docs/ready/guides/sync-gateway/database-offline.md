---
id: database-offline
title: Taking databases offline and online
permalink: ready/guides/sync-gateway/database-offline/index.html
---

While running, a Sync Gateway instance usually maintains connections with all of the databases that are defined in the configuration file that was used when the Sync Gateway instance was started. These databases are referred to as being online. When a database is online, Sync gateway serves both Public REST API and Admin REST API requests for the database.

In Sync Gateway 1.1, there was no ability to take a single database offline at Sync Gateway. Stopping a Sync Gateway instance would take all of the databases offline, even if that was not desirable.

Sync Gateway 1.2 introduces functionality that permits a specific database to be taken offline and brought back online, without requiring that the Sync Gateway instance be stopped and without affecting other databases that are served by the instance. An offline database is not accessible through Sync Gateway's Public REST API. However, some commands can be given to Sync Gateway through the Admin REST API. The database in Couchbase Server is not down.

> **Note:** This online/offline status for a database is with respect to a specific Sync Gateway instance. The status does not apply to other Sync Gateway instances, unless coordinated operations there have brought the databases on those instances into the same state.

## Motivation and use cases

The database offline/online functionality that is being introduced in Sync Gateway 1.2 permits you to take individual databases offline and to bring them online again. The motivations for this functionality include:

- Reducing the number of administrative tasks that require Sync Gateway instances to be restarted.
- Allowing operations that impact service delivery for a specific database (such as resynchronization) to occur without removing Sync Gateway instances from service, and thereby impacting the service delivery for other databases.

Specific uses for the database offline/online functionality include:

- Taking a database offline, without affecting other databases.
- Changing configuration properties for a database (while it is offline), without needing to restart Sync Gateway.
- Resynchronizing a database while it is offline.
- Detecting a lost DCP or TAP feed, and taking the database offline automatically.
- Creating a database in an offline state, so that the start of service delivery for the database can be postponed or coordinated across Sync Gateway instances.

## Taking a database offline

### Effects

The effects of taking a database offline are:

- Stopping serving Public REST API requests for the database
- Stopping serving the majority of Admin REST API requests for the database. A specific, short list of Admin REST API requests remain available.
- Closing changes feeds
- Stopping webhook event handlers

> **Note:** Taking a database offline that is in the progress of coming online will take the database offline after it comes online.

Following is an example of taking a database offline (using `curl`):

```bash
$ curl -X POST http://localhost:4985/db/_offline
```

### Admin REST API requests to an offline database

The following Admin API requests are allowed for an offline database:

- Getting status information for the database (`GET /{db}`)
- Updating the configuration for a database (`PUT /{db}/_config`). The new configuration is used when the database is 
 brought online.
- Resynchronizing a database (`POST /{db}/_resync`)

## Starting Sync Gateway and keeping a database offline

By default when Sync Gateway starts, it brings all databases that are defined in the configuration file online. To keep a database offline when Sync Gateway starts, you can add the configuration property `offline` to the database configuration properties for the database, with the value `true`.

Later, to bring the database online, you can use the `POST _online` Admin REST API request.

## Sync Gateway can take databases offline automatically

Sync Gateway will take a database offline automatically if specific conditions occur. Specifically, if Sync Gateway detects that the DCP feed or TAP feed for a database has been lost, then Sync Gateway takes the database offline automatically, so that the problem can be investigated. When the cause is known and has been corrected, you can use an Admin REST API request to bring the database back onine.

## Creating an offline database

You can use the `PUT /{db}` Admin REST API command to create a database. By default, the database that is created is brought online immediately. To create the database and keep it offline, include the configuration property `offline` with the value `true` in the database configuration properties for the database that you are posting in the request body.

Following is an example of creating an offline database (using `curl`):

```bash
$ curl -X PUT http://localhost:4985/db/ \
        -H "Content-Type: application/json" \
        -d '{"server":"http://localhost:8091","bucket":"bucket-1","users":{"GUEST":{"disabled":false,"admin_channels":["*"]}},"offline":true}'
```

## Bringing a database online

### Bringing a database online immediately

You can bring an offline database online immediately. You might do this if the reason for the database being taken offline has past, and if there is no reason to use a time delay.

Following is an example of bringing an offline database online immediately (using `curl`):

```bash
$ curl -X POST http://localhost:4985/db/_online
```

### Bringing a database online after a delay

You can bring an offline database online after a specific delay. Uses for this include:

- Making a database available for Couchbase Mobile clients at a specific time.
- Making databases on several Sync Gateway instances available at the same time.

Following is an example of bringing an offline database online after a delay (using `curl`):

```bash
$ curl -X POST http://localhost:4985/db/_online -H "Content-Type: application/json" -d '{"delay":3600}'
```

## Sync Gateway state diagrams

Following are state diagrams that represent the states for Sync Gateway, and for the connection between Sync Gateway and a Couchbase Server database, in Sync Gateway 1.1 and 1.2.

### States in Sync Gateway 1.1 and prior

This state diagram represents the states for Sync Gateway, and for the connection between Sync Gateway and a Couchbase Server database, in Sync Gateway 1.1. Numbers identify key points that are explained below the state diagram.

![](img/state-diagram-offline-11.png)

In the state diagram:

1. All databases defined in the configuration file are brought online (a connection is established with them) when Sync Gateway is started.
2. The only way to take a database offline (to interrupt the connection between Sync Gateway and the database) is to stop Sync Gateway. This also affects all other databases that the Sync Gateway instance serves.

### States in Sync Gateway 1.2

This state diagram represents the states for Sync Gateway and for the connection between Sync Gateway and a Couchbase Server database, in Sync Gateway 1.2. Numbers identify key points that are explained below the state diagram.

![](img/state-diagram-offline-12.png)

In the state diagram:

1. As in Sync gateway 1.1, all databases defined in the configuration file can be brought online (a connection is established with them) when a Sync Gateway instance is started. Also as before, all databases served by the instance can be taken fully offline by stopping a Sync Gateway instance.
2. Sync Gateway 1.2 adds the ability to start a Sync Gateway instance with one or more databases “offline.” This offline is not fully offline, but rather it blocks all Public REST API traffic and permits only specific Admin API commands. As before, the Sync Gateway instance can be stopped, which takes all databases served by the instance fully offline.
3. To the left of the gray dashed line, starting or stopping a Sync Gateway instance affects the connections to all 
of the databases that the instance serves. To the right of the gray dashed line, you perform operations on specific databases. For example, two databases could be online, while a third database could be taken offline, resynchronized, and then brought back online.
4. Through the Admin API, Sync Gateway 1.2 adds the ability to take one or more databases (that is, the connections to the databases) offline, without stopping the Sync Gateway instance.
5. Through the Admin API, you can bring offline databases back online, either immediately or with a specified delay.
6. When a database is offline, you can resync the database. During the resynchronization, the database status is 
ReSyncing.
7. When a database is offline, you can load configuration properties for the database, without stopping and re-starting the Sync Gateway instance. The new configuration properties are applied when the database is brought online.
