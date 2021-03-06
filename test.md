### 部署与配置
#### 部署拓扑
设置合理的部署拓扑，有效保证集群的容错性。保证集群中任何一台机器宕掉，集群仍能正常的提供读写服务。
`如果宕掉的一个服务正好是配置服务器，那么所有的块拆分和迁移都会暂停。不过，暂停分片操作基本不会影响集群的正常工作，当配置服务器恢复后，就能继续快拆分和迁移了。`

> 在发生任何分片操作时，所有的配置服务器必须在线。

#### 配置注意事项
##### 对现有集合进行分片
![](index_files/89b3b045-6bc2-4eb8-8f2a-d6ec30df7636.jpg)
##### 在初始加载时预拆分块
![](index_files/218b9582-64b0-416d-84d2-93dcecded370.jpg)
![](index_files/2038a0b6-9563-4b41-8bd9-4bb09d38a426.jpg)

#### 管理
##### 删除分片
删除分片，可以通过removeshard命令删除：
```
> use admin
> db.runCommand({removeshard:"shard-1/arete:30100,arete:30101"})
{
 "msg":"draining started successfully",
 "state":"started",
 "shard":"shard-1-test-rs",
 "ok":1
}
```
可以再次运行该命令来检查删除过程：
```
> use admin
> db.runCommand({removeshard:"shard-1/arete:30100,arete:30101"})
{
 "msg":"draining ongoing",
 "state":"ongoing",
 "remaining":{
    "chunks":376,
    "dbs":3
 },
 "ok":1
}
```

一旦分片被清空，你还要确认将要删除的分片不是数据库的主分片。可以通过查询config.databases集合的分片成员来检查;
```
> use config
> db.databases.find()
{ "_id" : "cloud-docs", "primary" : "shardA", "partitioned" : true }
{ "_id" : "test", "primary" : "shardB", "partitioned" : false }
{ "_id" : "bs", "primary" : "shard3", "partitioned" : true }
{ "_id" : "cheat_history", "primary" : "shard3", "partitioned" : true }
{ "_id" : "user_identify", "primary" : "shard3", "partitioned" : false }
{ "_id" : "guilds", "primary" : "shard3", "partitioned" : true }
```
从中可以看到，cloud-docs数据库属于shardA，而test数据库则属于shardB。因为正在删除shardB，所以需要改变test数据库的主节点。为此，可以使用moveprimary命令（`该命令用于迁移在分片上剩余的非sharding数据`）:
```
> db.runCommand({moveprimary:"test",to:"shard-0-test-rs"});
```
> 一定要等分片数据迁移完成后，再迁移非分片的数据库。

在迁移完成后，
```
> use admin
> db.runCommand({removeshard:"shard-1/arete:30100,arete:30101"})
{
 "msg":"remove shard completed successfully",
 "state":"completed",
 "host":"arete:30100",
 "ok":1
}
```


