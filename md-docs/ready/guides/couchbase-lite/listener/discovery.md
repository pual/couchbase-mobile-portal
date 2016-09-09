---
id: listener
title: Listener
permalink: ready/guides/couchbase-lite/native-api/attachment/index.html
---

Once you have set up the Listener as an endpoint for other peers to replicate to or from, you can use different discovery methods to browse for peers and subscribe to those of interest.

This guide covers two recommended ways to discover peers:

- Bonjour: The implementation name differs depending on the platform (Bonjour for iOS/OS X, NSD for Android, JmDNS for Java and ZeroConf for .NET). All implementations follow the official spec and can be used to discover peers across different platforms.
- QR code

## Bonjour

The first step to using Bonjour for peer discovery is to advertize a service with the following properties:

- Type: Bonjour can be used by many other devices on the LAN (printers, scanners, other apps etc). The service type is a way to interact only with peers whose service type is the same.
- Name: A string to serve as identifier for other peers. It should be unique for each peer.
- Port: The port number the Listener is running on.
- Metadata: Optional data that will be sent in the advertizment packets (the size limit is around 15KB).

> **Note:** Bonjour browsers are useful to monitor devices broadcasting a particular service on the LAN ([OS X Bonjour browser](http://www.macupdate.com/app/mac/13388/bonjour-browser), [iOS app](https://itunes.apple.com/gb/app/discovery-bonjour-browser/id305441017), [Windows browser](http://hobbyistsoftware.com/bonjourbrowser))

Given a service type, you can use an API to browse for all services with that service type. Various callback methods are invoked as peers on the network go online and offline.

The service information that can be retrieved when browsing for clients (step 1) doesn't include the IP address for performance reasons. To limit the amount of network traffic going on, the system will only look up the IP address of a peer once there is an intent (step 2) to connect to it:

![](img/docs-peer-discover-diagram.png)

Once the IP is resolved in step 3, the replication with that peer can be started in step 4. The following sections cover the different APIs for the **advertiser** (device A) and **subscriber** (device B).

### Advertiser

Start a listener with the following.

<div class="tabs" />

```objective-c+
[listener setBonjourName:@"chef123" type:@"_myapp._tcp"];
```

```swift+
listener.setBonjourName("chef123", type: "_myapp._tcp")
```

```java+
JmDNS jmdns = new JmDNS();
  
ServiceInfo serviceInfo = ServiceInfo.create("_myapp._tcp", "chef123", 55000, "A service description");
jmdns.registerService(serviceInfo);
```

```android+
// Create the NsdServiceInfo object, and populate it.
NsdServiceInfo serviceInfo = new NsdServiceInfo();
  
serviceInfo.setServiceName("chef123");
serviceInfo.setServiceType("_myapp._tcp);
serviceInfo.setPort(55000);
nsdManager.registerService(serviceInfo, NsdManager.PROTOCOL_DNS_SD, registrationListener);
// registrationListener is an instance of NsdManager.RegistrationListener
```

```c#

```

### Subscriber

To browse for peers on the network, each implementation has an asynchronous API to get notified as peers go online and offline from the network. First, you must implement the protocol or interface before starting the network discovery:

- Bonjour: Implement the NSNetServiceBrowserDelegate protocol
- NSD: Create a new instance of the NsdManager.DiscoveryListener class
- JmDNS: Implement the ServiceListener interface

After setting the listener or delegate, create a new instance of the discovery object.

<div class="tabs" />

```objective-c+
NSNetServiceBrowser* browser = [NSNetServiceBrowser new];
browser.includesPeerToPeer = YES;
browser.delegate = self;
[browser searchForServicesOfType:@"_myapp._tcp" inDomain:@"local."];
```

```swift+
browser = NSNetServiceBrowser.new()
browser.includesPeerToPeer = true
browser.delegate = self
browser.searchForServiceOfType("_myapp._tcp", inDomain: "local.")
```

```java+
jmdns.addServiceListener("_myapp._tcp", new DiscoveryListener(database, jmdns, serviceName));
```

```android+
mNsdManager.discoverServices("_myapp._tcp", NsdManager.PROTOCOL_DNS_SD, mDiscoveryListener);
// mDiscoveryListener is an instance of NsdManager.DiscoveryListener
```

### Hostname resolution

The hostname resolution can be done in the listener/protocol you have implemented previously.

<div class="tabs" />

```objective-c+
- (void) netServiceBrowser:(NSNetServiceBrowser *)browser didFindService:(NSNetService *)service moreComing:(BOOL)moreComing {
  // Start async resolve, to find service's hostname
  service.delegate = self;
  [service resolveWithTimeout:5];
}
```

```swift+
public func netServiceBrowser(browser: NSNetServiceBrowser, didFindService service:   NSNetService, moreComing: Bool) {
  // Start async resolve, to find service's hostname
  service.delegate = self
  service.resolveWithTimeout(5.0)
}
```

```java+
@Override
public void serviceAdded(ServiceEvent event) {
  jmdns.requestServiceInfo(event.getType(), event.getName(), 10);
}
```

```android+
@Override
public void onServiceFound(NsdServiceInfo service) {
  nsdManager.resolveService(serviceInfo, resolveListener);
  // Instance of NsdManager.ResolveListener
}
```

```c#

```

When the IP is received, the corresponding method will get called at which point the replication can be started.

<div class="tabs" />

```objective-c+
// NSNetService delegate callback
- (void) netServiceDidResolveAddress:(NSNetService *)service {
    // Construct the remote DB URL
    NSURLComponents* components = [[NSURLComponents alloc] init];
    components.scheme = @"http"; // Or "https" uf peer uses SSL
    components.host = service.hostName;
    components.port = [NSNumber numberWithInt:service.port];
    components.path = [NSString stringWithFormat:@"/@%", remoteDatabaseName];
    NSURL* url = [components URL];
      
    // Start replications
    CBLReplication* push = [database createPushReplication:url];
    CBLReplication* pull = [database createPullReplication:url];
    [push start];
    [pull start];
}
```

```swift+
// NSNetService delegate callback
func netServiceDidResolveAddress(service: NSNetService) {
    // Construct the remote DB URL
    let components = NSURLComponents()
    components.scheme = "http" // Or "https" if peer uses SSL
    components.host = service.hostName!
    components.port = service.port
    components.path = "/" + remoteDatabaseName
    let url = components.URL!
    
    // Start replications
    let push = database?.createPushReplication(url)!
    let pull = database?.createPullReplication(url)!
    push?.start()
    pull?.start()
}
```

```java+
@Override
public void serviceResolved(ServiceEvent event) {
  System.out.println("RESOLVED");
  String[] serviceUrls = event.getInfo().getURLs();
  try {
    URL url = new URL(serviceUrls[0]);
    Replication pullReplication = database.createPullReplication(url);
    pullReplication.setContinuous(true);
    pullReplication.start();
    Replication pushReplication = database.createPushReplication(url);
    pushReplication.setContinuous(true);
    pushReplication.start();
  } catch (IOException e){
    throw new RuntimeException(e);
  }
}
```

```android+
@Override
public void onServiceResolved(NsdServiceInfo serviceInfo) {
    Log.e(Application.TAG, "Resolve Succeeded. " + serviceInfo);
    String remoteStringURL = String.format("http:/%s:%d/%s",
            serviceInfo.getHost(),
            serviceInfo.getPort(),
            StorageManager.databaseName);
    URL remoteURL = null;
    try {
        remoteURL = new URL(remoteStringURL);
    } catch (MalformedURLException e) {
        e.printStackTrace();
    }
    Database database = StorageManager.getInstance().database;
    Replication push = database.createPushReplication(remoteURL);
    Replication pull = database.createPullReplication(remoteURL);
    push.setContinuous(true);
    pull.setContinuous(true);
    push.start();
    pull.start();
}
```

```c#

```

### Resources

Useful resources to work with mDNS include:

- **Bonjour for iOS and Mac applications:** The Couchbase Lite SDK exposes part of the Bonjour API for an easier integration. The official documentation for iOS and Mac applications can be found in the [NSNetService Programming Guide](https://developer.apple.com/library/mac/documentation/Networking/Conceptual/NSNetServiceProgGuide/Introduction.html).
- **NSD for Android applications:** The de facto framework for Android is called Network Service Discovery (NSD) and is compatible with Bonjour since Android 4.1. The official guide can be found in the [Android NSD guide](https://developer.android.com/training/connect-devices-wirelessly/nsd.html).
- **JmDNS:** Implementation in Java that can be used in Android and Java applications ([official repository](https://github.com/jmdns/jmdns)).

## QR code

### PhotoDrop

[PhotoDrop](https://github.com/couchbaselabs/photo-drop) is a P2P sharing app similar to the iOS AirDrop feature that you can use to send photos across devices. The source code is available for iOS and Android and it uses a QR code for peer discovery. The QR code is used for advertising an adhoc endpoint URL that a sender can scan and send photos to.





