Here's a table that compares the Live Preview Mode and Deploy To Server. For channel and access changes, the Live Preview Mode is very useful. Just make sure to copy the changes to your config file once you're done!

|FAQ|Live Preview Mode|Deploy To Server|
|:--|:----------------|:---------------|
|Will it persist the changes to disk?|No|No|
|Will it persist the changes when reloading the Admin UI?|No|Yes|
|Will it keep the documents in the walrus database so I don't have to add them again?|Yes|No|



## Database Encryption

The Couchbase Lite API allows you to encrypt the database on the device. By providing an encryption key, all the data stored in the database will be secure. To decrypt it in the future, the same key must be used.

> **Note:** There are two storage types available in Couchbase Lite, _ForestDB_ and _SQLite_. This tutorial only covers _SQLite_ but you can refer to the documentation to find the instructions to include _ForestDB_ in your project.

You must first include the correct dependencies before enabling encryption. 

<block class="ios" />

- Locate the `openDatabase(username:withKey:newKey)` method in **AppDelegate.swift**, this method is called from `startSession` which is in turn getting called in `processLogin` after the user has logged in.
- The Database's `openDatabaseName` is used to create an encrypted database.

```swift
let dbname = username
let options = CBLDatabaseOptions()
options.create = true

if kEncryptionEnabled {
    if let encryptionKey = key {
        options.encryptionKey = encryptionKey
    }
}

try database = CBLManager.sharedInstance().openDatabaseNamed(dbname, withOptions: options)
if newKey != nil {
    try database.changeEncryptionKey(newKey)
}
```

### Try it out

- Drag **Extras/libsqlcipher.a** from the Couchbase Lite SDK folder to the Frameworks folder in the Xcode project.
  ![](img/image50.png)

- In the **Build Phases** tab, remove **libsqlite3.tbd** from the **Link Binary With Libraries** section and add **libsqlcipher.a**.
  ![](https://cl.ly/1K2Q1k3V473l/image49.gif)

- Set `kEncryptionEnabled` to true in **AppDelegate.swift**.

  ```swift
  let kEncryptionEnabled = true
  ```

- Delete the application if you ran it previously without encryption.
    <img src="img/image17.png" class="portrait" />

- Build and run.
- Browse to the database file and you'll find that it's now encrypted.

## Offline Login

What about the scenario where a user attempts to login while being offline? Since data may already be in the database, it would be nice to allow the app to use it. From the previous section, you’ve learned that the database can be safely accessed with an encryption key so the user can attempt to enter credentials as if they were online (the **name** and **password** of the Sync Gateway user). To achieve offline login, you are mapping the user’s **name** to be the **database name** and the user’s **password** to be the **encryption key**.

If the user enters an incorrect password (that is not the valid encryption key), the database will return an error with a status code of 401 to indicate it’s the wrong password.

To make it even more challenging, consider the scenario where a user’s password has been modified on Sync Gateway. Since the encryption key is the old password it will not be accessible, so when the user’s password changes on Sync Gateway the local encryption key and server password will be out of sync. To detect that, you’re going to use the replication change event and check for the status code. If it’s a 401, you will logout the user and display the login screen. However, the data stored in the database will have changes that were not pushed to Sync Gateway yet. You can offer users the ability to update the encryption key only if they remember their old password (because it’s the only way to open and read data from the database encrypted with the old password). If they don’t remember it, the changes made while they were offline will be lost. The marble diagram below shows every scenario and outcome.

<img src="img/image24.png" class="center-image" />

In **AppDelegate.swift**, scroll to the `processLogin` method. Notice that if an error with status 401 is thrown it calls `handleEncryptionError` which displays a popup with an input text and two options.

<img src="img/image27.png" class="portrait" />

When the user clicks **Delete** it will remove the database and create a new one with no data in it. If the user remembers the old password and clicks **Migrate** it will call the `processLogin` method again passing it the old and new encryption keys. The following code at the end of the `openDatabase` method is changing that encryption key to the user's new password.

```swift
if newKey != nil {
    try database.changeEncryptionKey(newKey)
}
```

Build and run. Change the password for the user with the following curl request.

```bash
curl -vX PUT 'http://localhost:4985/todo/_user/user1' \
      -H 'Content-Type: application/json' \
      -d '{"name": "user1","password": "newpass"}'
```

Add a new list which will wake up the replicator to push the new document. Since the user credentials have changed, the task list will not be persisted to Sync Gateway. The way to get notified of this is by checking the replication `lastError` property in the replication change event. In this case it will be a **401 Unauthorized**.

The **lastError** property is checked in the `replicationProgress` method of **AppDelegate.swift**.

```swift
func replicationProgress(notification: NSNotification) {
    UIApplication.sharedApplication().networkActivityIndicatorVisible =
        (pusher.status == .Active || puller.status == .Active)
    let error = pusher.lastError ?? puller.lastError
    if (error != syncError) {
        syncError = error
        if let errorCode = error?.code {
            NSLog("Replication Error: %@", error!)
            if errorCode == 401 {
                Ui.showMessageDialog(
                    onController: self.window!.rootViewController!,
                    withTitle: "Authentication Error",
                    withMessage:"Your username or password is not correct.",
                    withError: nil,
                    onClose: {
                        self.logout()
                })
            }
        }
    }
}
```

If the error code is 401 it displays the popup below and logs the user out of the application.

<img src="./img/image18.png" class="portrait" />

Click **OK** and on the login screen type **user1** and **newpass**, the new password on the login screen.

<img src="./img/image00.png" class="portrait" />

Click **Login**. Next, type **pass** on the "Password Changed" popup, the previous password.

<img src="./img/image14.png" class="portrait" />

Click **Migrate**. Notice that the list added after the password changed is there.

<img src="./img/image10.png" class="portrait" />

### Key Rotation

Key rotation is defined as the process of decrypting data with an old key and re-keying the data with a new one. The benefits of key rotation are all centered on security; for example, if the password to sensitive data is being shared between many users, you may decide to use key rotation to add an extra layer of security. By regularly changing the password you will mitigate the scenario where the encryption key can be compromised unknowingly. Rotating your keys offers more protection and better security for your sensitive business data, but it is not a requirement and it should be considered on a per-application basis.

<block class="ios rn" />