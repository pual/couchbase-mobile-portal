---
id: functional-testing
title: Functional Testing
permalink: ready/training/functional-testing/index.html
---

In this lesson you'll learn how to perform functional tests on your Couchbase Mobile application. You'll test the Sync Function to make sure it conforms to the business rules of the application.

// Download project

## Introduction

In this section you will learn how to categorize the different types of functional tests. The Sync Function is the application's core security component so it's important to test it extensively.

When a document is processed in the Sync Function, there are usually 4 steps to determine if the operation will succeed.

![](img/image15.png)

In the project folder you will find the sync function. The table below translates each line into what it means from an access control standpoint.

|Scenario|Document Type|Type of Test|
|:-------|:------------|:---------------|
|The type property must be defined and immutable|*|Validation|
||||
|Only users with **admin** role can write|moderator|Write|
|The _id property must be follow "moderator.{username}"|moderator|Validation|
|The username property is immutable|moderator|Validation|
|Document is added to user channel|moderator|Routing|
|User has access to moderators channels|moderator|Read|
||||
|Only owner can create lists for themselves|task-list|Write|
|Moderators can create lists for other users|task-list|Write|
|Name and owner properties are required|task-list|Validation|
|The _id must be "{username}.uuid"|task-list|Validation|
|Owner field is immutable|task-list|Validation|
|Add doc to "task-list.{doc._id}" channel|task-list|Validation|
||||
||task||
|`taskList.id`, `taskList.owner` and `task` are required|task|Validation|
|The doc.username user is granted read access to the "task-list.{doc.taskList.id}" channel|task-list:user|Routing|

### Try it out

1. Download Sync Gateway
2. Start it from the command with the configuration file that was provided.

    ```bash
    ~/path/to/sync_gateway sync-gateway-config.json
    ```

## Testing Validation Rules

The first type of test is to validate that the document meets the schema requirements.

|Scenario|Document Type|Type of Test|
|:-------|:------------|:---------------|
|`taskList.id`, `taskList.owner` and `task` are required|task|Validation|

### Try it out

1. Save a new task document with an invalid schema.

    ```bash
    curl -H 'Content-type: application/json' \
        -X POST 'http://localhost:4985/todo/' \
        -d '{"type": "task", "taskList": {"owner": "user2"}}'

    {"error":"Forbidden","reason":"taskList.id is not provided."}

    curl -H 'Content-type: application/json' \
        -X POST 'http://localhost:4985/todo/' \
        -d '{"type": "task", "taskList": {"id": 123}}'

    {"error":"Forbidden","reason":"task.owner is not provided."}

    curl -H 'Content-type: application/json' \
        -X POST 'http://localhost:4985/todo/' \
        -d '{"type": "task", "taskList": {"owner": "user2", "id": 123}}'

    {"error":"Forbidden","reason":"task is not provided."}
    ```

    The operation is rejected in all 3 requests with a **403 Forbidden** error because the schema is not valid. The test has passed.

## Testing Write Permissions

### User write permissions

In this section you will test that normal users can't create a list for another user.

|Scenario|Document Type|Type of Test|
|:-------|:------------|:---------------|
|Only owner can create lists for themselves|task-list|Write|

#### Try it out

1. Save a new list for another user.

    ```bash
    curl -H 'Content-Type: application/json' \
        -X POST 'http://user1:pass@localhost:4984/todo/' \
        -d '{"name": "Groceries", "type": "task-list", "owner": "user2"}'

    {"error":"Forbidden","reason":"missing role"}
    ```

    The operation is rejected with a **403 Forbidden** error because user1 is attempting to create a list for user2. The test has passed.

### Moderator write permissions

In this section you will test that moderators have the right to create lists for other users.

|Scenario|Document Type|Type of Test|
|:-------|:------------|:---------------|
|Moderators can create lists for other users|task-list|Write|

#### Try it out

1. Grant user1 with the **moderator** role.

    ```bash
    curl -H 'Content-Type: application/json' \
        -X PUT 'http://localhost:4985/todo/_user/user1' \
        -d '{"admin_roles": ["moderator"]}'
    ```

2. Create a new list for user2.

    ```bash
    curl -H 'Content-Type: application/json' \
        -X POST 'http://user1:pass@localhost:4984/todo/' \
        -d '{"_id": "user2.123", "name": "Groceries", "type": "task-list", "owner": "user2"}'

    {"error":"Forbidden","reason":"exception thrown"}
    ```

    The operation is rejected with a **403 Forbidden** error eventhough user1 has the moderator role. The test has failed.

### Making a test pass

The previous test failed. It looks like the Sync Function doesn't honor the "moderator" role. Open `sync-gateway-config.json` and replace and add the following the `try/catch` block.

```javascript
try {
    // Users can create/update lists for themselves.
    requireUser(doc.owner);
} catch (e) {
    // Moderators can create/update lists for other users.
    requireRole("moderator");
}
```

#### Try it out

1. Save the updated configuration file and restart Sync Gateway.
2. Assign the moderator role to user1 again.

    ```bash
    curl -H 'Content-Type: application/json' \
        -X PUT 'http://localhost:4985/todo/_user/user1' \
        -d '{"admin_roles": ["moderator"]}'
    ```

3. Create a new list for user2

    ```bash
    curl -H 'Content-Type: application/json' \
        -X POST 'http://user1:pass@localhost:4984/todo/' \
        -d '{"_id": "user2.123", "name": "Groceries", "type": "task-list", "owner": "user2"}'

    {"error":"Forbidden","reason":"exception thrown"}
    ```

    The operation is accepted. The test has passed.

## Testing Update Permissions

In this section you will test that the owner property of a list cannot be changed.

|Scenario|Document Type|Type of Test|
|:-------|:------------|:---------------|
|Owner field is immutable|task-list|Validation|

### Try it out

1. Create a list for user1.

    ```bash
    curl -H 'Content-Type: application/json' \
        -X POST 'http://user1:pass@localhost:4984/todo/' \
        -d '{"_id": "user1.123", "name": "Groceries", "type": "task-list", "owner": "user1"}'

    {"id":"user1.123","ok":true,"rev":"1-7008921932d980b285d18c173e0dff1f"}
    ```

2. Change the owner property to user2.

    ```bash
    curl -H 'Content-Type: application/json' \
        -X PUT 'http://user1:pass@localhost:4984/todo/user1.123?rev=1-7008921932d980b285d18c173e0dff1f' \
        -d '{"_id": "user1.123", "name": "Groceries", "type": "task-list", "owner": "user2"}'

    {"error":"Forbidden","reason":"owner is immutable."}
    ```

    The operation is rejected with a **403 Forbidden** error. The test has passed.

## Testing Routing

In this section you will test the action of sharing a list.

|Scenario|Document Type|Type of Test|
|:-------|:------------|:---------------|
|The doc.username user is granted read access to the "task-list.{doc.taskList.id}" channel|task-list:user|Routing|

Different users have access to different channels. The Sync Gateway changes feed has a special filter property to receive changes related to a channel only.

### Try it out

1. Create a new list with 3 tasks.

    ```bash
    curl -vX POST 'http://user1:pass@localhost:4984/todo/_bulk_docs' \
        -H 'Content-Type: application/json' \
        -d '{"docs": [{"_id": "user1.123", "name": "Groceries", "type": "task-list", "owner": "user1"}, {"type": "task", "taskList": {"owner": "user1", "id": "user1.123"}, "task": "potatoes"}, {"type": "task", "taskList": {"owner": "user1", "id": "user1.123"}, "task": "tomatoes"}, {"type": "task", "taskList": {"owner": "user1", "id": "user1.123"}, "task": "apples"}]}'

    [{"id":"user1.123","rev":"1-7008921932d980b285d18c173e0dff1f"},{"id":"766233c993c6ec0e74d5e2c679d53155","rev":"1-67572fec1d27bd246ce45c691648586d"},{"id":"061f8c1e7b0d8a02571559ebf062f2ea","rev":"1-389c6e91a20adae28acd1e5f76d732b8"},{"id":"94305dc7a5ce76a75141c9836d3924a7","rev":"1-f7f5519d3f2c0a5bdf3a122c8bf93042"}]
    ```

2. Subscribe to the changes feed for the channel the document is expected to be written to.

    ```bash
    curl -vX GET 'http://user1:pass@localhost:4984/todo/_changes?feed=longpoll&since=6'
    ```

    > **Tip:** Tail the Sync Gateway logs.

3. Create a new list user to and monitor the results of the changes feed for user1.

    ```bash
    curl -vX POST 'http://user1:pass@localhost:4984/todo/' \
        -H 'Content-Type: application/json' \
        -d '{"_id": "user1.123.user2", "type": "task-list:user", "taskList": {"id": "user1.123", "owner": "user1"}, "username": "user2"}'
    ```

    The document is routed to the "task-list.user1.123" channel. The test has passed.

## Conclustion

Well done! You've completed this lesson on functional testing. In the next lesson you'll learn how to run performance tests. Feel free to share you feedback, findings or ask any questions on the forums.