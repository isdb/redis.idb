db.c -- redisDb operation functions
redis.h -- the redisDb struct defined here
config.c -- the new store-path parameter here

Options
-------

* idb-enabled: enable/disable iDB storage. (iDBEnabled)
* idb-path: iDB store path. (iDBPath)
* idb-sync:  yes or no, write the iDB sync or async. (iDBSync)
* idb-pagesize: the max page size, pagesize <=0 means allow all keys list once (IDBMaxPageCount)
* rdb-enabled: enable/disable redis database dump file storage. (rdbEnabled)
* idb-type: iDB store type. (iDBType) deprecated.



Internal
---------

* !+ Save value as json string into iDB storage.
* !* dbsizeCommand(db.c)
* !* renameGenericCommand should be optimal
  * delete and add is not enough.
* saveDictToIDB: iDelete should ignore error!!
* deleteKeyOnIDB: supports async now.

* redis string object may be store the int. so u must be care of encoding.

          sds s = sdsempty();
          if (o->type == REDIS_STRING && o->encoding == REDIS_ENCODING_RAW) {
          size_t len = sdslen(o->ptr);
          len = len > 10 ? 10: len;
          s = sdscatrepr(s, o->ptr, len);
          }
 

* aof and replication
  * startAppendOnly(aof.c) will write all keys in dict to aof file when switching REDIS_AOF_ON
  * propagate(redis.c) will notify the changes of keys to aof and replication.
  * aof and replication will notify the changes only when server.dirty != 0 (see call func in redis.c)

        if (dirty)
            flags |= (REDIS_PROPAGATE_REPL | REDIS_PROPAGATE_AOF);
* idb-sync
  * dirtyKeys: dict
  * dirtyQueue: dict

* db.watched_keys: which clients is watching key.
    key -> clients
* redisClient
  * argc means arguments count
  * argv[] from 0 .. argc-1, 0 means self.
    * keys a?, argc[0]="keys", argc[1] = "a?"
* dict
  * dictsize(aDict) : get the count of keys in the dict

* + load/save expired info in iDB
* + remove expired keys in iDB 

* + commands
  * subkeys keyPath pattern skipCount count
* sdsnewlen(NULL, len):
  mine(idb) means: reserved free space len for NULL string.
  redis means: alloc a unknown string with length!
  now restore it, I added a new sdsalloc func for my requirement,
* redis data in iDB format(signalModifiedKey/lookupKey in db.c):
  * .type=redis
  * .value= 
        rio vRedisIO;
        rioInitWithBuffer(&vRedisIO,sdsempty());
        if (expired) {
            redisAssert(rdbSaveType(&vRedisIO,REDIS_RDB_OPCODE_EXPIRETIME_MS));
            redisAssert(rdbSaveMillisecondTime(&vRedisIO,vExpiredTime));
        }
        redisAssert(rdbSaveObjectType(&vRedisIO,val));
        redisAssert(rdbSaveObject(&vRedisIO,val));
        iPut(server.storePath, key->ptr, sdslen(key->ptr), vRedisIO.io.buffer.ptr, NULL, server.storeType);

the internal pubsub feature can notify key changed:

* notifyKeyspaceEvent(notify.c) 
  * howto get key do not triggle the events??? so I can use it as iDB save,
    * It seems that it would triggle events on read!!




Build
------

    git clone idb.c deps/idb
    make persist-settings
    make

One redis server can have many redisDb. from 0 to server.dbnum-1.

maybe I should add a new name defintion in redisDb:

the real storePath = server.storePath + '\' + redisDb.name

And I have to add a new command to set the redisDb name:

    use 1 as mydb_name

It do not necessary at all for iDB!!
