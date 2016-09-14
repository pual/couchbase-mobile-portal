---
id: performance-testing
title: Performance Testing
permalink: ready/training/test/performance-testing/index.html
---

In this lesson you'll learn how to perform performance tests on your Couchbase Mobile application. You'll test the performance of Sync Gateway.

## Testing environment

// Run VM instructions

## Running a performance test

To run a performance test you must first generate some load. One way of increasing the load is to insert random documents through the REST API. The bash script below inserts 1000 documents.

```bash
#!/usr/bin/env bash

COUNTER=0

while [ $COUNTER -lt 1000 ]; do
		uuid=$(uuidgen)
		curl -X POST -H 'Content-Type: application/json' \
					-d '{"type":"task-list", "name": "'"${uuid}"'", "owner": "user1", "_id": "user1.'"${uuid}"'"}' \
					'http://user1:pass@localhost:4984/todo/' -w "\n"
		let COUNTER=COUNTER+1
done
```

To measure performance we generally rely on a metric before and after the test was run. In this case you will use the Sync Gateway response time for the GET /{db}/_changes request.

### Try it out

1. Start Sync Gateway with the config file in the accompanying project. Then measure the response time of the changes feed using curl.

    ```bash
    curl -o /dev/null -s -w %{time_total}\\n 'http://user1:pass@localhost:4984/todo/_changes'
    ```

    The response should be approximately 1ms.

2. Run the load testing script.

    ```bash
    bash docs.sh
    ```

3. Query the Sync Gateway changes feed again.

    ```bash
    curl -o /dev/null -s -w %{time_total}\\n 'http://user1:pass@localhost:4984/todo/_changes'
    ```

    The response should be approximately 60 ms.

## Tuning Sync Gateway

The performance observed is not optimal. Indeed, the Sync Gateway configuration file sets the `maxFileDescriptors` property to `10`. This is a good example of a parameter that can impact the performance observed on the Sync Gateway instance. You will increase that value in the config file and also at the OS Level.

### Try it out

1. Modify the value for `maxFileDescriptors` to `250000` in **sync-gateway-config.json**.
2. Run the load testing script.

    ```bash
    bash docs.sh
    ```

3. Query the Sync Gateway changes feed and measure the HTTP response time.

    ```bash
    curl -o /dev/null -s -w %{time_total}\\n 'http://user1:pass@localhost:4984/todo/_changes'
    ```

    _Observation:_ makes no difference.

## Conclusion

Well done! You've completed this lesson on functional testing. In the next lesson you'll learn how to deploy Sync Gateway and Couchbase Server. Feel free to share your feedback, findings or ask any questions on the forums.