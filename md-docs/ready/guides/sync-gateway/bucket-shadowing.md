---
id: bucket-shadowing
title: Bucket shadowing
permalink: ready/guides/sync-gateway/bucket-shadowing/index.html
---

> **Warning:** The Bucket Shadowing feature exists in order to allow Couchbase Mobile to integrate with some Couchbase Server-based applications that are already in production. This feature comes with many limitations and constraints on its usage, and is not suitable for many of the scenarios where one may be tempted to use it.

We **strongly** recommended using the Sync Gateway REST API for most --if not all-- integration use cases.

## Bucket Shadowing

**Bucket Shadowing** allows the Sync Gateway to serve an existing Couchbase Server bucket, making the contents of that bucket syncable with mobile clients. Actually this isn't completely true -- rather than directly serving the bucket, the gateway manages its own "shadow" bucket that contains the same documents but with the extra revision history metadata it needs. (For complicated reasons it can't store that metadata directly in the original bucket, because your Couchbase app already writes to those documents and the changes would conflict.)

Bucket shadowing is actually a different style of sync that operates between your app bucket and the gateway's shadow bucket. Every time your app changes a document, the gateway detects that and copies the change into its bucket as a new revision of the version-tracked document. And every time a mobile client revises a gateway document, the current revision is saved to your app bucket.

## Availability

Bucket shadowing was added on Jan 9 2014 (commit 70f92fd). It is _not_ available in beta 2.

## Configuration

We assume you already have a Couchbase Server with a bucket whose contents you want to make syncable.

You'll need another Couchbase bucket to act as the Sync Gateway's shadow. This doesn't have to be on the same server, although that's the most convenient way to do it. The two servers just have to be mutually reachable.

Configure the Sync Gateway as per the existing documentation. Then in your JSON configuration add a new property called `shadow` to the database configuration object; its value must be an object with properties `server` and `bucket`, representing the location of the app bucket to shadow:

    "shadow": {
        "server": "http://localhost:8091",
        "bucket": "myapp"
    }

You can optionally add a `"doc_id_regex"` property, whose value must be a regular expression: only document IDs / keys matching this regex will be transferred (in either direction).

> **Note:** If you're running a cluster of multiple sync gateways serving the same database, make sure that you only add the `shadow` property to _one_ gateway's configuration. Otherwise you'll have multiple tasks simultaneously trying to copy the same documents to and from the app bucket, which will result in collisions.

You may also want to add the key `"Shadow"` to the top-level configuration's `"log"` property, to get logging output from the shadowing task.

When you start the gateway, it will run through the app bucket's history (its tap feed) copying any new or changed documents into the gateway database. Depending on how large the bucket is, this may take a while. (Unfortunately there's no way to bypass this on subsequent launches of the gateway, due to limitations of the tap feed implementation.)

If you shut down the gateway (or it crashes), and changes are subsequently made to the app bucket, the gateway will find and apply those changes when it next starts up. However, the reverse situation doesn't work yet: if the app bucket becomes unavailable while the gateway is running, changes made to the gateway's database won't get propagated to the app bucket when it comes back. (Hopefully we can fix this before GA.)

## Limitations

- Bucket shadowing isn't highly available, and can't be scaled out – only one node should be configured for shadowing.
- When there are frequent updates to a document, bucket shadowing requires a higher `revs_limit` than non-bucket shadowing to ensure the revision hasn't been pruned by the time the shadowing echo arrives.

## FAQ

**Q:** Doesn't this double the storage required?  
**A:** Yes, unfortunately. (Actually it's a little more than doubling because of the extra revision-history metadata.) In the future we may be able to avoid storing a copy of the document body in the gateway.

**Q:** Does the bucket used by the gateway have to be on the same Couchbase server as the app bucket?  
**A:** No. In fact, there's probably a performance benefit to having them on separate servers, because the gateway's traffic won't be putting a load on the main server. (You could view the gateway as being a type of caching proxy for mobile clients.)

**Q:** Is deleting documents supported?  
**A:** Yes, although remember that deletions in a sync gateway database are just special "tombstone" revisions. If you delete a document in the app bucket, a deletion revision gets added in the database. If a client deletes a document and syncs to the gateway, the deletion revision is replicated and causes the app-bucket document to be deleted.

**Q:** What happens if the app updates a doc in the bucket at the same time that a mobile client pushes a change to it?  
**A:** In the gateway's database you get a conflict, just as if two clients had changed the document. Both revisions exist, and one will be (arbitrarily) picked as the default. The default revision will be copied back to the app bucket. When a client resolves the conflict by adding or deleting revisions, the resolved revision will be copied to the app bucket.

**Q:** Can I set an expiry value on documents?  
**A:** As documented in [issue 1161](https://github.com/couchbase/sync_gateway/issues/1161), documents that are added to the Couchbase bucket that have an expiry value set will be ignored by the Sync Gateway bucket shadowing process.
