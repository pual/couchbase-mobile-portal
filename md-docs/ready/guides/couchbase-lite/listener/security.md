---
id: listener
title: Listener
permalink: ready/guides/couchbase-lite/native-api/attachment/index.html
---

Basic authentication is the recommended approach for protecting database access on the LAN. The listening peer must provide the username/password pair when instantiating the Listener.

<div class="tabs" />

```objective-c+
CBLListener* listener = [[CBLListener alloc] initWithManager:manager port:0];
listener.passwords = @{@"hello": @"pw123"};
[listener start:nil];  
```

```swift+
var listener: CBLListener = CBLListener(manager: manager, port: 0)
listener.passwords = ["hello": "pw123"]
listener.start(nil)
```

```java+
Credentials credentials = new Credentials("hello", "pw123");
LiteListener liteListener = new LiteListener(manager, 0, credentials);
liteListener.start();
```

```android+
Credentials credentials = new Credentials("hello", "pw123");
LiteListener liteListener = new LiteListener(manager, 0, credentials);
liteListener.start();
```

```c#
CouchbaseLiteTcpListener listener = new CouchbaseLiteTcpListener (manager, 0, CouchbaseLiteTcpOptions.AllowBasicAuth);
listener.SetPasswords(new Dictionary<string, string>() { { "hello", "pw123" } });
listener.Start ();
```

The peer that intends to run the replication must provide the same username/password http://username:password@hostname:port/dbname.

## SSL for Peer-to-

<div class="tabs" />

```objective-c+
if (![listener setAnonymousSSLIdentityWithLabel: @"MyApp SSL" error: &error])
    // handle error
```

```swift+
if !listener.setAnonymousSSLIdentityWithLabel("MyApp SSL", error: error) {
   // handle error }
```

```java+
No code example is currently available.
```

```c#
var path = System.IO.Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "unit_test.pfx");
var cert = X509Manager.GetPersistentCertificate("127.0.0.1", "123abc", path);
CouchbaseLiteTcpListener listener = new CouchbaseLiteTcpListener(manager, 0, CouchbaseLiteTcpOptions.UseTLS, cert);
```

The Listener is now serving SSL using an automatically generated identity.

### Wait, Is This Secure?

Yes and no. It encrypts the connection, which is unquestionably much better than not using SSL. But unlike the usual SSL-in-a-browser approach you're used to, it doesn't identify the server/listener to the client. The client has to take the cert on faith the first time it connects.