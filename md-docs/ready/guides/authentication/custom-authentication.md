---
id: custom-authentication
title: Custom Authentication
permalink: ready/guides/authentication/custom-authentication/index.html
---

It's possible for an application server associated with a remote Couchbase Sync Gateway to provide its own custom form of authentication. Generally this will involve a particular URL that the app needs to post some form of credentials to; the App Server will verify those, then tell the Sync Gateway to create a new session for the corresponding user, and return session credentials in its response to the client app. The following diagram shows an example architecture to support Google SignIn in a Couchbase Mobile application, the client sends an access token to the App Server where a server side validation is done with the Google API and a corresponding Sync Gateway user is then created if it's the first time the user logs in. The last request creates a session:

![](img/custom-auth-flow.png)

The request on the Admin REST API to create a new session given a user name is the following:

```bash
$ curl -vX POST -H 'Content-Type: application/json' \
        -d '{"name": "myusername", "ttl": 180}' \
        http://localhost:4985/sync_gateway/_session
// Response message body
{
  "session_id": "904ac010862f37c8dd99015a33ab5a3565fd8447",
  "expires": "2015-09-23T17:27:17.555065803+01:00",
  "cookie_name": "SyncGatewaySession"
}
```

The HTTP response body contains the credentials of the session. It is recommended to return the session details to the client application in the same form and to use the setCookie method on the replication object with the parameters:

- **name** corresponds to the `cookie_name`
- **value** corresponds to the `session_id`
- **path** is the hostname of the Sync Gateway
- **expirationDate** corresponds to the `expires`
- **secure** Whether the cookie should only be sent using a secure protocol (e.g. HTTPS)
- **httpOnly** Whether the cookie should only be used when transmitting HTTP, or HTTPS, requests thus restricting 
access from 
other, non-HTTP APIs

Given the HTTP response above, the setCookie is used as follows:

<div class="tabs"></div>

```objective-c+
NSDictionary* session = @{
    @"session_id": @"904ac010862f37c8dd99015a33ab5a3565fd8447",
    @"expires": @"2015-09-23T17:27:17.555065803+01:00",
    @"cookie_name": @"SyncGatewaySession"
    };
  
NSString* dateString = session[@"expires"];
NSDateFormatter* dateFormatter = [[NSDateFormatter alloc] init];
dateFormatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ss.SSSZ";
NSDate* date = [dateFormatter dateFromString:dateString];
  
CBLReplication* pull = [database createPullReplication:syncGatewayURL];
[pull setCookieNamed:session[@"cookie_name"] withValue:session[@"session_id"] path:@"/" expirationDate:date secure:NO];
[pull start];
```

```swift+
let session: Dictionary<String, String> = [
  "session_id": "904ac010862f37c8dd99015a33ab5a3565fd8447",
  "expires": "2015-09-23T17:27:17.555065803+01:00",
  "cookie_name": "SyncGatewaySession"
]
  
let dateString = session["expires"]!
let dateFormatter = NSDateFormatter()
dateFormatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss.SSSZ"
let date = dateFormatter.dateFromString(dateString)!
  
let pull = database.createPullReplication(syncGatewayURL)
pull?.setCookieName(session["cookie_name"]!, withValue: session["session_id"]!, path: "/", expirationDate: date, secure: false)
pull?.start()
```

```java+android+
Map<String, String> session = new HashMap<String, String>();
session.put("session_id", "904ac010862f37c8dd99015a33ab5a3565fd8447");
session.put("expires", "2015-09-23T17:27:17.555065803+01:00");
session.put("cookie_name", "SyncGatewaySession");
  
DateFormat formatter = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSZ");
Date date = null;
try {
    date = formatter.parse(session.get("expires"));
} catch (ParseException e) {
    e.printStackTrace();
}
  
Replication pull = database.createPullReplication(syncGatewayURL);
pull.setCookie(session.get("cookie_name"), session.get("session_id"), "/", date, false, false);
pull.start();
```

```c+
No code example is currently available.
```

## Recap

An app server can create a session for a user by sending a POST request to `/dbname/_session`. This works only on the admin port.

The request body is a JSON document with the following properties:

- `name`: User name

- `ttl`: Number of seconds until the session expires. This is an optional parameter. If ttl is not provided, the default value of 24 hours is used.

The response body is a JSON document that contains the following properties:

- `session_id`: Session string
  - `cookie_name`: Name of the cookie the client should send
  - `expires`: Date and time that the session expires

This allows the app server to optionally do its own authentication using the following control flow:

1. Client sends credentials to your app server.

2. App server authenticates the credentials however it wants (LDAP, OAuth, and so on).

3. App server sends a POST request with the user name to the Sync Gateway Admin REST API server /dbname/_session endpoint.

4. If the request fails with a 404 status, there is no Sync Gateway user account with that name. The app server can then create one (also using the Admin REST API) and repeat the _session request.

5. The app server adds a Set-Cookie: HTTP header to its response to the client, using the session cookie name and value received from the gateway.

Subsequent client requests to the gateway will now include the session in a cookie, which the gateway will recognize. For the cookie to be recognized, your site must be configured so that your app's API and the gateway appear on the same public host name and port.

## Session Expiration

By default, a session created on Sync Gateway lasts 24 hours. If you create sessions by sending a POST request to `/db/_session`, you can set a custom value that overrides the system default. 

## Authentication for Web Apps

A common use case is to sync with HTML 5 apps that interact directly with the Sync Gateway REST API or use PouchDB to initiate replications. The web app is usually served on a different origin (domain) than the Sync Gateway domain so you will first need to enable CORS in the config file.

By default, browsers won't send the credentials (cookies) in a CORS request but there is an option called `withCredentials` to enable it (see the [Mozilla documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS#Requests_with_credentials)).

The following configuration enables CORS and creates a user (`john/pass`).

```javascript
{
  "log": ["HTTP+"],
  "CORS": {
    "Origin": ["http://localhost:9000"],
    "LoginOrigin": ["http://localhost:9000"],
    "Headers": ["Content-Type"],
    "MaxAge": 17280000
  },
  "databases": {
    "todo": {
      "server": "walrus:",
      "users": {
        "john": {"password": "pass", "admin_channels": ["*"]}
      }
    }
  }
}
```

For authentication to work, you must set the **Origin** and **LoginOrigin** to the origin of the web app (the hostname of the web server). You can read more about CORS support on the [Sync Gateway configuration guide](/documentation/mobile/current/develop/guides/sync-gateway/config-properties/index.html#cors-configuration). Then, you can send a POST /{db}/_session request passing the name and password of the user to authenticate.

Below is an example using the Swagger client as the JavaScript library to consume the Sync Gateway REST API.

```javascript
var client = new SwaggerClient({
  url: 'http://developer.couchbase.com/mobile/swagger/sync-gateway-public/spec.json',
  usePromise: true,
  enableCookies: true
})
  .then(function (client) {
    client.session.post_db_session({db: 'todo', name: 'john', password: 'pass'})
      .then(function (res) {
        console.log(res);
        if (res.status == 200) {
          return client.query.get_db_all_docs({db: 'todo', include_docs: true});
        }
      })
      .then(function (res) {
        console.log(res.obj);
      })
      .catch(function (err) {
        console.log(err);
      });
  });
```

> **Note:** Keep in mind that in this example the Swagger client is pointing to the spec hosted on developer.couchbase.com. We often publish changes to those specs for documentation purposes; if it's a breaking change then it will modify the request and parameter names in the Swagger client and break your code. You can refer to the [changelog of the specs](https://github.com/couchbaselabs/couchbase-mobile-portal/blob/master/swagger/CHANGELOG.md) to find the list of methods and parameters that changed. In production, we highly encourage you to download the spec as a `.json` file and pass it to the Swagger client using the `{spec: <spec>}` option.

The same can be done in vanilla JavaScript with the XMLHttpRequest API of course.

```javascript
var SYNC_GATEWAY_URL = 'http://localhost:4984/todo';
var login = new XMLHttpRequest();
login.open('POST', SYNC_GATEWAY_URL + '/_session', true);
login.setRequestHeader('Content-Type', 'application/json');
login.onreadystatechange = function() {
  if (login.readyState == 4 && login.status == 200) {
    
    var allDocs = new XMLHttpRequest();
    allDocs.open('GET', SYNC_GATEWAY_URL + '/_all_docs', true);
    allDocs.setRequestHeader('Content-Type', 'application/json');
    
    allDocs.onreadystatechange = function() {
      if (allDocs.readyState == 4 && allDocs.status == 200) {
        console.log(allDocs.response);
      }
    };
    
    allDocs.withCredentials = true;
    allDocs.send();
    
  }
};
login.send(JSON.stringify({"name": "john", "password": "pass"}));
```

Note that you must set the `withCredentials` option on all requests to Sync Gateway that require authentication.