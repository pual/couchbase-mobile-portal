---
id: basic-authentication
title: Basic Authentication
permalink: ready/guides/authentication/basic-authentication/index.html
---

By default, Sync Gateway does not enable authentication. This is to make it easier to get up and running with synchronization. You can enable authentication with the following properties in the configuration file:

```javascript
{
  "databases": {
    "mydatabase": {
      "users": {
        "GUEST": {"disabled": true},
        "john": {"password": "pass"}
      }
    }
  }
}
```

In this configuration file, you are creating a user where the username is `john` and password is `pass`. Then, the user name and password to authenticate a user can be specified in two ways covered below.

### Authenticator

You can provide a user name and password to the basic authenticator class method of the Authenticator class. Under the hood, the replicator will send the credentials in the first request to retrieve a SyncGatewaySession cookie and use it for all subsequent requests during the replication. This is the recommended way of using Basic authentication.

<div class="tabs"></div>

```objective-c+
NSURL* url = [NSURL URLWithString: @"https://example.com/mydatabase/"];
CBLReplication *push = [database createPushReplication: url];
CBLReplication *pull = [database createPullReplication: url];
push.continuous = pull.continuous = YES;
id<CBLAuthenticator> auth;
auth = [CBLAuthenticator basicAuthenticatorWithName: @"john"
                                           password: @"pass"];
push.authenticator = pull.authenticator = auth;
```

```swift+
let url = NSURL(string: "https://example.com/mydatabase/")
let push = database.createPushReplication(url)
let pull = database.createPullReplication(url)
push.continuous = true
pull.continuous = true
var auth: CBLAuthenticatorProtocol?
auth = CBLAuthenticator.basicAuthenticatorWithName("john", password: "pass")
push.authenticator = auth
pull.authenticator = auth
```

```java+android+
URL url = new URL("https://example.com/mydatabase/");
Replication push = database.createPushReplication(url);
Replication pull = database.createPullReplication(url);
pull.setContinuous(true);
push.setContinuous(true);
Authenticator auth = new BasicAuthenticator("john", "pass");
push.setAuthenticator(auth);
pull.setAuthenticator(auth);
```

```c+
var url = new Uri("https://example.com/mydatabase/");
var push = database.CreatePushReplication(url);
var pull = database.CreatePullReplication(url);
var auth = AuthenticatorFactory.CreateBasicAuthenticator("john", "pass");
push.Authenticator = auth;
pull.Authenticator = auth;
push.Continuous = true;
pull.Continuous = true;
```

### Replication URL

With this method, you can embed the username and password in the replication URL with the following syntax `https://username:password@example.com/database`. The downside of this approach is that the user credentials are sent in every request (as opposed to a cookie session).

> On iOS and Mac OS you can also take advantage of the URL loading system's credential store and Keychain. Credentials registered this way will automatically be applied to requests made by the replicator. You can easily do this by creating an `NSURLCredential` object and assigning it to the replication's `credential` property. Or if you use the Keychain APIs to persistently store a credential in the application's keychain, it will always be available to the replicator. This is the most secure way to store a credential, since the keychain file is encrypted.

## App Server for User Registration

In the previous section, the user credentials (**john/pass**) were hardcoded in the configuration file. If you intend to have a sign up screen on your application then users must be created dynamically. For this reason, Sync Gateway users can also be created on the Admin REST API (port 4985 by default). This port is not accessible from the clients directly however.

To allow users to sign up, it is recommended to have an app server sitting alongside Sync Gateway that performs the user validation, creates a new user on the Sync Gateway admin port and then returns the response to the application.

The following example will use the Sync Gateway Swagger spec and Node.js client to perform this operation. However, you may use the server-side language of your choice since it's a simple HTTP request.

Install the following dependencies.

```bash
npm install swagger-client express body-parser
```

Open a new file called `app.js` with the following.

```javascript
var Swagger = require('swagger-client')
  , express = require('express')
  , bodyParser = require('body-parser');

var client = new Swagger({
  url: 'http://developer.couchbase.com/mobile/swagger/sync-gateway-admin/spec.json',
  usePromise: true
})
  .then(function (res) {
    client = res;
  });

var app = express();
app.use(bodyParser.json());

app.post('/signup', function (req, res) {
  client.user.post_db_user({db: 'todo', body: {name: req.body.name, password: req.body.password}})
    .then(function (userRes) {
      res.status(userRes);
      res.send(userRes);
    })
    .catch(function (err) {
      res.status(err.status);
      res.send(err);
    });
});

app.listen(3000, function () {
  console.log('App listening at http://localhost:3000');
});
```

Here, you're using the `post_db_user` method to create the user and the Node.js express module to create a route and return the response from Sync Gateway on the admin port. Start the app server from the command line.

```bash
node app.js
```

Create a user using curl.

```bash
curl -H 'Content-Type: application/json' -X POST 'http://localhost:3000/signup' \
     -d '{"name": "john", "password": "pass"}'
```

You can now send this request from your mobile and web clients and display a message according to the response status:

- `201`: The user was successfully created
- `409`: A user with this name already exists