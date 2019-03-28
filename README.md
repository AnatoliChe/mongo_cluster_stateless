# mongo_cluster_stateless
Deploy MongoDB Sharded Cluster in Docker Stack in stateless way

Mongodb shareded cluster
(stateless version only for the testing proposes. If you need real bd You will need 'volumes:' in your yaml file)

             [[[   balancers 0 - 3   ]]]
    [repl set a]       [repl set b]    [cache mongos (repl setc)]
 [[[shards a0-a2]]] [[[shards b0-b2]]]   [[[conf servers c0-c2]]]

#start stack
docker stack deploy -c mongo.yaml mongo

#create shard a
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo 'rs.initiate({_id : \"a\", members: [{ _id : 0, host : \"mongo-a0\" },{ _id : 1, host : \"mongo-a1\" },{ _id : 2, host : \"mongo-a2\" }]})' | mongo --host mongo-a0"
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo 'rs.status()' | mongo --host mongo-a0"

#create shard b
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo 'rs.initiate({_id : \"b\", members: [{ _id : 0, host : \"mongo-b0\" },{ _id : 1, host : \"mongo-b1\" },{ _id : 2, host : \"mongo-b2\" }]})' | mongo --host mongo-b0"
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo 'rs.status()' | mongo --host mongo-b0"

#create config servers pool
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo 'rs.initiate({_id: \"c\",configsvr: true, members: [{ _id : 0, host : \"mongo-cfg0\" },{ _id : 1, host : \"mongo-cfg1\" }, { _id : 2, host : \"mongo-cfg2\" }]})' | mongo --host mongo-cfg0"
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo 'rs.status()' | mongo --host mongo-cfg0"

#add shards to balancer
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo -e 'sh.status()\nsh.addShard(\"a/mongo-a0\")\nsh.addShard(\"b/mongo-b0\")\nsh.status()' | mongo --host balancer"

#change chanksize
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo -e 'use config\ndb.settings.find({\"_id\": \"chunksize\"})\ndb.settings.save({\"_id\": \"chunksize\", value: 1})\n use shardTestDB)' | mongo --host balancer"

#Create collection and fullfill with data
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo -e 'use shardTestDB\nsh.enableSharding(\"shardTestDB\")\nsh.shardCollection(\"shardTestDB.users\", {\"username\": 1})\nfor(var i=0; i<100000; i++){db.users.insert({\"username\": \"user\"+i, \"created at\" : new    Date()}) }\n sh.status()' | mongo --host balancer"

#Check status and balancing of data
docker run --network=mongo_stacknet  -ti mongo  bash -c "echo -e 'sh.status()' | mongo --host balancer"

#You should see something like:
## databases:
##        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
##        {  "_id" : "shardTestDB",  "primary" : "a",  "partitioned" : true,  "version" : {  "uuid" : UUID("fd203060-eaa2-4e21-b331-9c37cab78ae6"),  "lastMod" : 1 } }
##                shardTestDB.users
##                        shard key: { "username" : 1 }
##                        unique: false
##                        balancing: true
##                        chunks:
##                                a	6
##                                b	3
##                        { "username" : { "$minKey" : 1 } } -->> { "username" : "user1" } on : b Timestamp(2, 0) 
##                        { "username" : "user1" } -->> { "username" : "user17257" } on : b Timestamp(3, 0) 
##                        { "username" : "user17257" } -->> { "username" : "user24515" } on : b Timestamp(4, 0) 
##                        { "username" : "user24515" } -->> { "username" : "user31774" } on : a Timestamp(4, 2) 
##                        { "username" : "user31774" } -->> { "username" : "user39033" } on : a Timestamp(4, 3) 
##                        { "username" : "user39033" } -->> { "username" : "user4741" } on : a Timestamp(4, 4) 
##                        { "username" : "user4741" } -->> { "username" : "user6061" } on : a Timestamp(3, 4) 
##                        { "username" : "user6061" } -->> { "username" : "user8921" } on : a Timestamp(4, 1) 
##                        { "username" : "user8921" } -->> { "username" : { "$maxKey" : 1 } } on : a Timestamp(3, 1) 
##
