# Tutorial Notes

## Start Database

`mongod --dbpath <folder>`

Defaults

-   Port: 27017
-   Data storage: /data/db or C:\data\db\*

*   Doesn't work on macOS Catalina or later, so specify the directory

## Install Process

instructions here
https://coding-boot-camp.github.io/full-stack/mongodb/how-to-install-mongodb

Left off at:
"Click OK on any remaining screens, then restart Git Bash completely. You can verify that the installation was correct by going to Git Bash and typing the following:"

## 2. Database Setup

### Mongod

get MongoDB database running locally:
`mongod --dbpath mongod_only`

E.g.:
`mongod --dbpath <folder>`

Defaults

-   Port: 27017
-   Data storage: /data/db or C:\data\db\*

Default doesn't work on macOS Catalina or later because it's read only, so specify the directory

### Replica Set

-   Data replicated in multiple places. Primary data copied to secondaries.
-   Should have at least 3 members in a replica set.
    -   Choose an uneven number of voting replica set members so if primary goes down, the others can "vote" to choose a new primary

### Set up replica set from command line

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

### Set up a replica set with config files
