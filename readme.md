# Anonimongo - Another pure Nim Mongo driver
[Mongodb][1] is a document-based key-value database which emphasize in high performance read
and write capabilities together with many strategies for clustering, consistency, and availability.

Anonimongo is a driver for Mongodb developed using pure Nim. As library, it's developed to enable
developers to be able to access and use Mongodb in projects using Nim. Several implementad as
APIs for low-level which works directly with [Database][7] and higher level APIs which works with
[Collection][8]. Any casual user just have to access the [Collection API][9] directly instead of
working with various [Database operations][10].

The APIs are closely following [Mongo documentation][4] with a bit variant for `explain` API. Each supported
command for `explain` added optional string value to indicate the verbosity of `explain` command.  
By default, it's empty string which also indicate the command operations working without the need to
`explain` the queries. For others detailed caveats can be found [here](#caveats).

Almost all of the APIs are in asynchronous so the user must `await` or use `waitfor` if the
scope where it's called not an asynchronous function.  
Any API that insert/update/delete will return [WriteResult][wr-doc]. So the user can check
whether the write operation is successful or not with its field boolean `success` and the field
string `reason` to know what's the error message returned. However it's still throwing any other
error such as `BsonFetchError`, `KeyError`, `IndexError` and others kind of errors type which
raised by the underlying process. Those errors are indicating an err in the program flow hence elevated
on how the user handles them.

[This page][5] (`anonimongo.html`) is the elaborate documentation. It also explains several
modules and the categories for those modules. [The index][6] also available.

## Examples

_Click to expand_

<details><summary>Simple operations</summary>

```nim
import times, strformat
import anonimongo

var mongo = newMongo(poolconn = 16) # default is 64
if not waitFor mongo.connect:
  # default is localhost:27017
  quit &"Cannot connect to {mongo.host}.{int mongo.port}"
var coll = mongo["temptest"]["colltest"]
let currtime = now().toTime()
var idoc = newseq[BsonDocument](10)
for i in 0 .. idoc.high:
  idoc[i] = bson({
    datetime: currtime + initDuration(hours = i),
    insertId: i
  })

# insert documents
let writeRes = waitfor coll.insert(idoc)
if not writeRes.success:
  echo "Cannot insert to collection: ", coll.name
else:
  echo "inserted documents: ", writeRes.n

let id5doc = waitfor coll.findOne(bson({
  insertId: 5
}))
doAssert id5doc["datetime"] == currtime + initDuration(hours = 5)

# find one and modify, return the old document by default
let oldid8doc = waitfor coll.findAndModify(
  bson({ insertId: 8},
  bson({ "$set": { insertId: 80 }}))
)

# find one document, which newly modified
let newid8doc = waitfor coll.findOne(bson({ insertId: 80}))
doAssert oldid8doc["datetime"].ofTime == newid8doc["datetime"]

# remove a document
let delStat = waitfor coll.remove(bson({
  insertId: 9,
}), justone = true)
doAssert delStat.success  # must be true if query success
doAssert delStat.kind == wkMany # remove operation returns
                                # the WriteObject result variant
                                # of wkMany which withhold the
                                # integer n field for n affected
                                # document in successfull operation
doAssert delStat.n == 1   # because we only delete one entry in
                          # case multiple documents selected

# count all documents in current collection
let currNDoc = waitfor coll.count()
doAssert currNDoc == (idoc.len - ndeleted)

close mongo
```
</details>
<details><summary>Authenticate</summary>

```nim
import strformat
import nimSHA2
import anonimongo
import tables # needed when authenticating

var mongo = newMongo()
let mhostport = &"{mongo.host}.{$mongo.port.int}"
if waitfor not mongo.connect:
  # default is localhost:27017
  quit &"Cannot connect to {mhostport}"
if not authenticate[SHA256Digest](mongo, username, password):
  quit &"Cannot login to {mhostport}"
close mongo

# Another way to connect and login
mongo = newMongo()
mongo.username = username
mongo.password = password
if waitfor not mongo.connect and not waitfor authenticate[SHA256Digest](mongo):
  quit &"Whether cannot connect or cannot login to {mhostport}"
close mongo
```
</details>
<details><summary>URI connect</summary>

```nim
import strformat, uri
import anonimongo

let uriserver = "mongo://username:password@localhost:27017/"
var mongo = newMongo(parseURI uriserver)
close mongo
```

</details>
<details><summary>Manual/URI SSL connect</summary>

```nim
# need to compile with -d:ssl option to enable ssl
import strformat, uri
import anonimongo

let uriserver = "mongo://username:password@localhost:27017/"
let sslkey = "/path/to/ssl/key.pem"
let sslcert = "/path/to/ssl/cert.pem"
let urissl = &"{uriserver}?tlsCertificateKeyFile=certificate:{encodeURL sslcert},key:{encodeURL sslkey}"

# uri ssl connection
var mongo = newMongo(parseURI urissl)
close mongo

# manual ssl connection
mongo = newMongo(sslinfo = initSSLInfo(sslkey, sslcert))
close mongo
```

</details>
<details><summary>Upload file to GridFS</summary>

```nim
# this time the server doesn't need SSL/TLS or authentication
# gridfs is useful when the file bigger than a document capsize 16 megabytes
import anonimongo

var mongo = newMongo()
var grid = mongo["target-db"].createBucket() # by default, the bucket name is "fs"
let res = waitfor grid.uploadFile("/path/to/our/file")
if not res.success:
  echo "some error happened: ", res.reason

var gstream = waitfor grid.getStream("our-available-file")
let data = waitfor gstream.read(5.megabytes) # reading 5 megabytes of binary data
doAssert data.len == 5.megabytes
close gstream
close mongo
```

</details>
<details><summary>Bson examples</summary>

```nim
import times
var simple = bson({
  thisField: "isString",
  embedDoc: {
    embedField1: "unicodeこんにちは異世界",
    "type": "cannot use any literal or Nim keyword except string literal or symbol",
    `distinct`: true, # this is acceptable make distinct as symbol using `

    # the trailing comma is accepted
    embedTimes: now().toTime,
  },
  "1.2": 1.2,
  arraybson: [1, "hello", false], # heterogenous elements
})
doAssert simple["thisField"] == "isString"
doAssert simple["embedDoc"]["embedField1"] == "unicodeこんにちは異世界"

# explicit fetch when BsonBase cannot be automatically converted.
doAssert simple["embedDoc"]["distinct"].ofBool
doAssert simple["1.2"].ofDouble is float64

# Bson support object conversion too
type
  IntString = object
    field1: int
    field2: string

var bintstr = bson({
  field1: 1000,
  field2: "power-level"
})

let ourObj = bintstr.to IntString
doAssert ourObj.field1 == 1000
doAssert ourObj.field2 == "power-level"
```

</details>

Check [tests](tests/) for more examples of detailed usages.  
Elaborate Bson examples and cases are covered in [bson_test.nim](tests/bson.nim)


## Install

```
nimble install anonimongo
```

Or to install it locally

```
git clone https://github.com/mashingan/anonimongo
cd anonimongo
nimble develop
```

or directly from Github repo

```
nimble install https://github.com/mashingan/anonimongo 
```

### For dependency

```
requires "anonimongo >= 0.2.0"
```

or directly from Github repo

```
requires "https://github.com/mashingan/anonimongo"
```

## Implemented APIs
This implemented APIs for Mongo from [Mongo reference manual][2]
and [mongo spec][3].

<details>
<summary>Features connection</summary>

- [x] URI connect
- [x] Multiquery on URI connect
- [ ] Multihost on URI connect
- [ ] Multihost on simple connect
- [x] SSL/TLS connection
- [x] SCRAM-SHA-1 authentication
- [x] SCRAM-SHA-256 authentication
- [x] `isMaster` connection
- [x] `TailableCursor` connection
- [x] `SlaveOk` operations
- [ ] Compression connection
</details>

<details>
<summary>Features commands</summary>

<details><summary>:white_check_mark: Aggregation commands 4/4</summary>

- [x] `aggregate`
- [x] `count`
- [x] `distinct`
- [x] `mapReduce`
</details>

<details><summary>:white_check_mark: Geospatial command 1/1</summary>

- [x] `geoSearch`
</details>

<details><summary>:white_check_mark: Query and write operations commands 7/7 (<del>8</del>)</summary>

- [x] `delete`
- [x] `find`
- [x] `findAndModify`
- [x] `getMore`
- [x] `insert`
- [x] `update`
- [x] `getLastError`
- [ ] `resetError` (deprecated)
</details>

<details><summary>:x: Query plan cache commands 0/6</summary>

- [ ] `planCacheClear`
- [ ] `planCacheClearFilters`
- [ ] `planCacheListFilters`
- [ ] `planCacheListPlans`
- [ ] `planCacheListQueryShapes`
- [ ] `planCacheSetFilter`
</details>

<details><summary>:ballot_box_with_check: Database operations commands 1/3</summary>

- [x] `authenticate`, implemented as Mongo proc.
- [ ] `getnonce`
- [ ] `logout`
</details>
<details><summary>:white_check_mark: User management commands 7/7</summary>

- [x] `createUser`
- [x] `dropAllUsersFromDatabase`
- [x] `dropUser`
- [x] `grantRolesToUser`
- [x] `revokeRolesFromUser`
- [x] `updateUser`
- [x] `usersInfo`
</details>
<details><summary>:white_check_mark: Role management commands 10/10</summary>

- [x] `createRole`
- [x] `dropRole`
- [x] `dropAllRolesFromDatabase`
- [x] `grantPrivilegesToRole`
- [x] `grantRolesToRole`
- [x] `invalidateUserCache`
- [x] `revokePrivilegesFromRole`
- [x] `rovokeRolesFromRole`
- [x] `rolesInfo`
- [x] `updateRole`
</details>

<details><summary>:x: Replication commands 0/13</summary>

- [ ] `applyOps` (internal command)
- [ ] `isMaster`
- [ ] `replSetAbortPrimaryCatchUp`
- [ ] `replSetFreeze`
- [ ] `replSetGetConfig`
- [ ] `replSetGetStatus`
- [ ] `replSetGetStatus`
- [ ] `replSetInitiate`
- [ ] `replSetMaintenance`
- [ ] `replSetReconfig`
- [ ] `replSetResizeOplog`
- [ ] `replSetStepDown`
- [ ] `replSetSyncFrom`
</details>
<details><summary>:x: Sharding commands 0/27</summary>

- [ ] `addShard`
- [ ] `addShardToZone`
- [ ] `balancerStart`
- [ ] `balancerStop`
- [ ] `checkShardingIndex`
- [ ] `clearJumboFlag`
- [ ] `cleanupOrphaned`
- [ ] `enableSharding`
- [ ] `flushRouterConfig`
- [ ] `getShardMap`
- [ ] `getShardVersion`
- [ ] `isdbgrid`
- [ ] `listShard`
- [ ] `medianKey`
- [ ] `moveChunk`
- [ ] `movePrimary`
- [ ] `mergeChunks`
- [ ] `removeShard`
- [ ] `removeShardFromZone`
- [ ] `setShardVersion`
- [ ] `shardCollection`
- [ ] `shardCollection`
- [ ] `split`
- [ ] `splitChunk`
- [ ] `splitVector`
- [ ] `unsetSharding`
- [ ] `updateZoneKeyRange`
</details>
<details><summary>:x: Session commands 0/8</summary>

- [ ] `abortTransaction`
- [ ] `commitTransaction`
- [ ] `endSessions`
- [ ] `killAllSessions`
- [ ] `killAllSessionByPattern`
- [ ] `killSessions`
- [ ] `refreshSessions`
- [ ] `startSession`
</details>
<details><summary>:ballot_box_with_check: Administration commands 13/28 (<del>29</del>)</summary>

- [ ] `clean` (internal namespace command)
- [ ] `cloneCollection`
- [ ] `cloneCollectionAsCapped`
- [ ] `collMod`
- [ ] `compact`
- [ ] `connPoolSync`
- [ ] `convertToCapped`
- [x] `create`
- [x] `createIndexes`
- [x] `currentOp`
- [x] `drop`
- [x] `dropDatabase`
- [ ] `dropConnections`
- [x] `dropIndexes`
- [ ] `filemd5`
- [ ] `fsync`
- [ ] `fsyncUnlock`
- [ ] `getParameter`
- [x] `killCursors`
- [x] `killOp`
- [x] `listCollections`
- [x] `listDatabases`
- [x] `listIndexes`
- [ ] `logRotate`
- [ ] `reIndex`
- [x] `renameCollection`
- [ ] `setFeatureCompabilityVersion`
- [ ] `setParameter`
- [x] `shutdown`
</details>
<details><summary>:white_check_mark: Diagnostic commands 17/17 (<del>26</del>)</summary>

- [ ] `availableQueryOptions` (internal command)
- [x] `buildInfo`
- [x] `collStats`
- [x] `connPoolStats`
- [x] `connectionStatus`
- [ ] `cursorInfo` (removed, use metrics.cursor from `serverStatus` instead)
- [x] `dataSize`
- [x] `dbHash`
- [x] `dbStats`
- [ ] `diagLogging` (removed, on Mongo 3.6, use mongoreplay instead)
- [ ] `driverOIDTest` (internal command)
- [x] `explain`
- [ ] `features` (internal command)
- [x] `getCmdLineOpts`
- [x] `getLog`
- [x] `hostInfo`
- [ ] `isSelf` (internal command)
- [x] `listCommands`
- [ ] `netstat` (internal command)
- [x] `ping`
- [ ] `profile` (internal command)
- [x] `serverStatus`
- [x] `shardConnPoolStats`
- [x] `top`
- [x] `validate`
- [ ] `whatsmyuri` (internal command)
</details>
<details><summary>:white_check_mark: Free monitoring commands 2/2</summary>

- [x] `getFreeMonitoringStatus`
- [x] `setFreeMonitoring`
</details>
<details><summary>:x: <del>Auditing commands 0/1</del>, only available for
Mongodb Enterprise and AtlasDB </summary>

- [ ] `logApplicationMessage`
</details>
</details>

## Caveats
There are several points needed to keep in mind.

<details><summary>Those are:</summary>

* `diagnostic.explain` and its corresponding `explain`-ed version of various commands haven't
been undergone extensive testing.
* `Query` only provided for `db.find` commands. It's still not supporting Query Plan Cache or
anything regarded that.
* Cannot provide `readPreference` option because cannot support multihost URI connection.
* It's taking too long to authenticate the connection pool which default at 64 connections
even without SSL/TLS connections. It's even longer when the auth mechanism is "SCRAM-SHA-256"
which is the default. Local connection authentication taking about 1 minutes
(almost 1 second for each connection, ymmv) to finish all authentication process. This happens
when compiled in debug mode.
</details>

### License
MIT

[1]: https://www.mongodb.com
[2]: https://docs.mongodb.com/manual/reference/command/
[3]: https://github.com/mongodb/specifications
[4]: https://docs.mongodb.com/manual/reference
[5]: https://mashingan.github.io/anonimongo/src/htmldocs/anonimongo.html
[6]: https://mashingan.github.io/anonimongo/src/htmldocs/theindex.html
[7]: https://mashingan.github.io/anonimongo/src/htmldocs/anonimongo/core/types.html#Database
[8]: https://mashingan.github.io/anonimongo/src/htmldocs/anonimongo/core/types.html#Collection
[9]: https://mashingan.github.io/anonimongo/src/htmldocs/anonimongo/collections.html
[10]: https://github.com/mashingan/anonimongo/tree/master/src/anonimongo/dbops
[wr-doc]: https://mashingan.github.io/anonimongo/src/htmldocs/anonimongo/core/types.html#WriteResult