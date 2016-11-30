---
id: query
title: Query
permalink: ready/guides/couchbase-lite/native-api/query/index.html
---

A **query** is the action of looking up results from a view's index. In Couchbase Lite, queries are objects of the `Query` class. To perform a query you create one of these, customize its properties (such as the key range or the maximum number of rows) and then run it. The result is a `QueryEnumerator`, which provides a list of `QueryRow` objects, each one describing one row from the view's index.

There's also a special type of query called an **all-docs query**. This type of query isn't associated with any view; or rather, you can think of it as querying an imaginary view that contains one row for every document in the database. You use an all-docs query to find all the documents in the database, or the documents with keys in a specific range, or even the documents with a specific set of keys. It can also be used to find documents with conflicts.

Couchbase Lite also provides **live queries**. Once created, a live query remains active and monitors changes to the view's index, notifying observers whenever the query results change. Live queries are very useful for driving UI components like table views.

## Creating and configuring queries

`Query` objects are created by a `View`'s `createQuery` method, and by a `Database`'s `createAllDocumentsQuery` method. In its default state a `Query` object will return every row of the index, in increasing order by key. But there are several properties you can configure to change this, before you run the query. Here are the most basic and common ones:

- `startKey`: the key to start at. The default value, `null`, means to start from the beginning.
- `endKey`: the last key to return. The default value, `null`, means to continue to the end.
- `descending`: If set to `true`, the keys will be returned in reverse order. (This also reverses the meanings of the `startKey` and `endKey` properties, since the query will now start at the highest keys and end at lower ones!)
- `limit`: If nonzero, this is the maximum number of rows that will be returned.
- `skip`: If nonzero, this many rows will be skipped (starting from the `startKey` if any.)

Some more advanced properties that aren't used as often:

- `keys`: If provided, the query will fetch only the rows with the given keys. (and `startKey` and `endKey` will be ignored.)
- `startKeyDocID`: If multiple index rows match the startKey, this property specifies that the result should start from the one(s) emitted by the document with this ID, if any. (Useful if the view contains multiple identical keys, making .startKey ambiguous.)
- `endKeyDocID`: If multiple index rows match the endKey, this property specifies that the result should end with from the one(s) emitted by the document with this ID, if any. (Useful if the view contains multiple identical keys, making .startKey ambiguous.)
- `indexUpdateMode`: Changes the behavior of index updating. By default the index will be updated if necessary before the query runs. You can choose to skip this (and get possibly-stale results), with the option of also starting an asynchronous background update of the index.
- `prefixMatchLevel`: If nonzero, enables prefix matching of string or array keys.
    - A value of 1 treats the endKey itself as a prefix: if it's a string, keys in the index that
      come after the endKey, but begin with the same prefix, will be matched. (For example, if the
      endKey is `"foo"` then the key `"foolish"` in the index will be matched, but not `"fong"`.) Or if
      the endKey is an array, any array beginning with those elements will be matched. (For
      example, if the endKey is `[1]`, then `[1, "x"]` will match, but not `[2]`.) If the key is any
      other type, there is no effect.
    - A value of 2 assumes the endKey is an array and treats its final item as a prefix, using the
      rules above. (For example, an endKey of `[1, "x"]` will match `[1, "xtc"]` but not `[1, "y"]`.)
    - A value of 3 assumes the key is an array of arrays, etc.
    
        Note that if the `.descending` property is also set, the search order is reversed and the above
    discussion applies to the `startKey`, **not** the `endKey`.

There are other advanced properties that only apply to reducing and grouping:

- `mapOnly`: If set to true, prevents the reduce function from being run, so you get all of the index rows instead of an aggregate. Has no effect if the view has no reduce function.
- `groupLevel`: If greater than zero, enables grouping of rows. The value specifies the number of items in the value array that will be grouped.

<div class="tabs"></div>

```objective-c+
// Set up a query for a view that indexes blog posts, to get the latest:
CBLQuery* query = [[self.db viewNamed: @"postsByDate"] createQuery];
query.descending = YES;
query.limit = 20;
```

```swift+
// Set up a query for a view that indexes blog posts, to get the latest:
let query = db.viewNamed("postsByDate").createQuery()
query.descending = true
query.limit = 20
```

```java+android+
// Set up a query for a view that indexes blog posts, to get the latest:
Query query = database.getView("postsByDate").createQuery();
query.setDescending(true);
query.setLimit(20);
```

```c+
// Set up a query for a view that indexes blog posts, to get the latest:
var query = database.GetView("postsByDate").CreateQuery();
query.Descending = true;
query.Limit = 20;
```

## All-documents queries

An all-docs query isn't associated with a view; or rather, you can think of it as querying an imaginary view that contains one row for every document in the database, whose key is the document ID. It supports all the standard view options, so you can query ranges of document IDs, reverse the order, and even query a specific set of documents using the `keys` property.

All-docs queries also have a special property called `allDocsMode` that can customize their behavior. Its values are:

- `allDocs`: The default mode. Returns all non-deleted documents.
- `includeDeleted`: In this mode, deleted documents are included as well.
- `showConflicts`: In this mode, each `QueryRow`'s `conflictingRevisions` property can be used to find whether it's in conflict and what the IDs of the conflicting revisions are.
- `onlyConflicts`: Like `showConflicts`, but _only_ conflicted documents are returned.

(_These are not flags._ You can only choose one.)

<div class="tabs"></div>

```objective-c+
// Let's find the documents that have conflicts so we can resolve them:
CBLQuery* query = [self.db createAllDocumentsQuery];
query.allDocsMode = kCBLOnlyConflicts;
CBLQueryEnumerator* result = [query run: &error];
for (CBLQueryRow* row in result) {
    if (row.conflictingRevisions != nil) {
        NSLog(@"!!! Conflict in document %@", row.documentID);
        [self beginConflictResolution: row.document];
    }
}
```

```swift+
// Let's find the documents that have conflicts so we can resolve them:
let query = db.createAllDocumentsQuery()
query.allDocsMode = CBLAllDocsMode.OnlyConflicts
var error: NSError?
let result = query.run(&error)
while let row = result?.nextRow() {
    NSLog("!!! Conflict in document %@", row.documentID);
    self.beginConflictResolution(row.document)
}
```

```java+
// Let's find the documents that have conflicts so we can resolve them:
Query query = database.createAllDocumentsQuery();
query.setAllDocsMode(Query.AllDocsMode.ONLY_CONFLICTS);
QueryEnumerator result = query.run();
for (Iterator<QueryRow> it = result; it.hasNext(); ) {
    QueryRow row = it.next();
    if (row.getConflictingRevisions().size() > 0) {
        Log.w("MYAPP", "Conflict in document: %s", row.getDocumentId());
        beginConflictResolution(row.getDocument());
    }
}
```

```android+
// Let's find the documents that have conflicts so we can resolve them:
Query query = database.createAllDocumentsQuery();
query.setAllDocsMode(Query.AllDocsMode.ONLY_CONFLICTS);
QueryEnumerator result = query.run();
for (Iterator<QueryRow> it = result; it.hasNext(); ) {
    QueryRow row = it.next();
    if (row.getConflictingRevisions().size() > 0) {
        Log.w("MYAPP", "Conflict in document: %s", row.getDocumentId());
        beginConflictResolution(row.getDocument());
    }
}
```

```c+
// Let's find the documents that have conflicts so we can resolve them:
var query = database.CreateAllDocumentsQuery();
query.AllDocsMode = AllDocsMode.OnlyConflicts;
var rows = query.Run();
foreach (var row in rows) 
{
    if (row.GetConflictingRevisions().Any())
    {
        Log.W(Tag, "Conflict in document: " + row.DocumentId);
        BeginConflictResolution(row.Document);
    }
}
```

## Running queries

After a `Query` object is set up just right, you call its `run` method to get the results. These are returned as a `QueryEnumerator` object, which mainly serves as an enumerable collection of `QueryRow` objects.

Each `QueryRow` has two main properties, its `key` and its `value`. These are what were emitted to the index. (Or in the case of an all-docs query, the key is the same as the document ID.) It also has a `documentID` property that identifies the document that the key and value were emitted from, although usually you'd access the `document` property instead, which gives you the `Document` object directly.

<div class="tabs"></div>

```objective-c+
// Let's query a view that maps product names to prices,
// starting with the "M"s and showing 100 widgets:
CBLQuery* query = [[self.db viewNamed: @"widgetsByName"] createQuery];
query.startKey = @"m";
query.limit = 100;
CBLQueryEnumerator* result = [query run: &error];
for (CBLQueryRow* row in result) {
    NSLog(@"Widget named %@ costs $%.2f", row.key, [row.value doubleValue]);
}
```

```swift+
// Let's query a view that maps product names to prices,
// starting with the "M"s and showing 100 widgets:
let query = db.viewNamed("widgetsByName").createQuery()
query.startKey = "m"
query.limit = 100
var error: NSError?
let result = query.run(&error)
while let row = result?.nextRow() {
    NSLog("Widget named %@ costs $%.2f", row.key as String, row.value as Double);
}
```

```java+android+
// Let's query a view that maps product names to prices,
// starting with the "M"s and showing 100 widgets:
Query query = database.getView("widgetsByName").createQuery();
query.setStartKey("m");
query.setLimit(100);
QueryEnumerator result = query.run();
for (Iterator<QueryRow> it = result; it.hasNext(); ) {
    QueryRow row = it.next();
    Log.w("MYAPP", "Widget named %s costs $%f", row.getKey(), ((Double)row.getValue()).doubleValue());
}
```

```c+
// Let's query a view that maps product names to prices,
// starting with the "M"s and showing 100 widgets:
var query = database.GetView("widgetsByName").CreateQuery();
query.StartKey = "m";
query.Limit = 100;
var rows = query.Run();
foreach (var row in rows)
{
    var name = row.Key;
    var cost = Convert.ToDouble(row.Value);
    Log.W(Tag, "Widget named " + name + " costs $" +  cost);
}
```

## Re-running queries, and LiveQuery

It's OK to run the same Query again. You can even change its settings before the next run. But if you find yourself wanting to re-run a query over and over to check for updates, there are some optimizations to consider.

First, there's a quick check to see whether the previous query results are still up to date. If you keep the QueryEnumerator object and check its `stale` property, a `false` value means that the view index hasn't changed and re-running the query won't give you a different result set.

Second, even if the enumerator says it's stale and you re-run the query, the new results might not be any different. The `stale` method is conservative and might report false positives, and even if the index did change, your query might not include any of the changed rows. You can quickly check if the new QueryEnumerator you got is equivalent to the old one by comparing the objects for equality (e.g. using `equals` in Java, or `-isEqual:` in Objective-C.)

<div class="tabs"></div>

```objective-c+
// Check whether the query result set has changed:
if (self.queryResult == nil || self.queryResult.stale) {
    CBLQueryEnumerator *newResult = [self.query run: &error];
    if (![self.queryResult isEqual: newResult]) {
        self.queryResult = newResult;
        [self updateMyUserInterface];
    }
}
```

```swift+
// Check whether the query result set has changed:
if (queryResult == nil || queryResult.stale) {
    let newResult = query.run(&error)
    if (queryResult != newResult) {
        queryResult = newResult
        self.updateMyUserInterface()
    }
}
```

```java+android+
// Check whether the query result set has changed:
if (queryResult == null || queryResult.isStale()) {
    QueryEnumerator newResult = query.run();
    if (!queryResult.equals(newResult)) {
        queryResult = newResult;
        updateMyUserInterface();
    }
}
```

```c+
// Check whether the query result set has changed:
if (queryResult == null || queryResult.Stale) 
{
    QueryEnumerator newResult = query.Run();
    if (!queryResult.Equals(newResult))
    {
        queryResult = newResult;
        UpdateMyUserInterface();
    }
}
```

There's a class that actually does this work for you, called `LiveQuery`. A live query stays active and monitors the database and view index for changes. When there's a change it re-runs itself automatically, and if the query results changed it notifies any observers. LiveQuery is a great way to build reactive user interfaces, especially table/list views, that keep themselves up to date. For example, as the replicator runs and pulls new data from the server, a LiveQuery-driven UI will automatically update to show the data without the user having to manually refresh. This helps your app feel quick and responsive.

<div class="tabs"></div>

```objective-c+
- (void) initializeQuery {
    // Set up my live query during view initialization:
    CBLQuery* query = [[self.db viewNamed: @"widgets"] createQuery];
    query.limit = 100;
    self.liveQuery = query.asLiveQuery;
    [self.liveQuery addObserver: self forKeyPath: @"rows"
                        options: 0 context: NULL];
    [self.liveQuery start];
}
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context 
{
    if (object == self.liveQuery) {
        [self displayRows: self.liveQuery.rows]; // update the UI
    }
}
```

```swift+
func initializeQuery() {
    let query = db.viewNamed("widgets").createQuery()
    query.limit = 100
    liveQuery = query.asLiveQuery()
    liveQuery.addObserver(self, forKeyPath: "rows", options: nil, context: nil)
    liveQuery.start()
}
override func observeValueForKeyPath(keyPath: String, ofObject object: AnyObject, 
    change: [NSObject : AnyObject], context: UnsafeMutablePointer<Void>) {
    if object as? NSObject == liveQuery {
        displayRows(liveQuery.rows)
    }
}
```

```java+android+
private void initializeQuery() {
    // Set up my live query during view initialization:
    Query query = database.getView("widgets").createQuery();
    query.setLimit(100);
    LiveQuery liveQuery = query.toLiveQuery();
    this.liveQuery = liveQuery;
    liveQuery.addChangeListener(new LiveQuery.ChangeListener() {
        @Override
        public void changed(LiveQuery.ChangeEvent event) {
            if (event.getSource().equals(this.liveQuery)) {
                this.displayRows(event.getRows());
            }
        }
    });
    this.liveQuery.start();
}
```

```c+
private void InitializeQuery() 
{
    // Set up my live query during view initialization:
    var query = database.GetView("widgets").CreateQuery();
    query.Limit = 100;
    liveQuery = query.ToLiveQuery();
    liveQuery.Changed += (sender, e) => DisplayRows(e.Rows);
    liveQuery.Start();
}
```

## Querying key ranges

There are some subtleties to working with key ranges (`startKey` and `endKey`.) The first is that if you reverse the order of keys, by setting the `reverse` property, then the `startKey` needs to be _greater than_ the `endKey`. That's the reason they're named _start_ and _end_, rather than _min_ and _max_. In the following example, note that the key range starts at 100 and ends at 90; if we'd done it the other way around, we'd have gotten an empty result set.

<div class="tabs"></div>

```objective-c+
// Set up a query for the highest-rated movies:
CBLQuery* query = [[self.db viewNamed: @"postsByDate"] createQuery];
query.descending = YES;
query.startKey = @100;  // Note the start key is higher than the end key
query.endKey = @90;
```

```swift+
// Set up a query for the highest-rated movies:
let query = db.viewNamed("postsByDate").createQuery()
query.descending = true
query.startKey = 100 // Note the start key is higher than the end key
query.endKey = 90
```

```java+android+
// Set up a query for the highest-rated movies:
Query query = database.getView("postsByDate").createQuery();
query.setDescending(true);
query.setStartKey(new Integer(100));
query.setEndKey(new Integer(90));
```

```c+
// Set up a query for the highest-rated movies:
var query = database.GetView("postsByDate").CreateQuery();
query.Descending = true;
query.StartKey = 100;
query.EndKey = 90;
```

Second is the handling of compound (array) keys. When a view's keys are arrays, it's very common to want to query all the rows that have a specific value (or value range) for the first element. The start key is just a one-element array with that value in it, but it's not obvious what the _end_ key should be. What works is an array that's like the starting key but with a second object appended that's greater than any possible value. For example, if the start key is (in JSON) `["red"]` then the end key could be `["red", "ZZZZ"]` ... because none of the possible second items could be greater than "ZZZZ", right? Unfortunately this has obvious problems. The correct stop value to use turns out to be an empty object/dictionary, `{}`, making the end key `["red", {}]`. This works because the sort order in views puts dictionaries last.

<div class="tabs"></div>

```objective-c+
// Assume the view's keys are like [color, model]. We want all the red ones.
CBLQuery* query = [[self.db viewNamed: @"carsByColorAndModel"] createQuery];
query.startKey = @[@"red"]
query.endKey   = @[@"red", @{}];
```

```swift+
// Assume the view's keys are like [color, model]. We want all the red ones.
let query = db.viewNamed("carsByColorAndModel").createQuery()
query.startKey = ["red"]
query.endKey = ["red",[:]]
```

```java+android+
// Assume the view's keys are like [color, model]. We want all the red ones.
Query query = database.getView("carsByColorAndModel").createQuery();
query.setStartKey("red");
query.setEndKey(Arrays.asList("red", new HashMap<String, Object>()));
```

```c+
// Assume the view's keys are like [color, model]. We want all the red ones.
var query = database.GetView("carsByColorAndModel").CreateQuery();
query.StartKey = new List<object> {"red"};
query.EndKey = new List<object> {"red", new Dictionary<string, object>()};
```

## Reducing

If the view has a reduce function, it will be run _by default_ when you query the view. This means that all rows of the output will be aggregated into a single row with no key, whose value is the output of the reduce function. (See the View documentation for a full description of what reduce functions do.)

(It's important to realize that the reduce function runs on the rows that _would be output_, not all the rows in the view. So if you set the `startKey` and/or `endKey`, the reduce function runs only on the rows in that key range.)

If you don't want the reduce function to be used, set the query's `mapOnly` property to `true`. This gives you the flexibility to use a single view for both detailed results and statistics. For example, adding a typical row-count reduce function to a view lets you get the full results (with `mapOnly=true`) or just the number of rows (with `mapOnly=false`).

<div class="tabs"></div>

```objective-c+
// This view's keys are order dates, and values are prices.
// The reduce function computes an average of the input values.
CBLQuery* query = [ordersByDateView createQuery];
query.startKey = @"2014-01-01";
query.endKey   = @"2014-02-01";
query.inclusiveEnd = NO;
// First run without reduce to get the individual orders for January '14:
query.mapOnly = YES;
for (CBLQueryRow* row in [query run: &error]) {
    NSLog(@"On %@: order for $%.2f", row.key, [row.value doubleValue])
}
// Now run with reduce to get the average order price for January '14:
query.mapOnly = NO;
CBLQueryRow* aggregate = [[query run: &error] nextRow];
NSLog(@"Average order was $%.2f", [row.value doublevalue]);
```

```swift+
// This view's keys are order dates, and values are prices.
// The reduce function computes an average of the input values.
let query = ordersByDateView.createQuery()
query.startKey = "2014-01-01"
query.endKey = "2014-02-01"
query.inclusiveEnd = false
// First run without reduce to get the individual orders for January '14:
query.mapOnly = true
var error: NSError?
var result = query.run(&error)
while let row = result?.nextRow() {
    NSLog("On %@: order for $%.2f", row.key as String, row.value as Double);
}
// Now run with reduce to get the average order price for January '14:
query.mapOnly = false
result = query.run(&error)
if let aggregate = result?.nextRow() {
    NSLog("Average order was $%.2f", aggregate.value as Double)
}
```

```java+android+
// This view's keys are order dates, and values are prices.
// The reduce function computes an average of the input values.
Query query = database.getView("ordersByDateView").createQuery();
query.setStartKey("2014-01-01");
query.setEndKey("2014-02-01");
// First run without reduce to get the individual orders for January '14:
query.setMapOnly(true);
QueryEnumerator result = query.run();
for (Iterator<QueryRow> it = result; it.hasNext(); ) {
    QueryRow row = it.next();
    Log.w("MYAPP", "On %s: order for $%f", row.getKey(), ((Double)row.getValue()).doubleValue());
}
// Now run with reduce to get the average order price for January '14:
query.setMapOnly(false);
QueryEnumerator result = query.run();
QueryRow aggregate = result.next();
Log.w("MYAPP", "Average order was $%f", ((Double)aggregate.getValue()).doubleValue());
```

```c+
// This view's keys are order dates, and values are prices.
// The reduce function computes an average of the input values.
var query = database.GetView("ordersByDateView").CreateQuery();
query.StartKey = "2014-01-01";
query.EndKey = "2014-02-01";
query.InclusiveEnd = false;
// First run without reduce to get the individual orders for January '14:
query.MapOnly = true;
var rows = query.Run();
foreach (var row in rows)
{
    var date = row.Key;
    var price = Convert.ToDouble(row.Value);
    Log.D(Tag, String.Format("On {0}: order for ${1:0.##}", date, price));
}
// Now run with reduce to get the average order price for January '14:
query.MapOnly = false;
rows = query.Run();
Debug.Assert(rows.Count > 0);
var avg = Convert.ToDouble(rows.GetRow(0).Value);
Log.D(Tag, String.Format("Average order was ${0:0.##}", avg));
```

## Grouping by key

The `groupLevel` property of a query allows you to collapse together (aggregate) rows with the same keys or key prefixes. And you can compute aggregated statistics of the grouped-together rows by using a reduce function. One very powerful use of grouping is to take a view whose keys are arrays representing a hierarchy — like `[genre, artist, album, track]` for a music library — and query a single level of the hierarchy for use in a navigation UI.

In general, `groupLevel` requires that the keys be arrays; rows with other types of keys will be ignored. When the `groupLevel` is _n_, the query combines rows that have equal values in the first n items of the key into a single row whose key is the n-item common prefix.

`groupLevel=1` is slightly different in that it supports non-array keys: it compares them for equality. In other words, if a view's keys are strings or numbers, a query with `groupLevel=1` will return a row for each _unique_ key in the index.

We've talked about the keys of grouped query rows, but what are the values? The `value` property of each row will be the result of running the view's reduce function over all the rows that were aggregated; or if the view has no reduce function, there's no value. (See the View documentation for information on reduce functions.)

Here's an interesting example. We have a database of the user's music library, and a view containing a row for every audio track, with key of the form `[genre, artist, album, trackname]` and value being the track's duration in seconds. The view has a reduce function that simply totals the input values. The user's drilled down into the genre "Mope-Rock", then artist "Radiohead", and now we want to display the albums by this artist, showing each album's running time.

<div class="tabs"></div>

```objective-c+
CBLQuery* query = [hierarchyView createQuery];
query.groupLevel = 3;
query.startKey = @[@"Mope-Rock", @"Radiohead"];
query.endKey   = @[@"Mope-Rock", @"Radiohead", @{}];
// groupLevel=3 will return [genre, artist, album] keys.
NSMutableArray* albumTitles = [NSMutableArray array];
NSMutableArray* albumTimes  = [NSMutableArray array];
for (CBLQueryRow* row in [query run: &error]) {
    [albumTitles addObject: [row keyAtIndex: 2];  // title is 3rd item of key
    [albumTimes addObject:  row.value];   // value is album's running time
}
```

```swift+
var albumTitles: [String] = []
var albumTimes: [Int] = []
var error: NSError?
let result = query.run(&error)
while let row = result?.nextRow() {
    albumTitles.append(row.keyAtIndex(2) as String)
    albumTimes.append(row.value as Int)
}
```

```java+android+
Query query = database.getView("hierarchyView").createQuery();
query.setGroupLevel(3);
query.setStartKey(Arrays.asList("Mope-Rock", "Radiohead"));
query.setEndKey(Arrays.asList("Mope-Rock", "Radiohead", new HashMap<String, Object>()));
// groupLevel=3 will return [genre, artist, album] keys.
List<String> albumTitles = new ArrayList<String>();
List<String> albumTimes = new ArrayList<String>();
QueryEnumerator result = query.run();
for (Iterator<QueryRow> it = result; it.hasNext(); ) {
    QueryRow row = it.next();
    List<String> key = (List) row.getKey();
    albumTitles.add(key.get(2)); // title is 3rd item of key
    albumTimes.add((String)row.getValue()); // value is album's running time
}
```

```c+
var query = database.GetView("hierarchyView").CreateQuery();
query.GroupLevel = 3;
query.StartKey = new List<object> {"Mope-Rock", "Radiohead"};
query.EndKey = new List<object> {"Mope-Rock", "Radiohead", new Dictionary<string, object>()};
// GroupLevel = 3 will return [genre, artist, album] keys.
var albumTitles = new List<string>();
var albumTimes = new List<string>();
var rows = query.Run();
foreach (var row in rows)
{
    var keys = row.Key.AsList<string>();
    albumTitles.Add(keys[2]);           // title is 3rd item of key
    albumTimes.Add((string)row.Value);  // value is album's running time
}
```