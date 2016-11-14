---
id: other-providers
title: Other Providers
permalink: ready/guides/authentication/static-providers/index.html
---

### Facebook authentication

The Couchbase Sync Gateway allows users to authenticate using a Facebook account. Your application is responsible for generating a Facebook token; this generally needs to be done by running the Facebook login flow inside a web-view and capturing the generated token. Facebook provides SDKs for both iOS and Android to assist in doing this, and on iOS the Accounts framework can be used to retrieve an active login token (with the user's consent.)

You'll also need to ensure you've asked for the `email` permission in your Facebook app configuration.

Sync Gateway supports [Facebook Login](https://developers.facebook.com/docs/facebook-login/), which allows users to log in by using their Facebook account. To enable it, add a top-level `facebook` property to your server configuration file. For example:

```javascript
"facebook" : {
   "register" : true
}
```

Clients log in by sending a POST request to `/dbname/_facebook`, with a JSON body that contains the following objects:

- `access_token`: Access token returned by Facebook
- `email`: Email address of the user
- `remote_url`: Sync Gateway replication endpoint URL

Just as with a `_session` login, the response sets a session cookie.

The server will send the token to Facebook's servers to validate it, and if successful will generate a login session and return a session cookie to the app. The session will eventually expire, so you should be prepared to detect an authentication failure (HTTP 401 status) reported by the replication, and get a new token from Facebook.


## Facebook Account Registration

If the `register` property of the Facebook configuration is true, then clients can implicitly register new user accounts by authenticating through Facebook. If Sync Gateway verifies the client's assertion, but no existing user account has that email address, it creates a new user account for that email and returns a session cookie.

The user name for the new account is the same as the authenticated email address and has a random password. There is no way to retrieve the password, so in the future the client must continue to log in by using Facebook, unless the app server replaces the password with one known to the client.