# MemDB ![logo](https://github.com/memdb/memdb/wiki/images/logo.png)

[![Build Status](https://travis-ci.org/memdb/memdb.svg?branch=master)](https://travis-ci.org/memdb/memdb)
[![Dependencies Status](https://david-dm.org/memdb/memdb.svg)](https://david-dm.org/memdb/memdb)

### MongoDB with distributed ACID transaction

- __Fast and Scalable__ : Fast in memory data access. Capacity is horizontally scalable by adding more shards.

- __ACID Transaction__ : Full [ACID](https://en.wikipedia.org/wiki/ACID) transaction support on distributed environment.

- __MongoDB Compatible__ : MongoDB and Mongoose compatible.

![memdbshell.gif](https://github.com/memdb/memdb/wiki/images/memdbshell.gif)

## Documents

* [The Wiki](https://github.com/memdb/memdb/wiki)

## Quick Start

### Install Dependencies

* Install [Node.js](https://nodejs.org/download/)
* Install [Redis](http://redis.io/download)
* Install [MongoDB](https://www.mongodb.org/downloads)

__Make sure Redis and MongoDB has started__

### Install MemDB

* MemDB should be installed globally
```
sudo npm install -g memdb-server
```

### Configure MemDB

Copy default config file from `node_modules/memdb-server/memdb.conf.js` to `~/.memdb/` (mkdir if not exist), and modify it on your need. 
Please read comments carefully.

### Start MemDB

Use `memdbcluster` to control lifecycle of memdb server cluster
```
memdbcluster [start | stop | status] [--conf=memdb.conf.js] [--shard=shardId]
```

### Play with memdb shell
See the top GIF, note how ACID transaction work cross multiple shards.

### Mdbgoose

Mdbgoose is a modified __[Mongoose](http://mongoosejs.com)__ version that work for memdb

```js
var memdb = require('memdb-client');
var P = memdb.Promise;
var mdbgoose = memdb.goose;

// Define player schema
var playerSchema = new mdbgoose.Schema({
    _id : String,
    name : String,
    areaId : Number,
    deviceType : Number,
    deviceId : String,
    items : [mdbgoose.SchemaTypes.Mixed],
}, {collection : 'player'});
// Define player model
var Player = mdbgoose.model('player', playerSchema);

var main = P.coroutine(function*(){
    // Connect to memdb
    yield mdbgoose.connectAsync({
        shards : { // specify all shards here
            s1 : {host : '127.0.0.1', port: 31017},
            s2 : {host : '127.0.0.1', port: 31018},
        }
    });

    // Make a transaction in s1
    yield mdbgoose.transactionAsync(P.coroutine(function*(){

        var player = new Player({
            _id : 'p1',
            name: 'rain',
            areaId : 1,
            deviceType : 1,
            deviceId : 'id1',
            items : [],
        });

        // insert a player
        yield player.saveAsync();

        // find player by id
        var doc = yield Player.findByIdAsync('p1');
        console.log('%j', doc);

        // find player by areaId, return array of players
        var docs = yield Player.findAsync({areaId : 1});
        console.log('%j', docs);

        // find player by deviceType and deviceId
        player = yield Player.findOneAsync({deviceType : 1, deviceId : 'id1'});

        // update player
        player.areaId = 2;
        yield player.saveAsync();

        // remove the player
        yield player.removeAsync();

    }), 's1');
});

if (require.main === module) {
    main().finally(process.exit);
}
```

To run the sample above:
* Add the following index config in memdb.conf.js
```
collections : {
    player : {
        indexes : [
            {
                keys : ['areaId'],
            },
            {
                keys : ['deviceType', 'deviceId'],
                unique : true,
            },
        ]
    }
}
```
* restart memdb cluster
```
memdbcluster stop
memdbcluster start
```
* Make sure you have started shard 's1' on localhost:31017
* Install npm dependencies
```
npm install memdb-client
```
* Run with node >= 0.12 with --harmony option
```
node --harmony sample.js
```

__Check [here](https://github.com/memdb/memdb/wiki/API-Reference#mdbgoose) to see how to port your Mongoose project to Mdbgoose__


### Architecture
![architecture.png](https://github.com/memdb/memdb/wiki/images/architecture.png)

### Relationship between MemDB and MongoDB
MemDB is like a 'cache layer' built up on MongoDB which support distributed ACID transaction. 

MemDB has its own API which similar to MongoDB, however, you can still use MongoDB's native query API by directly access backend storage, here are the guidelines:
* Do simple query and update through MemDB API, which is ACID transaction safe.
* Do complex query through backend MongoDB, the read is not transaction safe.
* Do complex update through backend MongoDB __offline__ (All MemDB shards are shutdown).

Here are some basic rules for memdb:
* Data is not bind to specified shard, you can access any data from any shard.
* All operations inside a single transaction must be executed on one single shard.
* Access the same data from the same shard if possible, which will maximize performance.

### Further read
* [The Wiki](https://github.com/memdb/memdb/wiki)

## Contact Us
* [Github Issue](https://github.com/memdb/memdb/issues)
* Mailing list: [memdbd@googlegroups.com](https://groups.google.com/forum/#!forum/memdbd)
* Email: [memdbd@gmail.com](mailto:memdbd@gmail.com)

## License

Copyright 2015 The MemDB Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
implied. See the License for the specific language governing
permissions and limitations under the License. See the AUTHORS file
for names of contributors.
