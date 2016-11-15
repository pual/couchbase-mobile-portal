---
id: manager
title: Manager
permalink: ready/guides/couchbase-lite/native-api/manager/index.html
---

A `Manager` is the top-level object that manages a collection of Couchbase Lite `Database` instances. You need to create a `Manager` instance before you can work with Couchbase Lite objects in your Application.

## Creating a manager

You create a Manager object by calling a constructor or initializer on the Manager class.

<div class="tabs"></div>

```objective-c+
CBLManager *manager = [CBLManager sharedInstance];
if (!manager) {
		NSLog(@"Cannot create Manager instance");
}
```

```swift+
let manager = CBLManager.sharedInstance()
if manager == nil {
	NSLog("Cannot create Manager Instance")
}
```

```java+
JavaContext context = new JavaContext();
manager = new Manager(context, Manager.DEFAULT_OPTIONS);
```

```android+
AndroidContext androidContext = new AndroidContext(getApplicationContext());
manager = new Manager(androidContext, Manager.DEFAULT_OPTIONS);
```

```c+
var manager = Manager.SharedInstance;
```

## Dude, where's my database file?

The Manager creates a directory in the filesystem and stores databases inside it. Normally, you don't need to care where that is -- your application shouldn't be directly accessing those files. But sometimes it does matter.

- Android: The directory is the location returned by the Android Context object's `getFilesDir()`.
- iOS: `Application Support/CouchbaseLite/`
- macOS: `~/Library/Application Support/com.example.YourAppBundleID/CouchbaseLite/`

> **Note:** One notable case where the location can be important is on iOS: Apple's app review process tries to make sure that the only application data that will be backed up to iCloud is data created by the user. So it's a red flag when, on first launch, an app creates data in backed-up locations (including the Documents and Application Support directories) without user action. Unfortunately, that will happen if your app starts a pull replication on launch, or installs a pre-populated database. Some apps using Couchbase Lite have had their App Store approval held up for this reason!

On iOS or Mac OS you can change the location of the databases by instantiating the Manager via a constructor/initializer that takes a path as a parameter. This directory will be created if it doesn't already exist. (Of course you should be consistent about what path to use, since if you change the path the application won't be able to find any already-existing databases.)

On Android, you can subclass `com.couchbase.lite.android.AndroidContext` and override its `getFilesDir` method to 
return the desired directory.

<div class="tabs"></div>

```objective-c+
NSString* dir = WhereIWantCBLStuffToGo();
NSError* error;
self.manager = [[CBLManager alloc] initWithDirectory: dir
																						 options: NULL
																							 error: &error];
if (!manager) {
		NSLog(@"Cannot create Manager instance: %@", error);
}
```

```swift+
let dir = WhereIWantCBLStuffToGo()
var error: NSError?
self.manager = CBLManager(directory: dir, options: nil, error: &error)
if manager == nil {
	NSLog("Cannot create Manager instance: %@", (error ?? ""))
}
```

```java+
// Not Manager constructor to specify the directory.
// Instead you should subclass JavaContext and override
// the getFilesDir method.
```

```android+
// Not Manager constructor to specify the directory.
// Instead you should subclass AndroidContext and override
// the getFilesDir method.
```

```c+
var options = new ManagerOptions();
options.ReadOnly = true;
Manager manager = new Manager(Directory.CreateDirectory(dbPath), options);
```

## Global logging settings

You can customize the global logging settings for Couchbase Lite via the `Manager` class. Log messages are tagged, allowing them to be logically grouped by activity. You can control whether individual tag groups are logged.

The available tags are:

<div class="tabs"></div>

```objective-c+
In Objective-C tag groups is disabled by default.

BLIP
BLIPVerbose
CBLDatabase
CBLJSONMatcher
CBLListener
CBLListenerVerbose
CBLModel
CBL_Router
CBL_Server
CBL_URLProtocol
CBLValidation
CBLRemoteRequest
CBLMultiStreamWriter
ChangeTracker
ChangeTrackerVerbose
JSONSchema
MYDynamicObject
Query
RemoteRequest
Sync
SyncVerbose
View
ViewVerbose
WS
```

```swift+
In Swift tag groups is disabled by default.

BLIP
BLIPVerbose
CBLDatabase
CBLJSONMatcher
CBLListener
CBLListenerVerbose
CBLModel
CBL_Router
CBL_Server
CBL_URLProtocol
CBLValidation
CBLRemoteRequest
CBLMultiStreamWriter
ChangeTracker
ChangeTrackerVerbose
JSONSchema
MYDynamicObject
Query
RemoteRequest
Sync
SyncVerbose
View
ViewVerbose
WS
```

```java+android+
In Java tag groups are enabled at level WARN by default.

Log tags

Log.TAG_BLOB_STORE //BlobStore
Log.TAG //CBLite
Log.TAG_CHANGE_TRACKER //ChangeTracker
Log.TAG_DATABASE //Database
Log.TAG_LISTENER //Listener
Log.TAG_MULTI_STREAM_WRITER //MultistreamWriter
Log.TAG_QUERY //Query
Log.TAG_REMOTE_REQUEST //RemoteRequest
Log.TAG_ROUTER //Router
Log.TAG_SYNC //Sync
Log.TAG_VIEW //View

Log levels

Log.VERBOSE
Log.DEBUG
Log.INFO
Log.WARN
Log.ERROR
```

```c+
Log tags

Log.Domains.Database
Log.Domains.Query
Log.Domains.View
Log.Domains.Router
Log.Domains.Sync
Log.Domains.ChangeTracker
Log.Domains.Validation
Log.Domains.Upgrade
Log.Domains.Listener
Log.Domains.Discovery
Log.Domains.TaskScheduling
Log.Domains.All

Log levels

Log.LogLevel.Verbose
Log.LogLevel.Debug
Log.LogLevel.Error
Log.LogLevel.Warning
Log.LogLevel.Information
```

The following code snippet enables logging for the **Sync** tag.

<div class="tabs"></div>

```objective-c+
CBLManager enableLogging: @"Sync"];
```

```swift+
CBLManager.enableLogging("Sync")
```

```java+android+
Manager.enableLogging("Sync", Log.VERBOSE);
```

```c+
Log.Domains.Sync.Level = Log.LogLevel.Verbose
```

## Concurrency Support

> **Note:** In Java all Couchbase Lite objects may be shared freely between threads. The rest of this section is irrelevant for Java programs, and applies only to Objective-C.

In Objective-C, a `Manager` instance and the object graph associated with it may only be accessed from the thread or dispatch queue that created the `Manager` instance. Concurrency is supported through explicit method calls.

### Running individual blocks in the background

You can use the `CBLManager` method `backgroundTellDatabaseNamed:to:` to perform any operation in the background. Be careful with this, though! Couchbase Lite objects are per-thread, and your block runs on a background thread, so:

- You can’t use any of the Couchbase Lite objects (databases, documents, models…) you were using on the main thread. Instead, you have to use the CBLDatabase object passed to the block, and the other objects reachable from it.
- You can’t save any of the Couchbase Lite objects in the block and then call them on the main thread. (For example, if in the block you allocated some CBLModels and assigned them to properties of application objects, bad stuff would happen if they got called later on by application code.)
- And of course, since the block is called on a background thread, any application or system APIs you call from it need to be thread-safe.

In general, it’s best to do only very limited things using this API, otherwise it becomes too easy to accidentally use main-thread Couchbase Lite objects in the block, or store background-thread Couchbase Lite objects in places where they’ll be called on the main thread.

Here’s an example that deletes a number of documents given an array of IDs:

<div class="tabs"></div>

```objective-c+
// "myDB" is the CBLDatabase object in use on the main thread.
CBLManager* mgr = myDB.manager;
NSString* name = myDB.name;
[mgr backgroundTellDatabaseNamed: name to: ^(CBLDatabase *bgdb) {
    // Inside this block we can't use myDB; instead use the instance given (bgdb)
    for (NSString* docID in docIDs) {
        [[bgdb documentWithID: docID] deleteDocument: nil];
}];
```

```swift+
// "myDB" is the CBLDatabase object in use on the main thread.
let mgr = myDB.manager
let name = myDB.name
mgr.backgroundTellDatabaseNamed(name, to: { (bgdb: CBLDatabase!) -> Void in
  // Inside this block we can't use myDB; instead use the instance given (bgdb)
  for docID in docIDs {
    bgdb.documentWithID(docID).deleteDocument(nil)
  }
})
```

```java+android+
No code example is currently available.
```

```c+
No code example is currently available.
```

### Running Couchbase Lite on a background thread

If you want to do lots of Couchbase Lite processing in the background in Objective-C, the best way to do it is to start your own background thread and use a new `Manager` instance on it.

<div class="tabs"></div>

```objective-c+
- (BOOL)application:(UIApplication *)application
        didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    CBLManager *manager = [CBLManager sharedInstance];
    NSError* error;
    self.database = [manager databaseNamed: @"db" error: &error];
    
    // We also want to run some Couchbase Lite operations on a background thread.
    // Since we can't use the regular CBLManager instance there, we copy it
    // and pass the copy to the background thread to use:
    CBLManager* bgMgr = [manager copy];
    [NSThread detachNewThreadSelector: @selector(runBackground:)
                             toTarget: self 
                           withObject: bgMgr];
    return YES;
}
- (void) runBackground: (CBLManager*)bgMgr {
    NSError* error;
    CBLDatabase* bgDB = [bgMgr databaseNamed: @"db" error: &error];
    // ... now use bgDB
}
```

```swift+
func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: NSDictionary?) -> Bool {
  let manager = CBLManager.sharedInstance()
  var error: NSError?
  let database = manager.databaseNamed("db", error: &error)
  let bgMgr = manager.copy()
  NSThread.detachNewThreadSelector("runBackground:", toTarget: self, withObject: bgMgr)
  return true
}
func runBackground(bgMgr: CBLManager) {
  var error: NSError?
  let bgDB = [bgMgr.databaseNamed("db", error: &error)]
}
```

```java+android+
No code example is currently available.
```

```c+
No code example is currently available.
```

If you don't plan to use Couchbase Lite on the main thread at all, the setup is even easier. Just have the background thread create a new instance of CBLManager from scratch and use that:

<div class="tabs"></div>

```objective-c+
- (BOOL)application:(UIApplication *)application
        didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // We're not going to use Couchbase Lite at all on the main thread;
    // instead we start a background thread to run it on:
    [NSThread detachNewThreadSelector: @selector(runBackground)
                             toTarget: self 
                           withObject: nil];
    return YES;
}
- (void) runBackground {
    // Create a CBLManager instance to use on this background thread:
    CBLManager* manager = [[CBLManager alloc] init];
    NSError* error;
    CBLDatabase* db = [manager databaseNamed: @"db" error: &error];
    // ... now use the database
}
```

```swift+
func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: NSDictionary?) -> Bool {
  // We're not going to use Couchbase Lite at all on the main thread;
  // instead we start a background thread to run it on:
  NSThread.detachNewThreadSelector("runBackground", toTarget: self, withObject: nil)
  return true
}
func runBackground {
  let manager = CBLManager.sharedInstance()
  var error: NSError?
  let db = [manager.databaseNamed("db", error: &error)]
  // ... now use the database 
}
```

```java+android+
No code example is currently available.
```

```c+
No code example is currently available.
```
