---
id: security
title: Security and Access Control
permalink: ready/training/design/security/index.html
---

In this lesson you’ll learn how to secure your data model using Couchbase Mobile’s built-in security framework. You’ll design the security rules for each object in your data model. This includes access control, data validation, and access grants.

So far you learned how to model your data according to the application's data requirements. You may also have security requirements to enforce data access for each user of your app. With Couchbase Mobile your data schema is coupled with the security rules.

To understand why you must first have a basic understanding of the security features in Sync Gateway. It contains a feature called the Sync Function which has the ability to assign documents to something we call channels. Channels act as an organization mechanism. The graphic below may help to understand how channels work. It shows conceptually the idea of Sync Gateway feeding documents to channels during a replication.

![](img/image81.png)

You control assigning documents to channels through the Sync Gateway sync function. Each blue pipe in the diagram represents a channel. The green arrows illustrate the idea that the sync function can assign any individual document to any number of channels.

The Sync Function is the core API you interact with on Sync Gateway. Every time a new document is added the Sync Function is called and given a change to examine the document. It can do the following things:

- Validate the document.
- Authorize the change if it's an update operation.
- Assign the document to channels (i.e routing).
- Grant users access to channels (i.e read permission).

## List ownership

The list and tasks are routed to the same channel (456) and the owner of that list has access to that channel. The channel name corresponds to the list _id.

![](img/image82.png)

![](img/image62.png)

## List sharing

A list user document is created to grant a **username** access to a list.

![](img/image63.png)

## Moderator role

Users with the **moderator** role can access any list.

![](img/image64.png)

## Admin role

Users with the **admin** role can give the **moderator** role to any user.

![](img/image65.png)

## Conclusion

Well done! You've completed this lesson on designing the security model for each scenario in the application. In the next lesson you'll learn how to create an empty database to store documents. Feel free to share your feedback, findings or ask any questions on the forums.