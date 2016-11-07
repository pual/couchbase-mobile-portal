---
id: create-database
title: Create a Database
permalink: ready/training/develop/create-database/index.html
---

In this lesson you’ll be introduced to Couchbase Lite, our embedded NoSQL database. You’ll learn how to create a new embedded database and optionally use databases pre-packaged in your application.

[//]: # "COMMON ACROSS LESSONS"

<block class="ios" />

#### Requirements

- Xcode 8 (Swift 3)

#### Getting Started

Download the project below.

<div class="buttons-unit downloads">
  <a href="https://cl.ly/293t073Z3735/project.zip" class="button" id="project">
    <img src="img/download-xcode.png">
  </a>
</div>

Unzip the file and install the dependencies using Cocoapods.

```bash
$ cd xcode-project
$ pod install
```

Open **Todo.xcworkspace** in Xcode. Then build & run the project.

<block class="net" />

#### Requirements

- Visual Studio 2015+ (Windows) or Xamarin Studio 6+ (OS X)

#### Getting Started

Download the project below.

<div class="buttons-unit downloads">
  <a href="https://cl.ly/293t073Z3735/project.zip" class="button" id="project">
    <img src="img/download-vs.png">
  </a>
</div>

<block class="android" />

#### Requirements

- Android Studio

#### Getting Started

Download the project below.

<div class="buttons-unit downloads">
  <a href="https://cl.ly/293t073Z3735/project.zip" class="button" id="project">
    <img src="img/download-android.png">
  </a>
</div>

[//]: # "COMMON ACROSS LESSONS"

<block class="all" />

## Create a new database

The entrypoint in the Couchbase Lite SDK is the [Manager](/documentation/mobile/current/develop/guides/couchbase-lite/native-api/manager/index.html) class. There is no limit to how many databases can be created or opened on the device. You can think of a database as a namespace for documents and several databases can be used in the same app (one database per user of the app is a common pattern). The code below creates an empty database.

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

<block class="net" />

```c#
// This code can be found in CoreApp.cs
// in the OpenDatabase(string, string, string) method
var encryptionKey = default(SymmetricKey);
if(key != null) {
    encryptionKey = new SymmetricKey(key);
}

var options = new DatabaseOptions {
    Create = true,
    EncryptionKey = encryptionKey
};

Database = AppWideManager.OpenDatabase(dbName, options);
if(newKey != null) {
    Database.ChangeEncryptionKey(new SymmetricKey(newKey));
}
```

<block class="android" />

```java
// This code can be found in Application.java
// in the openDatabase(username, key, newKey) method
String dbname = username;
DatabaseOptions options = new DatabaseOptions();
options.setCreate(true);

if (mEncryptionEnabled) {
    options.setEncryptionKey(key);
}

Manager manager = null;
try {
    manager = new Manager(new AndroidContext(getApplicationContext()), Manager.DEFAULT_OPTIONS);
} catch (IOException e) {
    e.printStackTrace();
}
try {
    database = manager.openDatabase(dbname, options);
} catch (CouchbaseLiteException e) {
    e.printStackTrace();
}
```

<block class="all" />

Here you're using the `openDatabaseNamed` method where the database is the user currently logged in and `options.create` is set to `true`.

> **Note:** You can ignore the `kEncryptionEnabled` constant. Database encryption will be covered in the [Adding Security](/documentation/mobile/1.3/training/design/security/index.html) lesson.

<block class="all" />

### Try it out

1. Build and run.
2. Create a new list on the application's 'Task lists' screen.
3. The task list is persisted to the database.
    
<block class="ios" />

<img src="img/image40.png" class="portrait" />

<block class="xam" />

<img src="img/image40xa.png" class="portrait" />

<block class="wpf" />

<img src="img/image40w.png" class="portrait" />

<block class="android" />

<img src="img/image40a.png" class="portrait" />

<block class="all" />

## Using the pre-built database

In this section, you will learn how to bundle a pre-built Couchbase Lite database in an application. It can be a lot more efficient to bundle a database in your application and install it on the first launch. Even if some of the content changes on the server after you create the app, the app's first pull replication will bring the database up to date. Here, you will use a pre-built database that contains a list of groceries. The code below moves the pre-built database from the bundled location to the application directory.

<block class="ios" />

```swift
// This code can be found in AppDelegate.swift
// in the installPrebuiltDb() method
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

<block class="net" />

```c#
// This code can be found in CoreApp.cs
// in the InstallPrebuildDB() method
var db = AppWideManager.GetExistingDatabase("todo");
if(db == null) {
    try {
        using(var asset = typeof(CoreApp).Assembly.GetManifestResourceStream("todo.zip")) {
            AppWideManager.ReplaceDatabase("todo", asset, false);
        }
    } catch(Exception e) {
        Debug.WriteLine($"Cannot replicate the database: {e}");
    }
}
```

<block class="android" />

```java
// This code can be found in Application.java
// in the installPrebuiltDb() method
if (!mUsePrebuiltDb) {
    return;
}

try {
    manager = new Manager(new AndroidContext(getApplicationContext()), Manager.DEFAULT_OPTIONS);
} catch (IOException e) {
    e.printStackTrace();
}
try {
    database = manager.getExistingDatabase("todo");
} catch (CouchbaseLiteException e) {
    e.printStackTrace();
}
if (database == null) {
    try {
        ZipUtils.unzip(getAssets().open("todo.zip"), manager.getContext().getFilesDir());
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

<block class="all" />

The prebuilt database is installed using the database replacement API only if there isn't any existing database called 'todo'. Since you created an empty database called 'todo' in the previous step you must first remove the existing database.

### Try it out

<block class="ios" />

1. Open **AppDelegate.swift** and set the `kUsePrebuiltDb` constant to `true`.

    ```swift
    let kUsePrebuiltDb = true
    ```

2. Build and run (don't forget to delete the app first).
3. A Groceries list will now be visible on the Lists screen. Click on it to see the tasks.
  <img src="https://cl.ly/3e1J2I0G1U1U/image45.gif" class="portrait" />

<block class="wpf" />

1. Open **CoreApp.cs** and navigate to the `CoreAppStart.CreateHint()` method.
2. Change the `usePrebuiltDB` on the return value of the function to `true`.

    ```c#
    retVal.usePrebuiltDB = true;
    ```

3. Build and run (don't forget to delete the app first).

4. A Groceries list will now be visible on the Lists screen. Click on it to see the tasks.

    <img src="https://cl.ly/023H36113s2r/image45w.gif" class="center-image" />

<block class="xam" />
**iOS**
<img src="./img/image45.gif" class="portrait" />
**Android**
<img src="./img/image45xa.gif" class="portrait" />

<block class="android" />

1. Open **Application.java** and set the `mUsePrebuiltDb` constant to true.
2. Build and run (don't forget to delete the app first).
3. A Groceries list will now be visible on the Lists screen. Click on it to see the tasks.

    <img src="https://cl.ly/2z4715010K2Z/image45a.gif" class="portrait" />

<block class="all" />

> **Note:** Refer to the [Database](/documentation/mobile/current/develop/guides/couchbase-lite/native-api/database/index.html) guide to learn how to create **pre-built** databases.

## Conclusion

Well done! You've completed this lesson on creating a database. In the next lesson you will learn how to write and query documents from the database. Feel free to share your feedback, findings or ask any questions on the forums.
