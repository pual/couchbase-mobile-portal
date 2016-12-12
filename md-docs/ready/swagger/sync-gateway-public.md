---
id: sync-gateway-public
title: Sync Gateway Public REST API
permalink: ready/references/sync-gateway/rest-api/index.html
---

The API explorer below groups all the endpoints by functionality. You can click on a label to expand the list of endpoints.

You can also send a request to each endpoint against an instance of Sync Gateway. To do so, you must enable CORS with the following in the configuration file. Refer to the Sync Gateway [installation guide](/documentation/mobile/current/installation/sync-gateway/index.html) for more information on starting a new instance of Sync Gateway.

```javascript
{
	...
	"CORS": {
		"Origin":["*"],
		"LoginOrigin":["*"],
		"Headers":["Content-Type"],
		"MaxAge": 1728000
	},
	...
}
```

### API Explorer

{% include swagger.html name="sync-gateway-public" %}