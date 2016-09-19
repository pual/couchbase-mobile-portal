---
id: create-database
title: Create a Database
permalink: ready/training/create-database/index.html
---

In this lesson you’ll be introduced to Couchbase Lite, our embedded NoSQL database. You’ll learn how to create a new embedded database and optionally use databases pre-packaged in your application.

[//]: # "COMMON ACROSS LESSONS"

Download the project and Couchbase Lite SDK below.

<block class="ios" />

<div class="buttons-unit downloads">
  <a href="https://cl.ly/2B3I3x1k1s0e/xcode-project.zip" class="button" id="project">
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

<block class="rn ios" />

## Create a new database

The entrypoint in the Couchbase Lite SDK is the [Manager](/documentation/mobile/current/develop/guides/couchbase-lite/native-api/manager/index.html) class. There is no limit to how many databases can be created or opened on the device. You can think of a database as a namespace for documents and several databases can be used in the same app (one database per user of the app is a common pattern). All databases are stored in the application directory.

<block class="ios" />

- Open **AppDelegate.swift** and locate the `openDatabase(username:withKey:withNewKey)` method.
- This method is called when the application starts and contains the code to create a database.

    ```swift
    // TRAINING: Create a database
    let dbname = username
    let options = CBLDatabaseOptions()
    options.create = true

    if kEncryptionEnabled {
        if let encryptionKey = key {
            options.encryptionKey = encryptionKey
        }
    }

    try database = CBLManager.sharedInstance().openDatabaseNamed(dbname, with: options)
    ```

<block class="ios rn" />

Here you're using the `openDatabaseNamed` method where the database is the user currently logged in and `options.create` is set to `true` to create the database.

> **Note:** Database encryption will be covered in the [Adding Security](/documentation/mobile/current/develop/training/adding-security/index.html) lesson so you can safely ignore the `kEncryptionEnabled` constant in this lesson.

### Try it out

<block class="ios" />

- Build and run.
- Create a new list using the '+' navigation bar button on the application's 'Task lists' screen.
- The task list is persisted to the database.
    <img src="img/image40.png" class="portrait" />

<block class="ios rn" />

In the next section, you will learn how to bundle a pre-built Couchbase Lite database in an application.

## Using the pre-built database

You can also bundle a pre-built database in your application if your app needs to sync a lot of static data initially. It can be a lot more efficient to bundle a database in your application and install it on the first launch. Even if some of the content changes on the server after you create the app, the app's first pull replication will bring the database up to date. Here, you will use a pre-built database that contains a list of groceries. Download it from the link below and follow the instructions.

[Download the pre-built database](https://cl.ly/453l3M1O151a/prebuilt-db.zip)

<block class="ios" />

- Unzip the file.
- Open the prebuilt-db folder and drag **todo.cblite2** to the Copy Bundle Resources on the Build Phases tab in Xcode.
    <img src="img/image22.png" class="center-image" />
- Be sure to check the **Copy items if needed** and **Create folder references** options from the dropdown panel.
    <img src="img/skitch.png" class="center-image" />
- Open **AppDelegate.swift** and set `kUsePrebuiltDb` to `true`.

    ```swift
    let kUsePrebuiltDb = true
    ```

- This constant is used in the `installPrebuiltDb` method to determine if the pre-built database should be moved from the application assets folder to the Couchbase Lite directory.

    ```swift
    // TRAINING: Install pre-built database
    guard kUsePrebuiltDb else {
        return
    }
    
    let db = CBLManager.sharedInstance().databaseExistsNamed("todo")
    
    if (!db) {
        let dbPath = Bundle.main.path(forResource: "todo", ofType: "cblite2")
        do {
            try CBLManager.sharedInstance().replaceDatabaseNamed("todo", withDatabaseDir: dbPath!)
        } catch let error as NSError {
            NSLog("Cannot replace the database %@", error)
        }
    }
    ```

<block class="ios rn" />

The prebuilt database is installed using the `replaceDatabaseNamed` method only if there is no existing database called 'todo'. Since you created an empty database called 'todo' in the previous step you must first remove the existing database. The easiest way to do that is by deleting the app.

### Try it out

<block class="ios" />

- Build and run.
- A Groceries list will now be visible on the Lists screen. Click on it to see the tasks.
  <img src="https://cl.ly/3e1J2I0G1U1U/image45.gif" class="portrait" />

<block class="ios rn" />

> **Note:** Refer to the [Database](/documentation/mobile/current/develop/guides/couchbase-lite/native-api/database/index.html) guide to learn how to create **pre-built** databases.

## Conclusion

Well done! You've completed this lesson on creating a database and using a pre-built database. Feel free to share your feedback, findings or ask any questions on the forums.