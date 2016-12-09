---
id: sync-gateway-public
title: Sync Gateway Public REST API
permalink: ready/references/sync-gateway/rest-api/index.html
---

The API explorer below groups all the endpoints by functionality. You can click on a label to expand the list of endpoints. For each endpoint, the **Try it out** button can be used to generate the request using curl.

### API Explorer

{% include swagger.html name="sync-gateway-public" %}

### Try it out

You can also try out each endpoint against an instance of Sync Gateway running on `http://localhost:4984`. To do so, you must first enable CORS on Sync Gateway. [Download Sync Gateway](http://www.couchbase.com/nosql-databases/downloads#couchbase-mobile) and start it with the following configuration.

```javascript
{
	"log": ["*"],
	"CORS": {
		"Origin":["*"],
		"LoginOrigin":["*"],
		"Headers":["Content-Type"],
		"MaxAge": 1728000
	},
	"databases": {
		"db": {
			"server": "walrus:",
			"users": { "GUEST": { "disabled": false, "admin_channels": ["*"] } }
		}
	}
}
```