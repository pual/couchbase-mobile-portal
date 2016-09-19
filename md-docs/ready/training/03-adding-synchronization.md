---
id: adding-synchronization
title: Adding Synchronization
permalink: ready/training/adding-synchronization/index.html
---

In this lesson you’ll be introduced to Sync Gateway, our secure web gateway. You’ll learn how to use Couchbase Lite’s synchronization APIs, set up Sync Gateway for synchronization with the cloud and other devices, and resolve data conflicts within your application.

[//]: # "COMMON ACROSS LESSONS"

Download the project and Couchbase Lite SDK below.

<block class="ios" />

<div class="buttons-unit downloads">
  <a href="https://cl.ly/1x2m2u0Q3w2J/xcode-project.zip" class="button" id="project">
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

> **Tip:** To make things a bit more exciting, you may want to use the pre-built database containing a list of Groceries. Refer to the [Create a Database](/documentation/mobile/current/develop/training/using-the-database/index.html) lesson to learn how to use it.

<block class="ios rn" />

## Install Sync Gateway

Now that your application runs smoothly on the device you are ready to introduce Sync Gateway.

- [Download Sync Gateway](http://www.couchbase.com/nosql-databases/downloads#couchbase-mobile)
- Unzip the file and locate the executable at **~/Downloads/couchbase-sync-gateway/bin/sync_gateway**
- Create a new file called **sync-gateway-config.json** with the following.

```javascript
{
  "interface":":4984",
  "log": ["HTTP", "Auth"],
  "databases": {
    "todo": {
      "server": "walrus:",
      "users": {
  		  "GUEST": {"disabled": false, "admin_channels": ["*"] }
	    }
    }
  }
}
```

During development, you can set the **server** property to **walrus:** (also known as the Walrus mode) and it will keep the data in memory. Note that anytime you restart Sync Gateway in walrus mode, the database will be empty.

By default, Sync Gateway doesn't allow unauthenticated requests to be processed for security reasons. So you're enabling the **GUEST** user which represents all the unauthenticated clients that will be synchronizing with your Sync Gateway instance.

> **Note:** User authentication is covered in more detail in the [Adding Security](/documentation/mobile/current/develop/training/adding-security/index.html) lesson.

### Try it out

- Start Sync Gateway from the command-line in your project directory.

    ```bash
    $ sync_gateway sync-gateway-config.json
    ```

<block class="ios rn"/>

Sync Gateway is listening on two ports:

- 4984 is the public port which will be used from the application.
- 4985 is the admin port used for administrative tasks (for security reasons, it’s only accessible on localhost).

## Add synchronization

Typically, an application needs to send data to the server and receive it. In Couchbase Mobile, this is handled by replications which run on the device. A replication requires a Couchbase Lite database and a Sync Gateway URL, and synchronizes data between the two. They can be of two types:

- **Push:** The data is pushed from Couchbase Lite to Sync Gateway.
- **Pull:** The data is pulled from Sync Gateway to Couchbase Lite.

There are a few terminologies that designate the role of each database involved in a replication:

- **Source:** The database where the data is read.
- **Target:** The database where the data is written.
- **Local:** The database that resides where the replication is running.
- **Remote:** The database to which the replication is sending data.

To get synchronization running, do the following:

<block class="ios" />

- Open **AppDelegate.swift** and locate the `startReplication(withUsername, andPassword)` method.
- This method is called in `applicationDidFinishLaunchingWithOptions` when the application starts.

    ```swift
    // TRAINING: Start push/pull replications
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

Here you are starting a push and pull replication to have bi-directional sync with the remote Sync Gateway.

`kSyncGatewayUrl` is a constant in **AppDelegate.swift** and represents the URL to the Sync Gateway database (http://localhost:4984/todo/).

If the application is running on a phone, you must replace **localhost** by the internal IP of the machine running Sync Gateway and ensure that the phone and laptop are connected to the same network. You can change this value to any valid URL pointing to a Sync Gateway database on the cloud, for example.

### Try it out

<block class="ios" />

- Set `kSyncEnabled` to true in **AppDelegate.swift**.

    ```swift
    let kSyncEnabled = true
    ```

- Build and run.
- Open [http://localhost:4985/_admin/db/todo](http://localhost:4985/_admin/db/todo) in the browser and notice that all the documents are pushed to Sync Gateway! You may have more or less rows depending on whether you used the pre-built database.

![](./img/image19.png)

<block class="ios rn" />

## Resolve Conflicts

Due to the unpredictability of mobile connections it's inevitable that more than one device will update the same document simultaneously. Couchbase Lite provides features to resolve these conflicts. The resolution rules are written by the developer to keep full control over which edit (also called a revision) should be picked.

Couchbase Lite uses revisions to resolve conflicts detected during replication. One significant difference from other databases is document versioning. Couchbase Lite uses a technique called Multi-version Concurrency Control (MVCC) to manage conflicts between multiple writers. Document versioning is similar to the check-and-set mechanism (CAS) of Couchbase Server, except that in Couchbase Lite versioning is required rather than optional and the token is a UUID rather than an integer.

Revisions form a tree data structure since they can have multiple branches. Here is an example of a revision tree which depicts the act of resolving a conflict by deleting one branch of the revision tree by adding a tombstone revision, and then adding a new rev to the winning branch.

<img src="img/image16.png" class="portrait" />

The revision number follows a specific format.

<img src="img/image20.png" class="center-image" />

A conflict usually occurs when two writers are offline and save a different revision of the same document. In this application, it could occur when:

- Two database instances update the name of an existing list.
- Two database instances update an existing task. One updates the task’s **name** while the other updates the **complete** property.

To resolve conflicts you must first learn how to detect them. You will use the **allDocs** query with a few options. 

The **allDocs** query allows you to query an index of all the documents in the local database. The options that you will use define a query that will only return documents for which there are conflicting revisions. A **LiveQuery** automatically refreshes every time the database changes.

<block class="ios" />

- Open **AppDelegate.swift** and locate the `startConflictLiveQuery` method.
- This method is called in `applicationDidFinishLaunchingWithOptions`.

    ```swift
    guard kConflictResolution else {
        return
    }

    // TRAINING: Detecting when conflicts occur
    conflictsLiveQuery = database.createAllDocumentsQuery().asLive()
    conflictsLiveQuery!.allDocsMode = .onlyConflicts
    conflictsLiveQuery!.addObserver(self, forKeyPath: "rows", options: .new, context: nil)
    conflictsLiveQuery!.start()
    ```

- The KVO method then calls the `resolveConflicts` method when a new conflict is detected.

    ```swift
    // TRAINING: Responding to Live Query changes
    override func observeValue(forKeyPath keyPath: String?, of object: Any?,
                                change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
        if object as? NSObject == conflictsLiveQuery {
            resolveConflicts()
        }
    }
    ```

<block class="ios rn" />

Here’s how you will resolve conflicts:

- **task-list** conflicts: keep the default; the winner picked by Couchbase Lite will remain the current revision. However, you need to clean up the extraneous revision.
- **task** conflicts: n-way merge; changes from all conflicting revisions are merged into a dictionary and saved to a new revision.

Before you resolve the conflict you must learn how to create a conflict and what happens when there is no handling of conflicts.

### Automatic: task-list documents

Even if the conflict isn’t resolved, Couchbase Lite has to return something. It chooses one of the two conflicting revisions as the "winner". The choice is deterministic, which means that every device that is faced with the same conflict will pick the same winner, without having to communicate.

It’s not all that easy to create conflicts in development because you would have to deploy the app to multiple devices or simulators and stop the replications to create the conflict. So before you learn how to resolve conflicts, you will find the code to create one only for development purposes.

<block class="ios" />

- Open **ListsViewController.swift** and locate the `motionEnded(_:with)` method.
- This method is called when the shake gesture is detected.

    ```swift
    // TRAINING: Create task list conflict (for development only)
    let savedRevision = createTaskList(name: "Test Conflicts List")
    let newRev1 = savedRevision?.createRevision()
    let propsRev1 = newRev1?.properties
    propsRev1?.setValue("Update 1", forKey: "name")
    let newRev2 = savedRevision?.createRevision()
    let propsRev2 = newRev2?.properties
    propsRev2?.setValue("Update 2", forKey: "name")
    do {
        try newRev1?.saveAllowingConflict()
        try newRev2?.saveAllowingConflict()
    } catch let error as NSError {
        NSLog("Could not create document %@", error)
    }
    ```

<block class="ios rn" />

Here, you are using the **Revision** which is one layer below the **Document** API. The **Revision** has a method called `saveAllowingConflicts` which is helpful in this scenario to create two conflicting revisions.

#### Try it out

<block class="ios" />

- Build and run.
- On the list screen, shake the device or use the **^⌘Z** shortcut on the simulator.
- A new list called 'Update 1' is added. In fact this is a conflict but the other revision, 'Update 2', is not visible because it wasn't picked as the winner.
    <img src="img/image06.png" class="portrait" />

- To pick a winner, it compares the revision numbers and picks the higher one sorted lexicographically.
- Delete that list and notice that 'Update 2' is now displayed. Since you deleted the current revision and there is another revision on another branch of the tree, Couchbase Lite promotes the other one as the **current** revision.
    ![](img/image51.png)

<img src="https://cl.ly/023t2k3w3k3c/image47.gif" class="portrait" />

<block class="ios rn" />

This can be surprising at first but it’s the strength of using a distributed database that defers the conflict resolution logic to the application. It’s your responsibility as the developer to ensure conflicts are resolved! Even if you decide to let Couchbase Lite pick the winner you must remove extraneous conflicting revisions to prevent the behaviour observed above.

<block class="ios" />

- Open **AppDelegate.swift** and locate the `resolveConflicts(revisions revs: [CBLRevision])` method.
- This method is called for every row return by the conflicting revisions query.
 
```swift
let rows = conflictsLiveQuery?.rows
while let row = rows?.nextRow() {
    if let revs = row.conflictingRevisions, revs.count > 1 {
        let defaultWinning = revs[0]
        let type = (defaultWinning["type"] as? String) ?? ""
        switch type {
        // TRAINING: Automatic conflict resolution
        case "task-list", "task-list.user":
            let props = defaultWinning.userProperties
            let image = defaultWinning.attachmentNamed("image")
            resolveConflicts(revisions: revs, withProps: props, andImage: image)
        // TRAINING: N-way merge conflict resolution
        case "task":
            let merged = nWayMergeConflicts(revs: revs)
            resolveConflicts(revisions: revs, withProps: merged.props, andImage: merged.image)
        default:
            break
        }
    }
}
```

<block class="ios rn" />

First, you're checking if there is a conflict (i.e is there strictly more than 1 revision). Then, a switch statement is used to check for the document type. In the case of a "task-list" document, the `resolveConflicts(revisions:withProps:andImage)` method will only keep the current revision.

#### Try it out

<block class="ios" />

- To enable conflict resolution, set the `kConflictResolution` constant in **AppDelegate.swift** to true.

    ```swift
    let kConflictResolution = true
    ```

- Perform the same actions and this time deleting the list conflict doesn’t reveal the subsequent conflicting revision anymore.
    <img class="portrait" src="https://cl.ly/1N29282B3A0M/image48.gif"  />

<block class="ios rn" />

### N-way merge

For task documents, you will follow the same steps as previously except this time the conflict resolution will merge the differences between the conflicting revisions into a new revision.

<block class="ios" />

- Open **ListsViewController.swift** and locate the `motionEnded(_:with)` method.
- This method is called when the shake gesture is detected. Here, one revision changes the **task** property to **Update 1** and the other sets **complete** to **true**.

    ```swift
    // TRAINING: Create task conflict (for development only)
    let savedRevision = createTask(task: "Test Conflicts Task")
    let newRev1 = savedRevision?.createRevision()
    let propsRev1 = newRev1?.properties
    propsRev1?.setValue("Update 1", forKey: "task")
    let newRev2 = savedRevision?.createRevision()
    let propsRev2 = newRev2?.properties
    propsRev2?.setValue(true, forKey: "complete")
    do {
        try newRev1?.saveAllowingConflict()
        try newRev2?.saveAllowingConflict()
    } catch let error as NSError {
        NSLog("Could not save revisions %@", error)
    }
    ```

<block class="ios rn" />

#### Try it out

<block class="ios" />

- Disable conflict resolution to see what happens when conflicts are not resolved.

    ```swift
    let kConflictResolution = false
    ```

- Build and run.
- Open a new list and shake the device or use the **^⌘Z** shortcut to create a new task conflict. Here the winning revision is the one that set the **completed** property to true.
    <img src="./img/image23.png" class="portrait" />

- Delete the task. The revision that modified the **task** value to 'Update 1' becomes the **current revision**.
    <img src="./img/image26.png" class="portrait" />

Similarly to the previous section, you will learn how to resolve conflicts, this time for "task-list" documents. In this case, the resolution code will **merge the updates** (i.e n-way merge) of the conflicting revisions before promoting it as the current revisions.

- Open **AppDelegate.swift** and locate the `resolveConflicts` method.

```swift
let rows = conflictsLiveQuery?.rows
while let row = rows?.nextRow() {
    if let revs = row.conflictingRevisions, revs.count > 1 {
        let defaultWinning = revs[0]
        let type = (defaultWinning["type"] as? String) ?? ""
        switch type {
        // TRAINING: Automatic conflict resolution
        case "task-list", "task-list.user":
            let props = defaultWinning.userProperties
            let image = defaultWinning.attachmentNamed("image")
            resolveConflicts(revisions: revs, withProps: props, andImage: image)
        // TRAINING: N-way merge conflict resolution
        case "task":
            let merged = nWayMergeConflicts(revs: revs)
            resolveConflicts(revisions: revs, withProps: merged.props, andImage: merged.image)
        default:
            break
        }
    }
}
```

Notice that for 'task' documents, the `nWayMergeConflicts` method is used to merge the diffs of conflicting revisions before calling the `resolveConflicts(revisions:withProps:andImage)`.

#### Try it out

- Enable conflict resolution.

    ```swift
    let kConflictResolution = true
    ```

- Build and run. 
- Create a task conflict using the shake gesture (or **^⌘Z**) and this time the row contains the updated text **and** is marked as completed.
    ![](img/image03.png)

<block class="ios rn" />

## Conclusion

Well done! You've completed this lesson on enabling synchronization, detecting and resolving conflicts. In the next lesson you'll learn how to implement authentication and define access control rules in the Sync Function. Feel free to share your feedback, findings or ask any questions on the forums.
