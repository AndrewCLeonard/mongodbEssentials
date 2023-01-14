# Introduction

## Mongod

What is mongod?

-   core unit of MongoDB
-   daemon process for MongoDB
-   Handles data requests from the MongoDB shell or drivers
-   manages data access
-   Performs background management operations

## Start Database

`mongod --dbpath <folder>`

Defaults

-   Port: 27017
-   Data storage: /data/db or C:\data\db\*

*   Doesn't work on macOS Catalina or later, so specify the directory

## Install Process

instructions here
https://coding-boot-camp.github.io/full-stack/mongodb/how-to-install-mongodb

On Windows, if you use the Windows `mongosh` as opposed to the Linux, you _cannot_ use `fork`.

# 2. Database Setup

## Mongod

get MongoDB database running locally:
`mongod --dbpath mongod_only`

E.g.:
`mongod --dbpath <folder>`

Defaults

-   Port: 27017
-   Data storage: /data/db or C:\data\db\*

Default doesn't work on macOS Catalina or later because it's read only, so specify the directory

## Replica Set

-   Data replicated in multiple places. Primary data copied to secondaries.
-   Should have at least 3 members in a replica set.
    -   Choose an uneven number of voting replica set members so if primary goes down, the others can "vote" to choose a new primary

## Set up replica set from command line

keyfile for database to provide minimum authentification for members of replica set. But use something more secure for production
`openssl rand -base64 755 > keyfile`

add permissions
`chmod 400 keyfile`

shell parameter extensions to quickly create folders:
`mkdir -p m{1,2,3}/db`

run MongoDB processes:
`mongod --replSet myReplSet --dbpath ./m1/db --logpath ./m1/mongodb.log --port 27017 --fork --keyFile ./keyfile`
`mongod --replSet myReplSet --dbpath ./m2/db --logpath ./m2/mongodb.log --port 27018 --fork --keyFile ./keyfile`
`mongod --replSet myReplSet --dbpath ./m3/db --logpath ./m3/mongodb.log --port 27019 --fork --keyFile ./keyfile`

Now all three of these individual mongod processes are running

N.B.
At first I was getting:

```
about to fork child process, waiting until server is ready for connections.
forked process: 6013
ERROR: child process failed, exited with 48
To see additional information in this output, start without the "--fork" option.
```

See [Manage `mongod` Processes](https://www.mongodb.com/docs/manual/tutorial/manage-mongodb-processes/#start-mongod-processes) to stop running `mongod` processes and restart things

Initiate replica set and add instances

1. use `mongosh` to connect to 27017, which is the default
2. `rs.initiate()`
3. `use admin` to switch to admin db with config options
4. `db.createUser({user: 'Andy', pwd: passwordPrompt(), roles: ["root"]})`
    - user name can be whatever I choose
    - localhost exception allows you to create first user. After, you have to authenticate with privileges to create more users. Need to get this right.
    - passwordPrompt prevents pw from being visible in logs
5. `db.getSiblingDB("admin").auth("Andy", passwordPrompt())`
    - authenticate the first user we created.
6. add other 2 replica set members
    - `rs.add("localhost:27018")`
    - `rs.add("localhost:27019")`
7. check status of replica set
    - `rs.status()`
      OR
    - `db.serverStatus()["repl"]

For production environments, it's better to use config files

-   exit shell
-   `killall mongod` to shut down all instances
-   remove all files
    -   if necessary, go up a level to delete all files in that folder
    -   `rm -rif replica_set_cmdline`

## Set up a replica set with config files

set up keyfile with:
` openssl rand -base64 755 > keyfile`

for production environments, use X509 certificate instead

current user only has read permissions:
`chmod 400 keyfile`

shell parameter expansion:
`mkdir -p m{1,2,3}/db`

`touch m1.config

### create config file

normally would be copied from existing config file

keyfile for database to provide minimum authentification for members of replica set. But use something more secure for production
`openssl rand -base64 755 > keyfile`

add permissions
`chmod 400 keyfile`

| argument          | explanation                                                                                  |
| ----------------- | -------------------------------------------------------------------------------------------- |
| storage           | where the db stores its files                                                                |
| net               | server settings?                                                                             |
| security          |                                                                                              |
| systemLog         | where each mongod stores its logs. Specify destination and path                              |
| processManagement | N.B. fork only works on UNIX-like systems. Fork starts mongod as it's own individual process |
| replication       |                                                                                              |

```
storage:
    dbPath: m1/db
net:
    bindIp: localhost
    port: 27017
security:
    authorization: enabled
    keyFile: keyfile
systemLog:
    destination: file
    path: m1/mongod.log
processManagement:
    fork: true
replication:
    replsetName: mongodb-essentials-rs
```

\_this won't work on Windows **unless** you're using the Unix-like options.

copy `m1.conf` to `m2.conf` and `m3.conf`:

`cp m1.conf m2.conf`
`cp m1.conf m3.conf`

change path where mongod

1. storage
2. port
3. systemLog path

start mongod processes:
`mongod -f m1.conf`
`mongod -f m2.conf`
`mongod -f m3.conf`

connect to instances

1. open `mongosh` because I'm using default parameters here
2. `use admin` because that's where config is and where we can create users
3. `config = { _id: "mongodb-essentials-rs", members: [{_id: 0, host: "localhost:27017"}, {_id: 1, host: "localhost:27018"}, {_id: 2, host: "localhost:27019"}] }`
4. `rs.initiate(config)

Create the first user. After the first user, have to authenticate with privileges to create more users. Therefore, first user needs privileges to create other users.
\_use `passwordPrompt` so pw isn't visible in the logs
`db.createUser({user: 'andy', pwd: passwordPrompt(), roles: ["root"]})`

to authenticate, need to authenticate
`db.getSiblingDB("admin").auth("andy")`

check status of replicaset
`rs.status()`

Or, get more succinct info:
`db.serverStatus()['repl']`

You may need to restart replica set:
go back to folder and run `mongod -f m1.conf`, as well as m2 and m3.

## Import the Sample Data

### MongoDB Database Tools

| tool         | Explanation                    |
| ------------ | ------------------------------ |
| mongostat    | Statistics on a running mongod |
| mongodubp    | Export dump files to BSON      |
| mongorestore | Import dump files from BSON    |
| mongoexport  | Export data to JSON or CSV     |
| mongoimport  | Import data from JSON or CSV   |

### import datasets

```
mongoimport --username="andy" --authenticationDatabase="admin" --db=sample_data inventory.json
mongoimport --username="andy" --authenticationDatabase="admin" --db=sample_data movies.json
mongoimport --username="andy" --authenticationDatabase="admin" --db=sample_data orders.json
```

## Debug Your Deployment

-   database failing to start?
    -   check log files, e.g. `m1/mongod.log`
    -   look for errors
-   Disable the fork option
    -   change fork to false to show the error that occurs when you start `mongod`
-   check Oplog
    -   `db.oplog.rs.find( { "o.msg": { $ne: "periodic noop" } }).sort( {$natural: -1 } ).limit(1).pretty()`
-   increase the log level
    -   `db.getLogComponents()`
    -   `db.adminCommond({ setParameter: 1, LogLevel: 2 })` this would make LogLevel more verbose. Don't keep it on high level, this will slow things down
-   Go to Stack Overflow and say a prayer

# 3. Working with MongoDB

## The Document Model

-   MongoDB natively orks with JSON documents
-   You can store JSON data as it is

#### JSON and MongoDB

-   MongoDB uses a **binary-encoded serialization of JSON-like documents** called **BSON**
-   **BSON** is designed to be **lightweight, traversable,** and **efficient**
-   BSON can **store binary data** such as **images, timestamps,** and **longs**

## Namespaces, collections, and models

Data is organized into databases. A MongoDB deployment can contain multiple databases. Each database contains one or more collections. Each collection contains documents.

#### Data Organization

-   inside one database **deployment** can be one or more databases.
-   inside each database is a collection
    -   collection = grouping of MongoDB documents
    -   documents = individual records

**Example:**
`use blog`
`show collections` will be empty
(could also use `show tables`)

```
db.authors.insertOne({"name": "Naomi Pentrel"})
```

## MongoDB Query Language (MQL)

-   a tool to access data in MongoDB
-   Also called the MongoDB Query API
-   MQL allows you to perform CRUD operations
-   js-based shell

#### Insert Examples

field keys must have quotes around them. You don't need them in MongoDB shell, but you do in most other places.

```
db.authors.insertOne({"name": "Andrew Leonard"})
```

`insertMany`
_notice that it's an array_

```
db.authors.insertMany([{"name": "Elliot Horowitz"}, {"name": "Dwight Merriman"}, {"name": "Kevin P. Ryan"}])
```

#### Read Examples

```
db.authors.findOne()
```

multiple documents

```
db.authors.find()
```

match a condition:

```
db.authors.find({"name": "Andrew Leonard"})
```

#### Update Examples

update one:

```
db.authors.updateOne({"name": "Andrew Leonard"}, { $set: { website: "https://www.andrewcleonard.com"} })
```

Update many:

```
db.authors.updateMany({ }, { $set: { books: [] } })
```

#### Delete Documents

delete 1:

```
db.authors.deleteOne({ name: "Andrew Leonard"})
```

delete many:
_specifying nothing will delete everything!_

```
db.authors.deleteMany({ })
```

## Transactions

If there's no index, the db checks _every_ document. This is called a collection scan

-   not efficient

An **index** provides an **organized** way to look up data by storing a **subset of the data with pointers** to the location of the full records

-   if query can be answered with an index, it's called a **covered query**
-   more efficient queries and updates

Create an index when you frequently...

-   query on the same fields
-   perfrom range-based queries on fields
-   query same things repeatedly

Indexes have costs

-   need to be maintained = 10% write overhead

Types of Indexes

-   single field indexes (just one field)
-   partial indexes (add option to tell db to match certain conditions)
-   compound indexes (index on combination of fields. Useful if you query 2 or more queries together)
-   multikey indexes (up to 1 array value)
-   text indexes (search within text fields)
-   wildcard indexes (set of fields you don't know the name because it changes dynamically)
-   Geospatial indexes
-   hashed indexes (reduce index size)

### create an index

```
db.authors.createIndex({ name: 1 })
```

# 4. CRUD Operations

-   you can `insertOne` or `insertMany`

## **durability**

durability = a property that guarantees that acknowledged writes are _permanently stored_ in the db, even if the db or parts thereof become temporarily unavailable

durability is configurable by specifying a **writeConcern**

-   high durability - slower writes
-   low durability - faster writes

### Configuring Durability

wtimeout = time limit to prevent write operations from blocking indefinitely
j = whether we want guarantee that write be persistent to disk. Safer, but takes longer. If false, e.g. if server loses power, there's a moment of vulnerability
w = specifies number of mongod instances that need to acknowledge a write before the db tells it has been completed. Default is majority. 
```
db.authors.insertOne(
    { "name": "Naomi" },
    {
        w: "majority",
        j: "true",
        wtimeout: 100
    }
)
```

#### Which writeConcert?
cannot happen? w: "majority"
inconvenience? w: 1

## findOne and find


# My Troubleshooting Notes

Never use kill -9 (i.e. SIGKILL) to terminate a mongod instance.
https://www.mongodb.com/docs/manual/tutorial/manage-mongodb-processes/#start-mongod-processes

```

```
