---
id: integration
title: Integration
permalink: ready/training/integration/index.html
---

In this lesson you’ll learn how to integrate Couchbase Mobile with external systems using Sync Gateway. You’ll use the stream API for real-time streaming access to data changes, the batch APIs for bulk import/export operations, and webhooks for RESTful access to data changes.

[//]: # "COMMON ACROSS LESSONS"

#### Requirements

- Xcode 8 (Swift 3)
- Node.js 0.6.6 or higher

Download the project and Couchbase Lite SDK below.

<block class="ios" />

<div class="buttons-unit downloads">
  <a href="https://cl.ly/2H1c3E3D3l2S/xcode-project.zip" class="button" id="project">
    <img src="img/download-xcode.png">
  </a>
</div>

<div class="buttons-unit downloads">
  <a href="http://www.couchbase.com/nosql-databases/downloads#couchbase-mobile" class="button red">
    Download Couchbase Lite for iOS
  </a>
</div>

<img src="img/image41.png" class="center-image" />

Unzip the file and drag **CouchbaseLite.framework** to the **Frameworks** folder of the project in Finder. It's important to do this in Finder as opposed to Xcode.

<img src="img/drag-framework-finder.png" class="center-image" />

Open **Todo.xcodeproj** in Xcode. Then build & run the project.

<img src="img/image42.png" class="center-image" />

Throughout this lesson, you will navigate in different files of the Xcode project. We recommend to use the method navigator to scroll to a method.

<img src="https://cl.ly/0G263m3m1a0w/image44.gif" class="center-image" />

[//]: # "COMMON ACROSS LESSONS"

<block class="ios rn" />

## Stream API

The stream API returns a sorted list of changes made to documents in the database. It's the primary API to get notified of changes as they are processed by Sync Gateway and persisted to Couchbase Server.

Documents written to Sync Gateway are assigned a sequence value at write time. This sequence is used to order the changes feed. You can query the changes feed using a simple HTTP GET request to `/{db}/_changes`.

#### Try it out

<block class="ios rn" />

- Download Sync Gateway and start it with the configuration file in the root directory of the accompanying project.

    ```bash
    ~/Downloads/couchbase-sync-gateway/bin/sync_gateway sync-gateway-config.json
    ```

- Open **AppDelegate.swift** and set the following constants to `true`.

    ```swift
    let kLoginFlowEnabled = true
    let kSyncEnabled = true
    ```

- Run the application in Xcode, login with the **user1/pass** credentials and add a new list. It should appear as a new document on the Admin UI of Sync Gateway on [http://localhost:4985/_admin/db/todo](http://localhost:4985/_admin/db/todo).

- To access the stream API (also called the changes feed). Open a browser tab at [http://localhost:4985/todo/_changes](http://localhost:4985/todo/_changes).

    ![](img/image54.png)

- Create a new list and reload the browser tab to notice the new change with `seq: 3` as the sequence number.

    ```javascript
    {
      "seq":3,
      "id":"user1.CD59C6F0-D2FE-4C05-8712-52BDF86AA694",
      "changes":[{"rev":"1-5ad8375dc53dde5a2fd555b4b4a82184"}]
    }
    ```

- Delete the existing list and you should see a new change with `seq: 4` and `deleted: true`.

    ```javascript
    {
      "seq":4,
      "id":"user1.CD59C6F0-D2FE-4C05-8712-52BDF86AA694",
      "deleted":true,
      "changes":[{"rev":"2-3c169e667f727ab4b51e56f5878df7c7"}]
    }
    ```

Remember that deleting a document creates a new revision with the `deleted: true` property and no user properties. This is required in order to propagate the deletion to other devices. In the next section you will learn how to subscribe to the changes feed.

### Subscribing to changes

In the previous section you learned how to query the changes feed once. In this section you will write a script to subscribe to changes continuously. This becomes particularly useful for integrating Couchbase Mobile with other systems.

To be notified of a change as it happens, an HTTP socket must remain open between the client and Sync Gateway. The `feed` querystring option is used in this case and there are 2 different feed types:

- `feed=longpoll`: The response will contain all the changes since the specified sequence. If seq is the last sequence number (the most recent one) then the connection will remain open until a new document is processed by Sync Gateway and the change event is sent.
- `feed=continuous`: In this case, Sync Gateway will hold the connection open forever.

To subscribe to the changes feed you can use any HTTP library to send the `GET /_changes` request with query options. In this lesson however you will use the generated libraries based on the Swagger specs. At Couchbase, we rely heavily on Swagger. It's a great tool to document REST APIs and also automatically generate the HTTP client libraries associated with them.

![](img/image53.png)

- Open **bot/app.js** and locate the `getChanges(seq)` method.
- This method is called to get the changes since the specified sequence.

    ```javascript
    // Use the SwaggerJS module to dynamically load the Swagger spec
    new Swagger({
      url: 'http://developer.couchbase.com/mobile/swagger/sync-gateway-admin/spec.json',
      usePromise: true
    })
      .then(function (res) {
        client = res;
        
        // Start getting changes at seq: 2
        getChanges(2);
        
        function getChanges(seq) {
          // Use the Swagger client to connect to the changes feed
          client.database.get_db_changes({db: 'todo', include_docs: true, since: seq, feed: 'longpoll'})
            .then(function (res) {
              var results = res.obj.results;
              console.log(results.length + ' change(s) received');
              processChanges(results);
              // Get changes since the last sequence
              getChanges(res.obj.last_seq);
            })
            .catch(function (err) {
              console.log(err);
            });
        }
        
        ...

      });
    ```

Notice that the `get_db_changes` method is used with `since: <seq>` and `feed: longpoll` to get a set of changes since a specified sequence and subscribe to more changes later on. The `include_docs: true` option is used to include the document body in the response. Finally the `getChanges(seq)` method is called recursively passing the sequence to the `last_seq` property received in the response.

#### Try it out

- Open a Terminal window in the **bot** directory, install the dependencies and start the bot.

    ```bash
    npm install
    node app.js
    ```

- Make further changes in the application and notice that the number of changes are printed to the console.

    ![](https://cl.ly/1X0M0J2Q450U/image55.gif)

In the next section you will learn how to work with the Admin REST API.

## Bulk operations

A common operation on the Sync Gateway Admin REST API is to persist documents from another system or perform certain operations in response to a specific change. You will extend the changes feed handling code in the previous section to attach an image to **task** documents only if the text is "apple", "coffee" or "potatoes".

Similarly to the previous section, you will use the API methods available on the library provided by Swagger.

- Open **bot/app.js** and locate the `processChanges(results)` method.
- This method is called in `getChanges(seq)` with the changes received from Sync Gateway.

    ```javascript
    // Use the SwaggerJS module to dynamically load the Swagger spec
    new Swagger({
      url: 'http://developer.couchbase.com/mobile/swagger/sync-gateway-admin/spec.json',
      usePromise: true
    })
      .then(function (res) {
        client = res;
        
        ...
        
        function processChanges(results) {
          for (var i = 0; i < results.length; i++) {
            var doc = results[i].doc;
            var img;
            if (!doc._deleted && doc.type == 'task' && !doc._attachments) {
              switch (doc.task.toLowerCase()) {
                case 'apple':
                  img = fs.readFileSync('apple.png');
                  break;
                case 'coffee':
                  img = fs.readFileSync('coffee.png');
                  break;
                case 'potatoes':
                  img = fs.readFileSync('potatoes.png');
                  break;
              }
              if (img) {
                var base64 = img.toString('base64');
                doc._attachments = {
                  image: {
                    content_type: 'image\/png',
                    data: base64
                  }
                };
                client.database.post_db_bulk_docs({db: 'todo', BulkDocsBody: {docs: [doc]}})
                  .then(function (res) {
                    getChanges(results[i].seq);
                  })
                  .catch(function (err) {
                    console.log(err);
                  });
              }
            }
          }
        }

      });
    ```

Here's what the code is doing:

- Checks that the change isn't a deletion (tombstone), that the document type is "task" and that it has no attachments.
- If the `doc.task` property is either "apple", "coffee" or "potatoes" then it reads the corresponding image as a Base64 string and attaches it on the document's `_attachments` field.
- Use the `post_db_bulk_docs` method passing the document to save it as a new revision in Sync Gateway.

#### Try it out

- Run the application and make sure it's replicating to Sync Gateway on your machine.
- Start the bot.

    ```bash
    node app.js
    ```

- Add a task called "Apple", "Coffee" or "Potatoes" and an image should appear after a few seconds. That's the attachment that was added to Sync Gateway by the bot and in turn replicated to Couchbase Lite.

    ![](https://cl.ly/060e3a0p3717/image56.gif)

## Conclusion

Well done! You've completed this lesson on integration by using the Stream API to subscribe to changes and the REST API to persist a document back to Sync Gateway. Feel free to share your feedback, findings or ask any questions on the forums.