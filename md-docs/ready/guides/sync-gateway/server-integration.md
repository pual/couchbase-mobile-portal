---
id: server-integration
title: Server integration
permalink: ready/guides/sync-gateway/server-integration/index.html
---

This guide describe two approaches for integrating Sync Gateway with other servers. These approaches can be used to build services that react to changes in documents. Examples of use cases include:

- Sending notifications when specific documents change, for example, by email or SMS
- Customized auditing or logging

The integration approaches are:

- **Webhook event handlers:** Sync Gateway can detect document updates and post the updated documents to one or more external URLs. The first article in this guide describes this approach.
- **Changes worker pattern:** The changes worker pattern treats documents as state machines and uses a changes feed to obtain information about when documents change. This integration permits applications to implement business logic that reacts to changes in documents. The second article in this guide describes this approach.

Here's a table that compares each API in different scenarios:

|Scenario|Changes feed (pull)|Webhooks (push)|
|:-------|:------------------|:--------------|
|Sequence/Ordered|Yes|No|
|User Access Control|Fine Grain|Limited|
|Scalable|Yes|No|
|Data Stream replay on Failure|Yes|No|

## Using Webhooks

Sync Gateway provides a webhook feature that allows Sync Gateway to detect document changes and to post changed documents to URLs that you specify. In more detail, the steps for a single webhook event handler are:

1. **Raise and listen for events**: Document changes (creations, updates, and deletions) that are made through Sync Gateway's Public REST API, including document changes that result from Couchbase Lite push replications, raise events that webhook event handlers listen for.
2. **Filter**: You can define a `filter` function to examine the contents of the changed documents, and to decide which ones to post.
3. **Post**: Sync Gateway uses asynchronous HTTP or HTTPS POSTs to post the changed documents identified by the `filter` function to the specified URL. Without a `filter` function, Sync Gateway posts all changed documents.

You can define multiple webhook event handlers. For example, you could define webhooks with different filtering criteria and that post changed documents to different URLs.

> **Caution:** Webhooks post your application's data, which might include user data, to URLs. Consider the security implications.

### When events are raised

Sync Gateway raises a `document_changed` event every time it writes a document to a Couchbase Server bucket, such as during a Couchbase Lite push replication session.

### Configuring webhook event handlers

You configure event handlers for webhooks with the `event_handlers` property in the database configuration section of the JSON configuration file.

All event handlers share the following properties:

|Name|Type|Description|Default|
|:---|:---|:----------|:------|
|`document_changed`|Key for an array|Type of the event (`document_changed`)|None|
|`max_processes` (optional)|`integer`|Maximum number of events that can be processed concurrently. No more than `max_processes` processes will be spawned for event handling.|`500`|
|`wait_for_process` (optional)|`string`|Maximum wait time in milliseconds before canceling event processing for an event that is detected when the event queue is full.|`5`|

The section [Tuning performance](/documentation/mobile/current/develop/guides/sync-gateway/server-integration/index.html#tuning-performance) describes how to use the properties `max_processes` and `wait_for_process`, and the property `timeout` (described below), to tune performance.

Each `webhook` event handler has the following properties:

|Name|Type|Description|Default|
|:---|:---|:----------|:------|
|`handler`|string|Type of the event handler, in this case, `webhook`|None|
|`url`|string|URL to use for the HTTP or HTTPS POST|None|
|`filter` (optional)|string|A JavaScript function used to determine which documents to post. The filter function accepts the document body as input and returns a boolean value. If the filter function returns true, then Sync Gateway posts the document. If the filter function returns false, then Sync Gateway does not post the document. If no filter function is defined, then Sync Gateway posts all changed documents. Filtering only determines which documents to post. It does not extract specific content from documents and post only that.|None|
|`timeout` (optional)|integer|Time in seconds to wait for a response to the POST operation. Using a timeout ensures that slow-running POST operations don't cause the webhook event queue to back up. Slow-running POST operations are discarded (if they time out), so that new events can be processed. When the timeout is reached, Sync Gateway stops listening for a response. A value of 0 (zero) means no timeout.|60|

#### Examples

Following is a simple example of a `webhook` event handler. In this case, a single instance of a `webhook` event handler is defined for the event `document_changed`. Every time a document changes, the document is sent to the URL `http://someurl.com`.

```javascript
"event_handlers": {
	"document_changed": [
		{
			"handler": "webhook",
			"url": "http://someurl.com"
		}
	]
}
```

Following is an example that defines two `webhook` event handlers. The `filter` function in the first handler recognizes documents with `doc.type` equal to `A` and posts the documents to the URL `http://someurl.com/type_A`. The `filter` function in the second handler recognizes documents with `doc.type` equal to B and posts the documents to the URL `http://someurl.com/type_B`.

```javascript
"event_handlers": {
      "document_changed": [
        {"handler": "webhook",
         "url": "http://someurl.com/type_A",
         "filter": `function(doc) {
              if (doc.type == "A") {
                return true;
              }
              return false;
            }`
         },
        {"handler": "webhook",
         "url": "http://someurl.com/type_B",
         "filter": `function(doc) {
              if (doc.type == "B") {
                return true;
              }
              return false;
            }`
        }
     ]
  }
```

### Tuning performance

> **Note:** Default values of the properties `timeout`, `max_processes`, and `wait_for_process` should work well in the majority of cases. You should not need to adjust the values to tune performance.

Webhooks in Sync Gateway are designed to minimize performance impacts on Sync Gateway's regular processing.

Following is some background information about event processing that will help you to understand the properties that you can tune:

- Sync Gateway manages the number of processes that are spawned for webhook event handling, so that slow response times from the HTTP POST operations don't consume available CPU resources on Sync Gateway nodes.
- When a `webhook` event handler is defined, after Sync Gateway has updated a document, Sync Gateway adds a `document_changed` event to an asynchronous event-processing queue (the event queue). New processes are then spawned to apply the `filter` function to the documents and to perform the HTTP POST operations.

Based on your use case and server capabilities, you can tune the behavior of the webhooks feature. Adjust the value of configuration property `timeout` (described above) and the following properties in the `event_handlers` property in the database configuration section of the JSON configuration file:

|Name|Type|Description|Default|
|:---|:---|:----------|:------|
|`max_processes` (optional)|`integer`|Maximum number of events that can be processed concurrently, that is, no more than `max_processes` concurrent processes will be spawned for event handling.|`500`|
|`wait_for_process` (optional)|`string`|Maximum wait time in milliseconds before canceling event processing for an event that is detected when the event queue is full. If you set the value to 0 (zero), then incoming events are discarded immediately if the event queue is full.|`5`|

Possible configuration scenarios include:

- **Avoid any blocking:** To avoid any blocking of standard Sync Gateway processing, set `wait_for_process` to 0 (zero). If the event queue is full, any incoming events are discarded immediately.
- **Ensure that most webhook posts are sent:** To ensure that most webhook posts are sent, set `wait_for_process` to a sufficiently high value. POST operations do not guarantee delivery.

#### Example

Following is a sample configuration of event handlers that uses the performance-tuning properties:

```javascript
"event_handlers": {
    "max_processes" : 1000,
    "wait_for_process" : "20",
    "document_changed": [
     {"handler": "webhook",
      "timeout": 90,
      "url": "http://someurl.com"}
    ]
 }
```

In this example:

- Every time a document changes, the document is sent to the URL `http://someurl.com`.
- The event handler can process a maximum of `1000` events concurrently.
- The event handler will wait a maximum of `20` milliseconds before canceling event processing for an event that is detected when the event queue is full.
- The event handler waits `90` seconds for a response to a POST operation.

### Logging

When an event is not added to the event queue, but is instead discarded, a warning message is written the the Sync Gateway log.

You can configure Sync Gateway to log information about event handling, by including either the log key `Event` or `Events+` in the `Log` property in your Sync Gateway configuration file. `Events+` is more verbose.

## Changes Worker Pattern

This article describes how to use the changes worker pattern to integrate Sync Gateway with other backend processes. The changes worker pattern treats documents as state machines and uses a changes feed to obtain information about when documents change. This integration permits applications to implement business logic that reacts to changes in documents.

If you connect to a changes feed of some channels, you'll get lines of JSON for each matching change.

Now you can write a script to consume these channels and take action based on them. For instance if you have a channel called "needs-email" you could have a bot that sends an email and then saves the document back with a flag to keep it out of the needs-email channel.

> **Note:** Your workers should be idempotent or you should track the `last_seq` somewhere durable.

There are [existing libraries to handle a changes feed](https://github.com/iriscouch/follow) with a worker process.

### State Machine Document Pattern

One way to think of your documents when you are using them in a changes-driven worker process, is as a state machine. So maybe you have complex business logic to run when adding someone to a project. So not only does the new project member have to be invited to the project, the accepted invitation has to be approved by two layers of management. Yay management.

To implement this you would treat the document as a state machine. You can put a state field on it, and then the steps to get to the next state can alternate between human and robot powered.

So in this case, the project owner would create the invitation, and according to their level of access, they aren't able to send the invitation without management approval. So they save it with state="needs-approval" and the sync function automatically routes it to a channel of approvals needed by managers. The invitation is sitting in a needs-approval queue on some management iPhones, and when one of the managers clicks OK, then it enters state="approved" which cases the sync function to put it in a channel where it can be seen by the invitee.

Now when the invite clicks "accept" they can put it into state="accepted" which may or may not need more management approval before the a robot changes the state="active" which actually grants access the invitee access to the project.

### Example Document Workflow

This section describes a sample use case with point-of-sale (POS) applications.

Mobile applications are deployed at the edge of the Internet, but large parts of their business logic need to be run by central authorities. In this post I’ll describe a technique for asynchronous business logic using Couchbase Lite and Sync Gateway, that should be easy to reason about, and scale for most applications. Implementing this technique requires only that you are able to make HTTP and JSON requests to Sync Gateway from your application environment. Typically you’ll end up running your application code as a daemon in the same datacenter as Sync Gateway.

To make this example more concrete, let’s flesh out the idea of a mobile point-of-sale transactions application. The sales staff and managers install the app on their phones, and sales staff ring up customer’s items using a native UI and sensors to read package tags. When a sales associate enters all of a customer’s items, they run the transaction via their device (again using sensors to read credit card info) or send the transaction to a central terminal if the customer is paying cash.

The business logic we are are concerned with: Returns, certain high risk items, and transactions over $100 need a manager to approve them, before the transaction can be run.

With JSON Anywhere, there are lots of opportunities to let the database system do the hard work you may have relied on special purpose systems to handle in the past. The patterns in this post are designed so you can incrementally upgrade your system, adding features while building on existing code.

### Documents As State Machines

We’ll look at how treating documents like state machines can simplify workflow management. We start with a workflow that involves a human manager to approve transactions, and then move to a workflow that replaces the manager with an automated risk agent. Finally, we extend the agent to be push based instead of polling the database for jobs.

#### The Old World: 3-Tier Web Architecture

In the classic 3-tier web services architecture, you’d have a server farm dedicated to API connections from your mobile clients. The clients would send transactions to these API servers, which would take some action or actions, perhaps updating a database or connecting to another web service, and eventually return a success or failure code to the client. This request and response model should be familiar to anyone who has worked on a web application or API.

One problem with this approach is that it assumes a client’s connection will hold up long enough for the transaction to commit. To work around these issues, the application server may provide the client with an intermediate transaction token that can be used for retries, or use other robustness techniques.

Another problem (or benefit) with the request and response model is that it can be used to put a veil of simplicity over complex internal workings. This is a great way to work when the application runs in the cloud. But when the application is distributed across mobile clients, the request and response application model outruns its usefulness.

#### The New World: Let the Database Handle the Network

By the time even a simple custom API is hardened against the real world, it has become a real technical investment. Instead you want a standard data model based on state and events, so that clients can be notified of new changes without polling.

When the database abstracts the network, application developers can depend on changes to JSON documents showing up on other devices and in the cloud. They don’t need to write custom code to handle client/server interactions. For simpler apps this might mean doing away with the application server altogether, and mediating all communication via the database abstraction.

### In Practice

Once you have Couchbase Lite and Sync Gateway handling the network connection, you can communicate transaction status via the database. The shift in understanding here is becoming comfortable with storing documents that represent incomplete or “pending” transactions. The essence of the technique is writing a transaction in “pending” or “needs-approval” state on the client, and then granting your managers the permissions to move documents from “pending" in to “approved” state. So you can have sales associates directly writing small transactions, but automatically save large transactions as “pending” so they can be brought to the attention of the manager.

So when your transaction processing code sees a JSON document that represents a transaction that needs approval, because it has doc.state == “pending” it will not take any action, but instead wait for the document to move to the “approved” state, before running it.

By encapsulating a transaction state-machine as a document, you have the flexibility of moving the document from state to state on any platform you want. So you can have some state changes driven by a UI on a mobile device, and others via the web, or as we’ll talk about momentarily, by a robot.

In our application, we could ensure that all floor managers are subscribed to a channel for the transactions that need approval, so that they can be resolved on the manager’s device, giving a seamless experience to the customer.

#### We Want Robots!

The above scenario illustrates the document state machine pattern, but we were promised automation. Instead of keeping a manager on the floor to approve transactions, wouldn’t it be faster and more reliable to automate risk assessment in the cloud? Moving the transaction approval responsibility from floor managers to an automated risk management agent shouldn’t require big technical changes.

The risk management agent can be deployed as a program in the cloud, with access to Sync Gateway. It will be interested in transactions that need approval, so it can run a query in Couchbase Server to find documents in the “pending” state. The risk manager takes each transaction, and queries whatever external systems are required in order to approve or deny the transaction. Once it has made its determination, the risk agent updates the document in Sync Gateway with a the tag “approved” or “denied”. The approved transactions are in the same state they’d been moved into by the floor manager in an earlier version of the application, but now they are being approved by a robot not a human.

### Example of a push-notification robot

Link to example code and README with Urban Airship push API

We haven’t yet described how we’d trigger the robot to look for transactions that need approval. Polling is not efficient, so we’d like the robot to be notified each time there is a document that needs approval, so that it can being work immediately — without any expensive polling. We need Sync Gateway to push new pending transactions to our robot.

Sync Gateway already has an infrastructure for pushing messages to channels, so we’ll build on top of that, by creating a channel for "pending-transactions" in our sync function, and each time a transaction is saved in the pending state, it will show up on this channel. So the robot can just subscribe to the “pending-transactions” channel and it will be notified each time there is work for it to do.

Here is a sample implementation of a [push notification robot](https://github.com/couchbaselabs/CouchChat-iOS/tree/push/push-notifications) powered by Sync Gateway channels.