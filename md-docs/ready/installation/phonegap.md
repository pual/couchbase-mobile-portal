---
id: phonegap
title: PhoneGap
permalink: ready/installation/phonegap/index.html
---

You will install Couchbase Lite using the PhoneGap CLI module.

```bash
npm install -g phonegap
phonegap create UntitledApp
cd UntitledApp/
phonegap local plugin add https://github.com/couchbaselabs/Couchbase-Lite-PhoneGap-Plugin.git
```

On iOS, you must have the **ios-sim** module installed globally to start the emulator from the command line.

```bash
npm install -g ios-sim
phonegap run ios
```

## Getting Started

Open **www/js/index.js** and add the following in the `onDeviceReady` lifecycle method.

```javascript
app.receivedEvent('deviceready');
if (window.cblite) {
  window.cblite.getURL(function (err, url) {
    if (err) {
      app.logMessage("error launching Couchbase Lite: " + err)
    } else {
      app.logMessage("Couchbase Lite running at " + url);
    }
  });
} else {
  app.logMessage("error, Couchbase Lite plugin not found.")
}
```

Below the `onDeviceReady` method, add a new method called `logMessage`.

```javascript
logMessage: function(message) {
  var p = document.createElement("p");
  p.innerHTML = message;
  document.body.getElementsByClassName('app')[0].appendChild(p);
  console.log(message);
},
```

Build & run.

```bash
phonegap run ios
phonegap run android
```

The Couchbase Lite endpoint is displayed on the screen and you can start making RESTful queries to it using the HTTP library of your choice.

![](img/phonegap-ios-android.png)

## Listener API

The Couchbase Lite Listener exposes the same functionality as the native SDKs through a common RESTful API. You can perform the same operations on the database by sending HTTP requests to it.

The Swagger JS client allows us to leverage the Couchbase Lite REST API Swagger spec in hybrid mobile frameworks such as PhoneGap. To install the Swagger JS library in your project do the following:

- [Download the Swagger JS client](https://raw.githubusercontent.com/swagger-api/swagger-js/master/browser/swagger-client.min.js) to a new file **www/js/swagger-client.min.js**.
- [Download the Couchbase Lite Swagger spec](http://developer.couchbase.com/mobile/swagger/couchbase-lite/spec.json) to a new file **www/js/spec.js**. Your IDE might show an error because you've copied a JSON object into a JavaScript file but don't worry, prepend the following to set the spec on the `window` object.

  ```javascript
  window.spec = {
    "swagger": "2.0",
    "info": {
      "title": "Couchbase Lite",
      ...
    }
  }
  ```
  
  Here, you're embedding the API spec as part of the application so that the JS library can also work offline.

Reference both files in **www/index.html** before `app.initialize()` is executed.

```html
<script type="text/javascript" src="js/swagger-client.min.js"></script>
<script type="text/javascript" src="js/spec.js"></script>
<script type="text/javascript">
    app.initialize();
</script>
```

Then, open **www/js/index.js** and add the following in the `onDeviceReady` lifecycle method.

```javascript
var client = new SwaggerClient({
  spec: window.spec,
  usePromise: true,
})
  .then(function (client) {
    client.setHost(url.split('/')[2]);
    client.server.get_all_dbs()
      .then(function (res) {
        var dbs = res.obj;
        if (dbs.indexOf('todo') == -1) {
          return client.database.put_db({db: 'todo'});
        }
        return client.database.get_db({db: 'todo'});
      })
      .then(function (res) {
        return client.document.post({db: 'todo', body: {task: 'Groceries'}});
      })
      .then(function (res) {
        return client.query.get_db_all_docs({db: 'todo'});
      })
      .then(function (res) {
        alert(res.obj.rows.length + ' document(s) in the database');
      })
      .catch(function (err) {
        console.log(err)
      });
  });
```

First you set the host of the Swagger JS client to the url provided in the callback (it is of the form `lite.couchbase.` on iOS and `localhost:5984` on Android). The chain of promises creates the database, inserts a document and displays the total number of documents in an alert window using the `/{db}/_all_docs` endpoint.

Build and run.

```bash
phonegap run ios
phonegap run android
```

You should see the alert dialog displaying the number of documents.

<img src="img/swagger-phonegap.png" class="portrait" />