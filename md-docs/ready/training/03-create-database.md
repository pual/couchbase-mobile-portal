---
id: create-database
title: Create a Database
permalink: ready/training/develop/create-database/index.html
---

In this lesson you’ll be introduced to Couchbase Lite, our embedded NoSQL database. You’ll learn how to create a new embedded database and optionally use databases pre-packaged in your application.

[//]: # "COMMON ACROSS LESSONS"

#### Requirements

- Xcode 8 (Swift 3)

#### Getting Started

Download the project below.

<block class="ios" />

<div class="buttons-unit downloads">
  <a href="https://cl.ly/1R1t0S1x2j17/xcode-project.zip" class="button" id="project">
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

<block class="rn ios" />

## Create a new database

The entrypoint in the Couchbase Lite SDK is the [Manager](/documentation/mobile/current/develop/guides/couchbase-lite/native-api/manager/index.html) class. There is no limit to how many databases can be created or opened on the device. You can think of a database as a namespace for documents and several databases can be used in the same app (one database per user of the app is a common pattern). The code below creates an empty databse.

<block class="ios" />

```swift
// This code can be found in AppDelegate.swift
// in the openDatabase(username:withKey:withNewKey) method
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

Here you're using the `openDatabaseNamed` method where the database is the user currently logged in and `options.create` is set to `true`.

> **Note:** You can ignore the `kEncryptionEnabled` constant. Database encryption will be covered in the [Adding Security](/documentation/mobile/current/develop/training/adding-security/index.html) lesson.

### Try it out

<block class="ios" />

1. Build and run.
2. Create a new list using the '+' navigation bar button on the application's 'Task lists' screen.
3. The task list is persisted to the database.
    <img src="img/image40.png" class="portrait" />

<block class="ios rn" />

## Using the pre-built database

In this section, you will learn how to bundle a pre-built Couchbase Lite database in an application. It can be a lot more efficient to bundle a database in your application and install it on the first launch. Even if some of the content changes on the server after you create the app, the app's first pull replication will bring the database up to date. Here, you will use a pre-built database that contains a list of groceries. The code below moves the pre-built database from the assets folder to the application directory.

<block class="ios" />

```swift
// This code can be found in AppDelegate.swift
// in the installPrebuiltDb()
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

The prebuilt database is installed using the `replaceDatabaseNamed` method only if there isn't any existing database called 'todo'. Since you created an empty database called 'todo' in the previous step you must first remove the existing database. The easiest way to do that is by deleting the app.

### Try it out

<block class="ios" />

1. Open **AppDelegate.swift** and set the `kUsePrebuiltDb` constant to `true`.

    ```swift
    let kUsePrebuiltDb = true
    ```

2. Build and run (don't forget to delete the app first).
3. A Groceries list will now be visible on the Lists screen. Click on it to see the tasks.
  <img src="https://cl.ly/3e1J2I0G1U1U/image45.gif" class="portrait" />

<block class="ios rn" />

> **Note:** Refer to the [Database](/documentation/mobile/current/develop/guides/couchbase-lite/native-api/database/index.html) guide to learn how to create **pre-built** databases.

## Conclusion

Well done! You've completed this lesson on creating a database. In the next lesson you will learn how to write and query documents from the database. Feel free to share your feedback, findings or ask any questions on the forums.