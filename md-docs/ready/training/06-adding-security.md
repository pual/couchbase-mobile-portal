---
id: adding-security
title: Adding Security
permalink: ready/training/develop/adding-security/index.html
---

In this lesson you’ll learn how to add security to your Couchbase Mobile application. You’ll implement authentication and define access control, data validation, and access grant policies.

[//]: # "COMMON ACROSS LESSONS"

#### Requirements

- Xcode 8 (Swift 3)

#### Getting Started

Download the project below.

<block class="ios" />

<div class="buttons-unit downloads">
  <a href="https://cl.ly/2A2D3q3R2d1g/xcode-project.zip" class="button" id="project">
    <img src="img/download-xcode.png">
  </a>
</div>

Unzip the file and install the dependencies using Cocoapods.

```bash
$ cd xcode-project
$ pod install
```

Open **Todo.xcodeproj** in Xcode. Then build & run the project.

<img src="img/image42.png" class="center-image" />

Throughout this lesson, you will navigate in different files of the Xcode project. We recommend to use the method navigator to scroll to a method.

<img src="https://cl.ly/0G263m3m1a0w/image44.gif" class="center-image" />

[//]: # "COMMON ACROSS LESSONS"

<block class="ios rn" />

## Implement User Authentication

### Install Sync Gateway

Users are created with a name/password on Sync Gateway which can then be used on the Couchbase Lite replicator to authenticate as a given user. You can create users by hardcoding the user's name/password in the configuration file. Create a new file called **sync-gateway-config.json** with the following.

```javascript
{
  "log": ["HTTP", "Auth"],
  "databases": {
    "todo": {
      "server": "walrus:",
      "users": {
        "user1": {"password": "pass"},
        "user2": {"password": "pass"}
      }
    }
  }
}
```

#### Try it out

1. [Download Sync Gateway](http://www.couchbase.com/nosql-databases/downloads#couchbase-mobile)
2. Unzip the file and locate the executable at **~/Downloads/couchbase-sync-gateway/bin/sync_gateway**.
3. Start it from the command-line with the config file.

    ```bash
    $ /path/to/sync_gateway sync-gateway-config.json
    ```

4. Two users are now visible at [http://localhost:4985/_admin/db/todo/users](http://localhost:4985/_admin/db/todo/users).

### User Authentication

With Sync Gateway users defined you can now enable authentication on the Couchbase Lite replicator. The code below creates two replications with authentication.

```swift
// This code can be found in AppDelegate.swift
// in the startReplication(withUsername:andPassword:) method
pusher = database.createPushReplication(kSyncGatewayUrl)
pusher.continuous = true
NotificationCenter.default.addObserver(self, selector: #selector(replicationProgress(notification:)),
    name: NSNotification.Name.cblReplicationChange, object: pusher)

puller = database.createPullReplication(kSyncGatewayUrl)
puller.continuous = true
NotificationCenter.default.addObserver(self, selector: #selector(replicationProgress(notification:)),
                                        name: NSNotification.Name.cblReplicationChange, object: puller)

if kLoginFlowEnabled {
    let authenticator = CBLAuthenticator.basicAuthenticator(withName: username, password: password!)
    pusher.authenticator = authenticator
    puller.authenticator = authenticator
}

pusher.start()
puller.start()
```

<block class="ios rn" />

The `CBLAuthenticator` class has static methods for each authentication method supported by Couchbase Lite. Here, you're passing the name/password to the `basicAuthenticatorWithName` method. The object returned by this method can be set on the replication's `authenticator` property.

#### Try it out

<block class="ios" />

1. Set `kLoginFlowEnabled` to `true` in **AppDelegate.swift**.

    ```swift
    let kLoginFlowEnabled = true
    ```

2. Build and run.

3. Now login with the credentials saved in the config file previously (**user1/pass**) and create a new list. Open the Sync Gateway Admin UI at [http://localhost:4985/_admin/db/todo](http://localhost:4985/_admin/db/todo), the list document is successfully replicated to Sync Gateway as an authenticated user.

    ![](img/image35.png)

> **Note:** You can remove the local database and check if the pull replication retrieves the documents now present on Sync Gateway. On macOS, use the [SimPholders](https://simpholders.com/) utility app to quickly find the data directory of the application and delete the database called **user1**. Then restart the app and you'll notice that the "Today" list isn't displayed. That is, the list document wasn't replicated from Sync Gateway to Couchbase Lite. Indeed, the document is not routed to a channel that the user has access to. **Channel** and **access** are new terms so don't worry, we'll cover what they mean in the next section.
<video src="https://d3vv6lp55qjaqc.cloudfront.net/items/1s1G3C1i2a0G2P3o0G0m/movie1.mp4" controls="true" poster="https://cl.ly/1W2T3w463S0f/image72.png"></video>

## Define Access Control Policies

In order to give different users access to different documents, you must write a sync function. The sync function lives in the configuration file of Sync Gateway. It’s a JavaScript function and every time a new document, revision or deletion is added to a database, the sync function is called and given a chance to examine the document.

In the sync function, you can use different API methods to route documents to channels, grant users access to channels and even assign roles to users. Open the data modeling lesson in a new tab, it will be useful throughout this section.

Sync Functions can be relatively long so it's recommended to write a small piece of code and test that it does what you expect before moving on to the next clause. To achieve this, you will use the Admin UI to make changes to the Sync Function and the `/{db}/_bulk_docs` endpoint to add documents and then check the result in the Admin UI.

![](img/image39.png)

With walrus enabled in the **todos** database, every time you change the Sync Function, the instance of Sync Gateway is restarted with an empty database. That's why you must add them again to check for the result. To get started, do the following:

- Open the Admin UI on the Sync tab where you can edit the Sync Function [http://localhost:4985/_admin/db/todo/sync](http://localhost:4985/_admin/db/todo/sync)
- Open a Terminal window to send documents using curl

### Data Validation

The Sync Function takes two arguments:

- **doc:** The current revision being processed.
- **oldDoc:** The parent revisions if it's an update operation and `null` if it's a create operation.

The first operation to perform in the Sync Function is schema validation. If a document doesn't honour the expected document schema the **throw** function can be used to reject the operation. The following code rejects the operation if the document doesn't have a `type` property.

```javascript
function(doc, oldDoc){
  /* Validate */
  // Document type is required and immutable.
  if (!doc.type || (oldDoc && doc.type != oldDoc.type)) {
  	throw({forbidden: "type property missing"});
  }
}
```

#### Try it out

1. Copy the code above in the Sync Function input box of the Admin UI.
2. Then click on the **Deploy To Server** button. It will restart Sync Gateway with the updated sync function.

    <img src="https://cl.ly/3n2L071q051g/image37.gif" class="center-image" />

    > **Note:** It won't update the config file on your filesystem, make sure to copy/paste from the Admin UI to the config once you are happy with your changes.

3. Use the following curl command to add two documents. One has the `type` property and the second doesn't. Notice that the user credentials (user1/pass) are passed in the URL.

    ```bash
    curl -vX POST 'http://user1:pass@localhost:4984/todo/_bulk_docs' \
          -H 'Content-Type: application/json' \
          -d '{"docs": [{"type": "task-list", "name": "Groceries"}, {"names": "Today"}]}'

    [
      {"id":"e498cad0380e30a86ed5572140c94831","rev":"1-e4ac377fc9bd3345ddf5892b509c4d79"},
      {"error":"forbidden","reason":"type property missing","status":403}
    ]
    ```

    As expected, the response contains the _type property missing_ error message for the document that doesn't have the `type` property.

### Write Permissions

The sync function should handle every document **type** present in your application. Based on the document type and user logged in, it must ensure the user has write permissions for the given operation. If the user sending this document doesn't have the required privileges then the sync function should reject that operation. The folloing methods are used in the Sync Function to check the user's identity and role(s).

- **requireUser([]string names)**: authorizes a document update by rejecting it unless it's made by a specific user or users.
- **requireRole([]string roles)**: authorizes a document update by rejecting it unless the user making it has a specific role or roles.
- **requireAccess([]string channels)**: authorizes a document update by rejecting it unless the user making it has access to at least one of the given channels.

The following code first ensures that the `doc.owner` is indeed the user performing this operation. If that's not the case, the `} catch (e) {` block is called and another check is performed that checks if the user has the `moderator` role. If none of those conditions are met then the document is rejected.

```javascript
if (doc.type == "task-list") {
  /* Control Write Access */
  if (!oldDoc) {
    try {
      // Users can create/update lists for themselves.
      requireUser(doc.owner);
    } catch (e) {
      // Moderators can create/update lists for other users.
      requireRole("moderator");
    }
  }
}
```

#### Try it out

1. Copy the code above in the Sync Function input box of the Admin UI.
2. Then click on the **Deploy To Server** button. It will restart Sync Gateway with the updated sync function.
3. Add the following list documents.

    ```bash
    curl -vX POST 'http://user1:pass@localhost:4984/todo/_bulk_docs' \
          -H 'Content-Type: application/json' \
          -d '{"docs": [{"name": "Today", "type": "task-list", "owner": "user1"}, {"name": "Groceries", "type": "task-list", "owner": "user2"}]}'
          
    [
      {"id":"49c9d5cbbc1cfa05d253f4847d02b8a0","rev":"1-477f7c4f36a31ce9e86a5f465b168b8a"},
      {"error":"forbidden","reason":"missing role","status":403}
    ]
    ```

    The **Groceries** list is successfully rejected because the `owner` property is set to `user2` but the request authenticates as `user1`. Additionally, `user1` doesn't have the `moderator` role so it can't create a list for other users.

4. You can assign a role to a user using the `/{db}/_user` endpoint on the admin port.

    ```bash
    curl -H 'Content-Type: application/json' \
          -vX PUT 'http://localhost:4985/todo/_user/user1' \
          -d '{"admin_roles": ["moderator"]}'

    200 OK
    ```

    With user1 as a moderator, run the curl command on the `/{db}/_bulk_docs` endpoint again and this time both lists are persisted.

### Validating Changes

In this section you will add some validation logic for list documents. There are 3 types of validation:

- **Schema validation:** A document that doesn’t follow the schema specified in your application could have an impact on other clients when they receive it. That’s why you must check for the existence of the properties required in your application.
- **Key validation:** Some document keys may follow a specific format (prefix.uuid) that will be used throughout your system so making sure that it follows the desired format is good practice.
- **Read-only validation:** You may decide that some properties should be immutable and cannot change during an update operation.

```javascript
function validateNotEmpty(key, value) {
  if (!value) {
    throw({forbidden: key + " is not provided."});
  }
}

function validatePrefix(key, value, prefix) {
  console.log(value.split("."))
  if (value.split(".").length != 2) {
    throw({forbidden: key + " should be of the form {" + prefix + "}.{...}"});
  }
}

function validateReadOnly(key, doc, oldDoc) {
  if (oldDoc) {
    if (doc[key] != oldDoc[key]) {
      throw({forbidden: key + " is immutable."});
    }
  }
}
```

There are 3 types of validation performed here:

- **validateNotEmpty**: checks that the given property exists.
- **validatePrefix**: validates that the value conforms to the `prefix.{...}` format.
- **validateReadOnly**: enforces immutability on the property's value.

With those utility functions you can now write the validation logic for documents of type "task-list". Append the following inside the "task-list" if statement on the Admin UI.

The following code ensures that the document passed in the Sync Function meets the schema requirements.

```javascript
/* Validate */
if (!doc._deleted) {
  // Validate required fields.
  validateNotEmpty("name", doc.name);
  validateNotEmpty("owner", doc.owner);
  // Validate that the _id is prefixed with the owner.
  validatePrefix("_id", doc._id, "owner");
  // Don't allow task-list ownership to be changed.
  validateReadOnly("owner", doc, oldDoc);
}
```

Validation rules can be repetitive to write in the sync function so you will define 3 utility methods. Add the following before the last curly brace.

This validation is very similar to the one you added in the first step for document types. You can go ahead and click **Deploy To Server**.

### Routing

The sync function API provides several functions that you can use to route documents. The routing functions assign documents to channels. The main function for document routing is the following:

- **channel([]string channels)**: routes the document to the named channel(s).

Once the write access permissions and validation have been applied successfully, you insert the document in a channel. You can think of it as tagging lots of documents with the same label. There is no limit to how many documents can be mapped to a channel and you can create as many channels as you want.

Append the following inside the "task-list" if statement on the Admin UI.

```javascript
/* Route */
// Add doc to task-list's channel.
channel("task-list." + doc._id);
channel("moderators");
```

Click the **Deploy To Server**  and reload the page. Then add the following documents from the command line:

```bash
curl -vX POST 'http://user1:pass@localhost:4984/todo/_bulk_docs' \
      -H 'Content-Type: application/json' \
      -d '{"docs": [{"_id": "user1.2F5FB3E4-CD95-49F7-B232-5836F5869BEA", "name": "Today", "type": "task-list", "owner": "user1"}, {"_id": "user1.931645E7-84E0-46D1-82E0-0B39E5A70162", "name": "Groceries", "type": "task-list", "owner": "user1"}]}'

[
  {"id":"user1.2F5FB3E4-CD95-49F7-B232-5836F5869BEA","rev":"1-477f7c4f36a31ce9e86a5f465b168b8a"},{"id":"user1.931645E7-84E0-46D1-82E0-0B39E5A70162","rev":"1-7008921932d980b285d18c173e0dff1f"}
]
```

Both documents are saved and mapped to the corresponding channels in the Admin UI. The video below shows you how to navigate through the different channels.

[//]: # "TODO: Link to video."
<video src="https://cl.ly/3i2w062z1w1F/movie2.mp4" controls="true"></video>

### Grants

Based on the routing to channels, the sync function can enable users to access those channels. The API methods to perform a grant operation are the following:

- **access([]string users, []string channels)**: grants access to a channel to a specified user.
- **role([]string users, []string roles)**: grants a user a role, indirectly giving them access to all channels granted to that role.

Append the following inside the "task-list" if statement on the Admin UI.

```javascript
/* Grant Read Access */
// Grant task-list owner access to the task-list, its tasks, and its users.
access(doc.owner, "task-list." + doc._id);
access(doc.owner, "task-list." + doc._id + ".users");
```

This time click the **Live Preview Mode** button. Click the **random** button to pick a document at random and run it through the sync function again, generating the channels and this time the list of users that have access to each channel.

![](img/image38.png)

Here's a table that compares the Live Preview Mode and Deploy To Server. For channel and access changes, the Live Preview Mode is very useful. Just make sure to copy the changes to your config file once you're done!

|FAQ|Live Preview Mode|Deploy To Server|
|:--|:----------------|:---------------|
|Will it persist the changes to disk?|No|No|
|Will it persist the changes when reloading the Admin UI?|No|Yes|
|Will it keep the documents in the walrus database so I don't have to add them again?|Yes|No|

As you have learned in this section, writing a sync function for each document type is done in 4 stages: write permissions, validating changes, routing, read permissions.

![](img/image15.png)

If a document makes it through step 1 and 2 it will be written to Sync Gateway. Step 3 and 4 are used to defined the access control.

In the next section you will learn how to encrypt the local database on the device to provide additional security.

## Database Encryption

The Couchbase Lite API allows you to encrypt the database on the device. By providing an encryption key, all the data stored in the database will be secure. To decrypt it in the future, the same key must be used.

> **Note:** There are two storage types available in Couchbase Lite, _ForestDB_ and _SQLite_. This tutorial only covers _SQLite_ but you can refer to the documentation to find the instructions to include _ForestDB_ in your project.

You must first include the correct dependencies before enabling encryption. 

<block class="ios" />

- Locate the `openDatabase(username:withKey:newKey)` method in **AppDelegate.swift**, this method is called from `startSession` which is in turn getting called in `processLogin` after the user has logged in.
- The Database's `openDatabaseName` is used to create an encrypted database.

```swift
let dbname = username
let options = CBLDatabaseOptions()
options.create = true

if kEncryptionEnabled {
    if let encryptionKey = key {
        options.encryptionKey = encryptionKey
    }
}

try database = CBLManager.sharedInstance().openDatabaseNamed(dbname, withOptions: options)
if newKey != nil {
    try database.changeEncryptionKey(newKey)
}
```

### Try it out

- Drag **Extras/libsqlcipher.a** from the Couchbase Lite SDK folder to the Frameworks folder in the Xcode project.
  ![](img/image50.png)

- In the **Build Phases** tab, remove **libsqlite3.tbd** from the **Link Binary With Libraries** section and add **libsqlcipher.a**.
  ![](https://cl.ly/1K2Q1k3V473l/image49.gif)

- Set `kEncryptionEnabled` to true in **AppDelegate.swift**.

  ```swift
  let kEncryptionEnabled = true
  ```

- Delete the application if you ran it previously without encryption.
    <img src="img/image17.png" class="portrait" />

- Build and run.
- Browse to the database file and you'll find that it's now encrypted.

## Offline Login

What about the scenario where a user attempts to login while being offline? Since data may already be in the database, it would be nice to allow the app to use it. From the previous section, you’ve learned that the database can be safely accessed with an encryption key so the user can attempt to enter credentials as if they were online (the **name** and **password** of the Sync Gateway user). To achieve offline login, you are mapping the user’s **name** to be the **database name** and the user’s **password** to be the **encryption key**.

If the user enters an incorrect password (that is not the valid encryption key), the database will return an error with a status code of 401 to indicate it’s the wrong password.

To make it even more challenging, consider the scenario where a user’s password has been modified on Sync Gateway. Since the encryption key is the old password it will not be accessible, so when the user’s password changes on Sync Gateway the local encryption key and server password will be out of sync. To detect that, you’re going to use the replication change event and check for the status code. If it’s a 401, you will logout the user and display the login screen. However, the data stored in the database will have changes that were not pushed to Sync Gateway yet. You can offer users the ability to update the encryption key only if they remember their old password (because it’s the only way to open and read data from the database encrypted with the old password). If they don’t remember it, the changes made while they were offline will be lost. The marble diagram below shows every scenario and outcome.

<img src="img/image24.png" class="center-image" />

In **AppDelegate.swift**, scroll to the `processLogin` method. Notice that if an error with status 401 is thrown it calls `handleEncryptionError` which displays a popup with an input text and two options.

<img src="img/image27.png" class="portrait" />

When the user clicks **Delete** it will remove the database and create a new one with no data in it. If the user remembers the old password and clicks **Migrate** it will call the `processLogin` method again passing it the old and new encryption keys. The following code at the end of the `openDatabase` method is changing that encryption key to the user's new password.

```swift
if newKey != nil {
    try database.changeEncryptionKey(newKey)
}
```

Build and run. Change the password for the user with the following curl request.

```bash
curl -vX PUT 'http://localhost:4985/todo/_user/user1' \
      -H 'Content-Type: application/json' \
      -d '{"name": "user1","password": "newpass"}'
```

Add a new list which will wake up the replicator to push the new document. Since the user credentials have changed, the task list will not be persisted to Sync Gateway. The way to get notified of this is by checking the replication `lastError` property in the replication change event. In this case it will be a **401 Unauthorized**.

The **lastError** property is checked in the `replicationProgress` method of **AppDelegate.swift**.

```swift
func replicationProgress(notification: NSNotification) {
    UIApplication.sharedApplication().networkActivityIndicatorVisible =
        (pusher.status == .Active || puller.status == .Active)
    let error = pusher.lastError ?? puller.lastError
    if (error != syncError) {
        syncError = error
        if let errorCode = error?.code {
            NSLog("Replication Error: %@", error!)
            if errorCode == 401 {
                Ui.showMessageDialog(
                    onController: self.window!.rootViewController!,
                    withTitle: "Authentication Error",
                    withMessage:"Your username or password is not correct.",
                    withError: nil,
                    onClose: {
                        self.logout()
                })
            }
        }
    }
}
```

If the error code is 401 it displays the popup below and logs the user out of the application.

<img src="./img/image18.png" class="portrait" />

Click **OK** and on the login screen type **user1** and **newpass**, the new password on the login screen.

<img src="./img/image00.png" class="portrait" />

Click **Login**. Next, type **pass** on the "Password Changed" popup, the previous password.

<img src="./img/image14.png" class="portrait" />

Click **Migrate**. Notice that the list added after the password changed is there.

<img src="./img/image10.png" class="portrait" />

### Key Rotation

Key rotation is defined as the process of decrypting data with an old key and re-keying the data with a new one. The benefits of key rotation are all centered on security; for example, if the password to sensitive data is being shared between many users, you may decide to use key rotation to add an extra layer of security. By regularly changing the password you will mitigate the scenario where the encryption key can be compromised unknowingly. Rotating your keys offers more protection and better security for your sensitive business data, but it is not a requirement and it should be considered on a per-application basis.

<block class="ios rn" />

## Conclusion

Well done! You've completed this lesson on adding authentication, writing a sync function and adding database encryption. Feel free to share your feedback, findings or ask any questions on the forums.