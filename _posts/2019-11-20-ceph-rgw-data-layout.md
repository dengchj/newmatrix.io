[TOC]
# Overview
RGW 中三个基本概念：user， bucket， object。通过分析RGW data layout，可以清楚对象存储的三个基本概念是怎样在RGW 中实现的。

RGW 中数据分三种类型：

- data: 每个RGW object 会保存在一个或多个Rados object(s) 
- metadata: user,  bucket,  bucket.instance
- bucket index: 单独作为一种metadata
RGW 的数据在RADOS 层以三种形式存在：rados对象（data 部分），rados 对象扩展属性xattr，rados 对象的omap中。

## metadata
RGW metadata分为：

- user：保存user 信息
- bucket：维护一组bucket name 和bucket instance id 的映射
- bucket.instance：保存bucket instance 信息
- otp：N 版新增，otp (one-time password) 信息。根据虚拟或硬件Multi-Factor Authentication (MFA) 设备，基于已经时间同步的otp 算法生成一个密码。
metadata list，可以看到是上面列出的四种类型：
```
[root@stor14 build]# bin/radosgw-admin metadata list -c ceph.conf
[
    "bucket",
    "bucket.instance",
    "otp",
    "user"
]
```
查看user 下都有什么 ：
```
[root@stor14 build]# bin/radosgw-admin metadata list user -c ceph.conf
[
    "56789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01234",
    "bl_deliver",
    "testx$9876543210abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "user1",
    "test",
    "testid",
    "ms_sync"
]
```
list bucket.instance 看下，一般一个bucket 对应一个bucket.instance
```
[root@stor14 build]# bin/radosgw-admin metadata list bucket.instance -c ceph.conf
[
"bltest1:e34456f0-d371-4384-9007-70c60563fb0b.4281.17",
"test1:e34456f0-d371-4384-9007-70c60563fb0b.4281.15"
]
``` 

接着看下bltest1:e34456f0-d371-4384-9007-70c60563fb0b.4281.17 这个bucket.instance
```
[root@stor14 build]# bin/radosgw-admin metadata get bucket.instance:bltest1:e34456f0-d371-4384-9007-70c60563fb0b.4281.17 -c ceph.conf
{
    "key": "bucket.instance:bltest1:e34456f0-d371-4384-9007-70c60563fb0b.4281.17",
    "ver": {
        "tag": "_UHAkf8NRdh_eXm7CqYsFNA8",
        "ver": 1
    },
    "mtime": "2019-11-20 07:53:39.493618Z",
    "data": {
        "bucket_info": {
            "bucket": {
                "name": "bltest1",
                "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4281.17",
                "bucket_id": "e34456f0-d371-4384-9007-70c60563fb0b.4281.17",
                "tenant": "",
                "explicit_placement": {
                    "data_pool": "",
                    "data_extra_pool": "",
                    "index_pool": ""
                }
            },
            "creation_time": "2019-11-20 07:53:39.399284Z",
            "owner": "user1",
            "flags": 0,
            "zonegroup": "513b96de-2450-4292-a86a-314abbe29766",
            "placement_rule": "default-placement",
            "has_instance_obj": "true",
            "quota": {
                "enabled": false,
                "check_on_raw": false,
                "max_size": -1,
                "max_size_kb": 0,
                "max_objects": -1
            },
            "num_shards": 0,
            "bi_shard_hash_type": 0,
            "requester_pays": "false",
            "has_website": "false",
            "swift_versioning": "false",
            "swift_ver_location": "",
            "index_type": 0,
            "mdsearch_config": [],
            "reshard_status": 0,
            "new_bucket_instance_id": ""
        },
        "attrs": [
            {
                "key": "user.rgw.acl",
                "val": "AgKBAAAAAwISAAAABQAAAHVzZXIxBQAAAHVzZXIxBANjAAAAAQEAAAAFAAAAdXNlcjEPAAAAAQAAAAUAAAB1c2VyMQUDNgAAAAICBAAAAAAAAAAFAAAAdXNlcjEAAAAAAAAAAAICBAAAAA8AAAAFAAAAdXNlcjEAAAAAAAAAAAAAAAAAAAAA"
            }
        ]
    }
}
```
获取bltest1 这个存储桶相关信息：
```
[root@stor14 build]# bin/radosgw-admin metadata get bucket:bltest1 -c ceph.conf
{
    "key": "bucket:bltest1",
    "ver": {
        "tag": "_Sia7A02mjQlrvVkymKa_y2G",
        "ver": 1
    },
    "mtime": "2019-11-20 07:53:39.560897Z",
    "data": {
        "bucket": {
            "name": "bltest1",
            "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4281.17",
            "bucket_id": "e34456f0-d371-4384-9007-70c60563fb0b.4281.17",
            "tenant": "",
            "explicit_placement": {
                "data_pool": "",
                "data_extra_pool": "",
                "index_pool": ""
            }
        },
        "owner": "user1",
        "creation_time": "2019-11-20 07:53:39.399284Z",
        "linked": "true",
        "has_bucket_info": "false"
    }
}
``` 

由于我们没有做MFA 相关配置，otp 目前为空：
```
[root@stor14 build]# bin/radosgw-admin metadata list otp -c ceph.conf
[]
```
## bucket index
bucket index 是一类特殊的metadata，通过bucket index 我们可以list 指定存储桶下的所有rgw 对象。
bucket index 对象保存在存储池<zone>.rgw.buckets.index，名为 “.dir.<marker>” 的rados object。

bucket index 维护了一个k-v map，

- k-v map本身保存在rados object(s) 关联的omap 中。在不启用shard 时，一个存储桶会对应一个rados object；若shard 后，一个存储桶可能会对应多个index rados objects。
- omap 的key 为各rgw object name，omap value 这些rgw object 的一些基本元数据，如list bucket 时展示的元数据。
- 每个omap 有一个header，在header 中保存一些bucket的统计信息（对象数，总大小等）?


list 一下bucket index 对象（rados object）：
```
[root@stor14 build]# bin/rados -c ceph.conf ls -p default.rgw.buckets.index
.dir.e34456f0-d371-4384-9007-70c60563fb0b.4281.15
.dir.e34456f0-d371-4384-9007-70c60563fb0b.4281.17
```
可以看到两个index objects，此时default zone 下只有两个存储桶。
```
[root@stor14 build]# bin/radosgw-admin bucket list -c ceph.conf
[
    "test1",
    "bltest1"
]
```
其中对应存储桶bltest1
```
[root@stor14 build]# bin/radosgw-admin bucket stats --bucket test1 -c ceph.conf
{
    "bucket": "test1",
    "tenant": "",
    "zonegroup": "513b96de-2450-4292-a86a-314abbe29766",
    "placement_rule": "default-placement",
    "explicit_placement": {
        "data_pool": "",
        "data_extra_pool": "",
        "index_pool": ""
    },
    "id": "e34456f0-d371-4384-9007-70c60563fb0b.4281.15",
    "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4281.15", # 由marker可以看到对应index 对象 .dir.e34456f0-d371-4384-9007-70c60563fb0b.4281.15
    "index_type": "Normal",
    "owner": "user1",
    "ver": "0#3",
    "master_ver": "0#0",
    "mtime": "2019-11-20 07:53:38.898208Z",
    "max_marker": "0#",
    "usage": {
        "rgw.main": {
            "size": 10279,
            "size_actual": 16384,
            "size_utilized": 10279,
            "size_kb": 11,
            "size_kb_actual": 16,
            "size_kb_utilized": 11,
            "num_objects": 2
        }
    },
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    }
}
```
listomapkeys 看下
```
[root@stor14 build]# bin/rados -c ceph.conf listomapkeys .dir.e34456f0-d371-4384-9007-70c60563fb0b.4281.15 -p default.rgw.buckets.index
dcj1
dcj2
可以看到omap key：dcj1 和dcj2，再listomapvals 看下，此时可以看2个RGW 对象

[root@stor14 build]# bin/rados -c ceph.conf listomapvals .dir.e34456f0-d371-4384-9007-70c60563fb0b.4281.15 -p default.rgw.buckets.index
dcj1
value (218 bytes) :
00000000  08 03 d4 00 00 00 04 00  00 00 64 63 6a 31 01 00  |..........dcj1..|
00000010  00 00 00 00 00 00 01 07  03 6e 00 00 00 01 ca 13  |.........n......|
00000020  00 00 00 00 00 00 06 f1  d4 5d f1 9c e0 0e 20 00  |.........].... .|
00000030  00 00 38 30 64 34 36 33  66 36 32 35 65 39 32 37  |..80d463f625e927|
00000040  39 31 39 35 65 30 37 34  65 35 34 65 65 63 64 31  |9195e074e54eecd1|
00000050  34 39 05 00 00 00 75 73  65 72 31 05 00 00 00 75  |49....user1....u|
00000060  73 65 72 31 0a 00 00 00  74 65 78 74 2f 70 6c 61  |ser1....text/pla|
00000070  69 6e ca 13 00 00 00 00  00 00 00 00 00 00 08 00  |in..............|
00000080  00 00 53 54 41 4e 44 41  52 44 00 00 00 00 00 00  |..STANDARD......|
00000090  00 00 00 01 01 02 00 00  00 08 01 01 2c 00 00 00  |............,...|
000000a0  65 33 34 34 35 36 66 30  2d 64 33 37 31 2d 34 33  |e34456f0-d371-43|
000000b0  38 34 2d 39 30 30 37 2d  37 30 63 36 30 35 36 33  |84-9007-70c60563|
000000c0  66 62 30 62 2e 34 32 36  37 2e 35 34 00 00 00 00  |fb0b.4267.54....|
000000d0  00 00 00 00 00 00 00 00  00 00                    |..........|
000000da


dcj2
value (220 bytes) :
00000000  08 03 d6 00 00 00 04 00  00 00 64 63 6a 32 01 00  |..........dcj2..|
00000010  00 00 00 00 00 00 01 07  03 6e 00 00 00 01 5d 14  |.........n....].|
00000020  00 00 00 00 00 00 54 2e  d5 5d 49 86 f8 2a 20 00  |......T..]I..* .|
00000030  00 00 38 39 36 39 39 37  63 37 63 37 62 64 64 61  |..896997c7c7bdda|
00000040  30 38 61 62 30 39 62 38  35 34 36 32 33 61 30 30  |08ab09b854623a00|
00000050  30 30 05 00 00 00 75 73  65 72 31 05 00 00 00 75  |00....user1....u|
00000060  73 65 72 31 0a 00 00 00  74 65 78 74 2f 70 6c 61  |ser1....text/pla|
00000070  69 6e 5d 14 00 00 00 00  00 00 00 00 00 00 08 00  |in].............|
00000080  00 00 53 54 41 4e 44 41  52 44 00 00 00 00 00 00  |..STANDARD......|
00000090  00 00 00 01 01 02 00 00  00 08 01 02 2e 00 00 00  |................|
000000a0  65 33 34 34 35 36 66 30  2d 64 33 37 31 2d 34 33  |e34456f0-d371-43|
000000b0  38 34 2d 39 30 30 37 2d  37 30 63 36 30 35 36 33  |84-9007-70c60563|
000000c0  66 62 30 62 2e 34 35 36  34 2e 34 30 30 36 00 00  |fb0b.4564.4006..|
000000d0  00 00 00 00 00 00 00 00  00 00 00 00              |............|
000000dc
``` 
一般一个存储桶对应一个rados object，H版之后加入shard 后，也可能会一个存储桶对应多个rados objects。目前N 版已经可以做到auto reshard。我们通过配置rgw_override_bucket_index_max_shards = 5 将新建存储桶的shard 初始化为5 分片

新建存储桶test2 
```
[root@stor14 build]# s3cmd mb s3://test2
Bucket 's3://test2/' created
可以看到max_marker 有5个：0#,1#,2#,3#,4#
[root@stor14 build]# bin/radosgw-admin bucket stats --bucket test2 -c ceph.conf
{
    "bucket": "test2",
    "tenant": "",
    "zonegroup": "513b96de-2450-4292-a86a-314abbe29766",
    "placement_rule": "default-placement",
    "explicit_placement": {
        "data_pool": "",
        "data_extra_pool": "",
        "index_pool": ""
    },
    "id": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
    "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
    "index_type": "Normal",
    "owner": "user1",
    "ver": "0#1,1#1,2#1,3#1,4#1",
    "master_ver": "0#0,1#0,2#0,3#0,4#0",
    "mtime": "2019-11-20 12:26:38.164727Z",
    "max_marker": "0#,1#,2#,3#,4#",
    "usage": {},
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    }
}
```
此时除了之前的2个 index objects 外，多出来了5个 对象，名为 .dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.<NUMBERING>
```
[root@stor14 build]# bin/rados -c ceph.conf ls -p default.rgw.buckets.index
.dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.1
.dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.3
.dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.2
.dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.4
.dir.e34456f0-d371-4384-9007-70c60563fb0b.4281.15
.dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.0
.dir.e34456f0-d371-4384-9007-70c60563fb0b.4281.17
``` 
我们上传一个对象jcd1 至新建的存储桶test2
```
[root@stor14 build]# s3cmd ls s3://test2
[root@stor14 build]# s3cmd put ./ceph.conf s3://test2/jcd1
upload: './ceph.conf' -> 's3://test2/jcd1'  [1 of 1]
 5255 of 5255   100% in    0s    26.82 kB/s  done
[root@stor14 build]# bin/rados -c ceph.conf listomapkeys .dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.0 -p default.rgw.buckets.index

[root@stor14 build]# bin/rados -c ceph.conf listomapkeys .dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.1 -p default.rgw.buckets.index

[root@stor14 build]# bin/rados -c ceph.conf listomapkeys .dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.2 -p default.rgw.buckets.index
jcd1
[root@stor14 build]# bin/rados -c ceph.conf listomapkeys .dir.e34456f0-d371-4384-9007-70c60563fb0b.4848.1.3 -p default.rgw.buckets.index
```

可以看到上传的对象在分片2 中。
默认情况下bucket index obj 并没有attr 信息：
```
[root@stor14 build]# bin/rados -c ceph.conf listxattr .dir.e34456f0-d371-4384-9007-70c60563fb0b.4281.15 -p default.rgw.buckets.index
```
一般来说单shard 保存10万-15万对象为宜，shard 数也不是越多越好，过多的shard会导致部分类似list bucket的操作消耗大量底层存储IO，导致部分请求耗时过长。 

## data
每个RGW object 会保存在一个或多个rados obj 中。下面会详细介绍RGW Object 的layout。

# RGW Pools
RGW 使用各种 pool 来专门存放特定类型的数据, zone 所使用的存储池的命名 规则 {zone}.rgw.{functions} 。

可以看到default zone下配置的存储池：
```
[root@stor14 build]# bin/radosgw-admin zone get -c ceph.conf
{
    "id": "e34456f0-d371-4384-9007-70c60563fb0b",
    "name": "default",
    "domain_root": "default.rgw.meta:root",
    "control_pool": "default.rgw.control",
    "gc_pool": "default.rgw.log:gc",
    "lc_pool": "default.rgw.log:lc",
    "bl_pool": "default.rgw.log:bl",
    "log_pool": "default.rgw.log",
    "intent_log_pool": "default.rgw.log:intent",
    "usage_log_pool": "default.rgw.log:usage",
    "reshard_pool": "default.rgw.log:reshard",
    "user_keys_pool": "default.rgw.meta:users.keys",
    "user_email_pool": "default.rgw.meta:users.email",
    "user_swift_pool": "default.rgw.meta:users.swift",
    "user_uid_pool": "default.rgw.meta:users.uid",
    "otp_pool": "default.rgw.otp", # metadata 中提到的One-Time Password 用的存储池
    "system_key": {
        "access_key": "ms_sync",
        "secret_key": "ms_sync"
    },
    "bl_deliver_key": {
        "access_key": "bl_deliver",
        "secret_key": "bl_deliver"
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "default.rgw.buckets.index", 
                "storage_classes": {
                    "STANDARD": {
                        "data_pool": "default.rgw.buckets.data"
                    }
                },
                "data_extra_pool": "default.rgw.buckets.non-ec",
                "index_type": 0
            }
        }
    ],
    "metadata_heap": "",
    "realm_id": "5f47bd7c-94a1-48f5-8c51-5bb8f7075425"
}
```
**重要的RGW Pools：**

* .rgw.root - 保存 zone,zg,realm,period 相关的信息

* zone.rgw.control - librados 提供的 watch-notify 机制保证缓存一致性,如 notify.<N>

* zone.rgw.log - 记录不同的日志,--namespace usage, gc, lc, bl
    - gc:  用于垃圾回收的
    - lc: 用于lifecycle 
    - bl：用于bucket logging

* zone.rgw.buckets.data ,数据池

* zone.rgw.buckets.extra ,现在叫 rgw.buckets.non-ec ,multipart upload 过程中的临时数据会放这里。

zone.rgw.buckets.index - 存储 bucket 中对象的索引对象： “.dir.<marker>”

* zone.rgw.meta: 对象存储元数据池, --namespace users, root

    - root：<bucket> .bucket.meta.<bucket>:<marker> # see put_bucket_instance_info()

    - users.id：Contains _both_ per-user information (RGWUserInfo) in “<user>” objects and per-user lists of buckets in omaps of “<user>.buckets” objects. The “<user>” may contain the tenant if non-empty
    - users.keys：允许RGW在认证时通过access-key 去找到user id
看下<zone>.rgw.meta 池
```
[root@stor14 build]# bin/rados -c ceph.conf ls -p default.rgw.meta -N root
test1
.bucket.meta.test2:e34456f0-d371-4384-9007-70c60563fb0b.4848.1
.bucket.meta.bltest1:e34456f0-d371-4384-9007-70c60563fb0b.4281.17
bltest1
.bucket.meta.test1:e34456f0-d371-4384-9007-70c60563fb0b.4281.15
test2
[root@stor14 build]# bin/rados -c ceph.conf listxattr test1 -p default.rgw.meta -N root
ceph.objclass.version
[root@stor14 build]# bin/rados -c ceph.conf listxattr .bucket.meta.test1:e34456f0-d371-4384-9007-70c60563fb0b.4281.15 -p default.rgw.meta -N root
ceph.objclass.version
user.rgw.acl
```

看下users.uid

```
[root@stor14 build]# bin/rados -c ceph.conf ls -p default.rgw.meta -N users.uid
56789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01234
bl_deliver
testx$9876543210abcdef0123456789abcdef0123456789abcdef0123456789abcdef
0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
user1
test
testid
ms_sync
user1.buckets # *****
 
[root@stor14 build]# bin/radosgw-admin user list -c ceph.conf
[
    "56789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01234",
    "bl_deliver",
    "testx$9876543210abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "user1",
    "test",
    "testid",
    "ms_sync"
]
```
可以看到users.uid 中除了oss user，还有一个user1.buckets
“Contains _both_ per-user information (RGWUserInfo) in “<user>” objects and per-user lists of buckets in omaps of “<user>.buckets” objects. The “<user>” may contain the tenant if non-empty”

其实这个user1.buckets 对象是一个非常重要的对象。通过bucket index obj 可以list 某个存储桶中的rgw objects，如何list 某个对象存储用户下的存储桶呢？答案就是这个<user>.buckets 对象的omap 中list，指定用户下的所有存储桶信息会在<user>.buckets 对象的omap 中保存：
```
[root@stor14 build]# bin/rados -c ceph.conf listomapkeys user1.buckets -p default.rgw.meta -N users.uid
bltest1
test1
test2
[root@stor14 build]# bin/rados -c ceph.conf listomapvals user1.buckets -p default.rgw.meta -N users.uid
bltest1
value (172 bytes) :
00000000  09 05 a6 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000010  00 00 03 f1 d4 5d 00 00  00 00 00 00 00 00 07 03  |.....]..........|
00000020  77 00 00 00 07 00 00 00  62 6c 74 65 73 74 31 00  |w.......bltest1.|
00000030  00 00 00 2c 00 00 00 65  33 34 34 35 36 66 30 2d  |...,...e34456f0-|
00000040  64 33 37 31 2d 34 33 38  34 2d 39 30 30 37 2d 37  |d371-4384-9007-7|
00000050  30 63 36 30 35 36 33 66  62 30 62 2e 34 32 38 31  |0c60563fb0b.4281|
00000060  2e 31 37 2c 00 00 00 65  33 34 34 35 36 66 30 2d  |.17,...e34456f0-|
00000070  64 33 37 31 2d 34 33 38  34 2d 39 30 30 37 2d 37  |d371-4384-9007-7|
00000080  30 63 36 30 35 36 33 66  62 30 62 2e 34 32 38 31  |0c60563fb0b.4281|
00000090  2e 31 37 00 00 00 00 00  00 00 00 00 00 00 00 00  |.17.............|
000000a0  00 00 00 01 03 f1 d4 5d  26 99 cc 17              |.......]&...|
000000ac
test1
value (170 bytes) :
00000000  09 05 a4 00 00 00 00 00  00 00 27 28 00 00 00 00  |..........'(....|
00000010  00 00 02 f1 d4 5d 02 00  00 00 00 00 00 00 07 03  |.....]..........|
00000020  75 00 00 00 05 00 00 00  74 65 73 74 31 00 00 00  |u.......test1...|
00000030  00 2c 00 00 00 65 33 34  34 35 36 66 30 2d 64 33  |.,...e34456f0-d3|
00000040  37 31 2d 34 33 38 34 2d  39 30 30 37 2d 37 30 63  |71-4384-9007-70c|
00000050  36 30 35 36 33 66 62 30  62 2e 34 32 38 31 2e 31  |60563fb0b.4281.1|
00000060  35 2c 00 00 00 65 33 34  34 35 36 66 30 2d 64 33  |5,...e34456f0-d3|
00000070  37 31 2d 34 33 38 34 2d  39 30 30 37 2d 37 30 63  |71-4384-9007-70c|
00000080  36 30 35 36 33 66 62 30  62 2e 34 32 38 31 2e 31  |60563fb0b.4281.1|
00000090  35 00 00 00 00 00 00 00  00 00 40 00 00 00 00 00  |5.........@.....|
000000a0  00 01 02 f1 d4 5d 74 a9  42 32                    |.....]t.B2|
000000aa
test2
value (168 bytes) :
00000000  09 05 a2 00 00 00 00 00  00 00 87 14 20 02 00 00  |............ ...|
00000010  00 00 fd 30 d5 5d 05 00  00 00 00 00 00 00 07 03  |...0.]..........|
00000020  73 00 00 00 05 00 00 00  74 65 73 74 32 00 00 00  |s.......test2...|
00000030  00 2b 00 00 00 65 33 34  34 35 36 66 30 2d 64 33  |.+...e34456f0-d3|
00000040  37 31 2d 34 33 38 34 2d  39 30 30 37 2d 37 30 63  |71-4384-9007-70c|
00000050  36 30 35 36 33 66 62 30  62 2e 34 38 34 38 2e 31  |60563fb0b.4848.1|
00000060  2b 00 00 00 65 33 34 34  35 36 66 30 2d 64 33 37  |+...e34456f0-d37|
00000070  31 2d 34 33 38 34 2d 39  30 30 37 2d 37 30 63 36  |1-4384-9007-70c6|
00000080  30 35 36 33 66 62 30 62  2e 34 38 34 38 2e 31 00  |0563fb0b.4848.1.|
00000090  00 00 00 00 00 00 00 00  20 20 02 00 00 00 00 01  |........  ......|
000000a0  fd 30 d5 5d 57 b6 f2 3a                           |.0.]W..:|
000000a8
``` 

# RGW Object 
一个RGW Object 有一个或多个Rados Object 组成。RGW object 逻辑上一般分为head 和tail。

其中head( object logiccal head, olh)，元数据会保存在扩展属性xattr中:
- head 保存在单个 RADOS object
- head 是有固定的大小的(rgw_max_chunk_size 默认 4M)
- 内容可变 保存 RGW object 的元数据
    - ACLs
    - Manifest
    - Conten-Type
    - ETag
    - User-Defined Metadata(保存在 xattrs)

tail 部分:
- 内容不可变
- 对于小于 rgw_max_chunk_size 的 rgw-object 没有 tail
- 对于multipart 上传的对象，tail 分multipart （每个part 的第一个rados对象）和shadow 
- tail 被分隔成若干个 part，单个 part 保存 rgw-object 的连续数据,每个 part 默认 15M
- 普通的 rgw-object 只有单个 part(但 part 内可以有很多的 stripe),对于 multipart - 上传的 rgw-object 就有多个 part(最后一个 part 可能比较小)。 注意 part 还是逻辑上的
- 每个 part 都是 striped, striped part 由默认的 stripe size,每个 stripe part 映射到 rados-object,stripe 和 rados-object 一一对应

验证一下：
```
[root@stor14 build]# s3cmd put ./4MB s3://test2
upload: './4MB' -> 's3://test2/4MB'  [1 of 1]
 4194304 of 4194304   100% in    0s     8.14 MB/s  done
[root@stor14 build]# bin/rados -c ceph.conf ls -p default.rgw.buckets.data
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_jcd1
e34456f0-d371-4384-9007-70c60563fb0b.4281.15_dcj2
e34456f0-d371-4384-9007-70c60563fb0b.4281.15_dcj1
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_4MB
[root@stor14 build]# dd if=/dev/zero of=./5MB count=5 bs=1024k
记录了5+0 的读入
记录了5+0 的写出
5242880字节(5.2 MB)已复制，0.00357833 秒，1.5 GB/秒
[root@stor14 build]# s3cmd put ./5MB s3://test2
upload: './5MB' -> 's3://test2/5MB'  [1 of 1]
 5242880 of 5242880   100% in    0s     7.18 MB/s  done
[root@stor14 build]# bin/rados -c ceph.conf ls -p default.rgw.buckets.data
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_jcd1
e34456f0-d371-4384-9007-70c60563fb0b.4281.15_dcj2
e34456f0-d371-4384-9007-70c60563fb0b.4281.15_dcj1
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_4MB
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_5MB
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_.lWnArC6BcKPkRjbGf-B0q23BSpxjql1_1
``` 
我们先后有上传了2个对象:一个4MB，一个5MB，大小同对象名。可以看到，超过4MB 之后，RGW 对象分为了2 个RADOS 对象，其中
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_5MB 为head，e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_.lWnArC6BcKPkRjbGf-B0q23BSpxjql1_1 为tail。
查看2个rados obj 的大小：
```
[root@stor14 build]# bin/rados -c ceph.conf -p default.rgw.buckets.data stat e34456f0-d371-4384-9007-70c60563fb0b.4848.1_5MB
default.rgw.buckets.data/e34456f0-d371-4384-9007-70c60563fb0b.4848.1_5MB mtime 2019-11-20 20:49:11.000000, size 4194304
[root@stor14 build]# bin/rados -c ceph.conf -p default.rgw.buckets.data stat e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_.lWnArC6BcKPkRjbGf-B0q23BSpxjql1_1
default.rgw.buckets.data/e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_.lWnArC6BcKPkRjbGf-B0q23BSpxjql1_1 mtime 2019-11-20 20:49:11.000000, size 1048576
```
可以看到，head obj size=4194304，也即一个stripe 大小，加上tail obj size=1048576，正好等于RGW raw obj 的大小5242880。
我们可以在head 中可以list 元数据
```
[root@stor14 build]# bin/rados -c ceph.conf listxattr e34456f0-d371-4384-9007-70c60563fb0b.4848.1_5MB -p default.rgw.buckets.data
user.rgw.acl
user.rgw.content_type
user.rgw.etag
user.rgw.idtag
user.rgw.manifest # manifest 保存了RGW 对象的layout，可以看作是RGW 对象到RADOS 对象的映射关系图。
user.rgw.pg_ver
user.rgw.source_zone
user.rgw.storage_class
user.rgw.tail_tag
user.rgw.x-amz-content-sha256
user.rgw.x-amz-date
user.rgw.x-amz-meta-s3cmd-attrs
``` 
看下manifest
```
[root@stor14 build]# bin/rados -c ceph.conf getxattr e34456f0-d371-4384-9007-70c60563fb0b.4848.1_5MB -p default.rgw.buckets.data user.rgw.manifest > /tmp/m
```
decode 为可读：
```
[root@stor14 build]# bin/ceph-dencoder type RGWObjManifest import /tmp/m decode dump_json
{
    "objs": [],
    "obj_size": 5242880,
    "explicit_objs": "false",
    "head_size": 4194304, # head 大小
    "max_head_size": 4194304,
    "prefix": ".lWnArC6BcKPkRjbGf-B0q23BSpxjql1_", # 可以在tail obj 上看到，关联各tail objects
    "rules": [
        {
            "key": 0,
            "val": {
                "start_part_num": 0,
                "start_ofs": 4194304,
                "part_size": 0,
                "stripe_max_size": 4194304,
                "override_prefix": ""
            }
        }
    ],
    "tail_instance": "",
    "tail_placement": {
        "bucket": {
            "name": "test2",
            "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1", 
            "bucket_id": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
            "tenant": "",
            "explicit_placement": {
                "data_pool": "",
                "data_extra_pool": "",
                "index_pool": ""
            }
        },
        "placement_rule": "default-placement"
    },
    "begin_iter": {
        "part_ofs": 0,
        "stripe_ofs": 0,
        "ofs": 0,
        "stripe_size": 4194304,
        "cur_part_id": 0,
        "cur_stripe": 0,
        "cur_override_prefix": "",
        "location": {
            "placement_rule": "default-placement",
            "obj": {
                "bucket": {
                    "name": "test2",
                    "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
                    "bucket_id": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
                    "tenant": "",
                    "explicit_placement": {
                        "data_pool": "",
                        "data_extra_pool": "",
                        "index_pool": ""
                    }
                },
                "key": {
                    "name": "5MB",
                    "instance": "",
                    "ns": ""
                }
            },
            "raw_obj": {
                "pool": "",
                "oid": "",
                "loc": ""
            },
            "is_raw": false
        }
    },
    "end_iter": {
        "part_ofs": 4194304,
        "stripe_ofs": 4194304,
        "ofs": 5242880,
        "stripe_size": 1048576,
        "cur_part_id": 0,
        "cur_stripe": 1,
        "cur_override_prefix": "",
        "location": {
            "placement_rule": "default-placement",
            "obj": {
                "bucket": {
                    "name": "test2",
                    "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
                    "bucket_id": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
                    "tenant": "",
                    "explicit_placement": {
                        "data_pool": "",
                        "data_extra_pool": "",
                        "index_pool": ""
                    }
                },
                "key": {
                    "name": ".lWnArC6BcKPkRjbGf-B0q23BSpxjql1_1", # 当前shadow tail obj的key：manifest.prefix+stripe_id
                    "instance": "",
                    "ns": "shadow"  # tail 类型为shadow
                }
            },
            "raw_obj": {
                "pool": "",
                "oid": "",
                "loc": ""
            },
            "is_raw": false
        }
    }
}
```
manifest 中begin_iter 和end_iter的ofset 值及cur_part_id、cur_stripe_id，这些数据决定了RGW 在RADOS层的数据布局：当前5MB的RGW obj，不分part，分为2个stripe，第一个stripe 4194304（4MB），剩余1048576在第二个stripe，结束ofs为5242880。rules 中start_part_num=0 表示当前未分part。
看一个分part 上传的manifest:
```
root@stor14 build]# bin/rados -c ceph.conf getxattr e34456f0-d371-4384-9007-70c60563fb0b.4848.1_20MB -p default.rgw.buckets.data user.rgw.manifest > /tmp/m
[root@stor14 build]# bin/ceph-dencoder type RGWObjManifest import /tmp/m decode dump_json
{
    "objs": [],
    "obj_size": 20971520,
    "explicit_objs": "false",
    "head_size": 0,
    "max_head_size": 0,
    "prefix": "20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx",
    "rules": [
        {
            "key": 0,
            "val": {
                "start_part_num": 1, # "When uploading a part, in addition to the upload ID, you must specify a part number. You can choose any part number between 1 and 10,000. A part number uniquely identifies a part and its position in the object you are uploading. "
                "start_ofs": 0,
                "part_size": 15728640,
                "stripe_max_size": 4194304,
                "override_prefix": ""
            }
        },
        {
            "key": 15728640,
            "val": {
                "start_part_num": 2,
                "start_ofs": 15728640,
                "part_size": 5242880,
                "stripe_max_size": 4194304,
                "override_prefix": ""
            }
        }
    ],
    "tail_instance": "",
    "tail_placement": {
        "bucket": {
            "name": "test2",
            "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
            "bucket_id": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
            "tenant": "",
            "explicit_placement": {
                "data_pool": "",
                "data_extra_pool": "",
                "index_pool": ""
            }
        },
        "placement_rule": "default-placement"
    },
    "begin_iter": {
        "part_ofs": 0,
        "stripe_ofs": 0,
        "ofs": 0,
        "stripe_size": 4194304,
        "cur_part_id": 1,
        "cur_stripe": 0,
        "cur_override_prefix": "",
        "location": {
            "placement_rule": "default-placement",
            "obj": {
                "bucket": {
                    "name": "test2",
                    "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
                    "bucket_id": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
                    "tenant": "",
                    "explicit_placement": {
                        "data_pool": "",
                        "data_extra_pool": "",
                        "index_pool": ""
                    }
                },
                "key": {
                    "name": "20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1", # first part key
                    "instance": "",
                    "ns": "multipart"  # tail 类型为multipart
                }
            },
            "raw_obj": {
                "pool": "",
                "oid": "",
                "loc": ""
            },
            "is_raw": false
        }
    },
    "end_iter": {
        "part_ofs": 20971520,
        "stripe_ofs": 20971520,
        "ofs": 20971520,
        "stripe_size": 4194304,
        "cur_part_id": 3, ###??
        "cur_stripe": 0,
        "cur_override_prefix": "",
        "location": {
            "placement_rule": "default-placement",
            "obj": {
                "bucket": {
                    "name": "test2",
                    "marker": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
                    "bucket_id": "e34456f0-d371-4384-9007-70c60563fb0b.4848.1",
                    "tenant": "",
                    "explicit_placement": {
                        "data_pool": "",
                        "data_extra_pool": "",
                        "index_pool": ""
                    }
                },
                "key": {
                    "name": "20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.3", # last part key ,3 ??
                    "instance": "",
                    "ns": "multipart" # tail 类型为multipart
                }
            },
            "raw_obj": {
                "pool": "",
                "oid": "",
                "loc": ""
            },
            "is_raw": false
        }
    }
}
```
20MB大小的对象分为2个part 上传。观察manifest.rules，当前
key=0 part，也即第一个part，其start_part_num=1，part_size=15728640；
key=1 part，也即第二个part，其start_part_num=2，part_size=5242880，也即剩余大小；
AWS S3 要求multipart 时part number需要在1-1000间，不可以使用0。
 
**ACL** 
```
[root@stor14 build]# bin/rados -c ceph.conf getxattr e34456f0-d371-4384-9007-70c60563fb0b.4848.1_5MB -p default.rgw.buckets.data user.rgw.acl > /tmp/m
[root@stor14 build]# bin/ceph-dencoder type RGWAccessControlPolicy import /tmp/m decode dump_json
{
    "acl": {
        "acl_user_map": [
            {
                "user": "user1",
                "acl": 15
            }
        ],
        "acl_group_map": [],
        "grant_map": [
            {
                "id": "user1",
                "grant": {
                    "type": {
                        "type": 0
                    },
                    "id": "user1",
                    "email": "",
                    "permission": {
                        "flags": 15
                    },
                    "name": "user1",
                    "group": 0,
                    "url_spec": ""
                }
            }
        ]
    },
    "owner": {
        "id": "user1",
        "display_name": "user1"
    }
}
``` 
如果RGW 对象大小超过15MB, 利用s3cmd 上传会自动分段上传，也即multipart 上传。
```
[root@stor14 build]# s3cmd put ./20MB s3://test2
upload: './20MB' -> 's3://test2/20MB'  [part 1 of 2, 15MB] [1 of 1]
 15728640 of 15728640   100% in    1s    11.65 MB/s  done
upload: './20MB' -> 's3://test2/20MB'  [part 2 of 2, 5MB] [1 of 1]
 5242880 of 5242880   100% in    0s     8.21 MB/s  done

[root@stor14 build]# bin/rados -c ceph.conf ls -p default.rgw.buckets.data
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_2
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_jcd1
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2_1
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_1
e34456f0-d371-4384-9007-70c60563fb0b.4281.15_dcj2
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_20MB
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_3
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2
e34456f0-d371-4384-9007-70c60563fb0b.4281.15_dcj1
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_4MB
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_5MB
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_.lWnArC6BcKPkRjbGf-B0q23BSpxjql1_1
 
[root@stor14 build]# bin/rados -c ceph.conf ls -p default.rgw.buckets.data|grep 20MB
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_2
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2_1
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_1
e34456f0-d371-4384-9007-70c60563fb0b.4848.1_20MB
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_3
e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2
``` 
这个20MB 的RGW 对象分成了7 个RADOS 对象，其中
- head object：e34456f0-d371-4384-9007-70c60563fb0b.4848.1_20MB
- tail objects：
  - multipart obj：e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1
    - shadow objs：e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_1,
                   e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_2,
                   e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_3
  - multipart obj：e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2
    - shadow objs：e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2_1

可以看到，multipart 上传的对象会有一个head，默认每个part 15M，每个part 的第一个tail 为multipart obj，剩余都为shadow obj。
查看个rados 的大小：
```
[root@stor14 build]# bin/rados -c ceph.conf -p default.rgw.buckets.data stat e34456f0-d371-4384-9007-70c60563fb0b.4848.1_20MB
default.rgw.buckets.data/e34456f0-d371-4384-9007-70c60563fb0b.4848.1_20MB mtime 2019-11-20 20:58:01.000000, size 0
head obj size 为0，multipart 情况head 只存元数据xattr
[root@stor14 build]# bin/rados -c ceph.conf -p default.rgw.buckets.data stat e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1
default.rgw.buckets.data/e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1 mtime 2019-11-20 20:58:00.000000, size 4194304
[root@stor14 build]# bin/rados -c ceph.conf -p default.rgw.buckets.data stat e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_1
default.rgw.buckets.data/e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_1 mtime 2019-11-20 20:58:00.000000, size 4194304
[root@stor14 build]# bin/rados -c ceph.conf -p default.rgw.buckets.data stat e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_2
default.rgw.buckets.data/e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_2 mtime 2019-11-20 20:58:00.000000, size 4194304
[root@stor14 build]# bin/rados -c ceph.conf -p default.rgw.buckets.data stat e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_3
default.rgw.buckets.data/e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_3 mtime 2019-11-20 20:58:00.000000, size 3145728
[root@stor14 build]# bin/rados -c ceph.conf -p default.rgw.buckets.data stat e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2
default.rgw.buckets.data/e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2 mtime 2019-11-20 20:58:01.000000, size 4194304
[root@stor14 build]# bin/rados -c ceph.conf -p default.rgw.buckets.data stat e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2_1
default.rgw.buckets.data/e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2_1 mtime 2019-11-20 20:58:01.000000, size 1048576
```
0 + 4194304 + 4194304 + 4194304 + 3145728 + 4194304 + 1048576 = 20971520，正好是RGW obj "20MB"的大小。
一般的tail obj（shadow obj）中没有元数据信息，不过multipart obj 会有一些元数据信息：
```
[root@stor14 build]# bin/rados -c ceph.conf listxattr e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_2 -p default.rgw.buckets.data

[root@stor14 build]# bin/rados -c ceph.conf listxattr e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1 -p default.rgw.buckets.data
user.rgw.acl
user.rgw.etag
user.rgw.pg_ver
user.rgw.source_zone
user.rgw.x-amz-content-sha256
user.rgw.x-amz-date
```
可以看到没有manifest。
我们可以从multipart tail obj 中拿到和从head obj中一样的acl 元数据。
```
[root@stor14 build]# bin/rados -c ceph.conf getxattr e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.2 -p default.rgw.buckets.data user.rgw.acl > /tmp/m
[root@stor14 build]# bin/ceph-dencoder type RGWAccessControlPolicy import /tmp/m decode dump_json
{
    "acl": {
        "acl_user_map": [
            {
                "user": "user1",
                "acl": 15
            }
        ],
        "acl_group_map": [],
        "grant_map": [
            {
                "id": "user1",
                "grant": {
                    "type": {
                        "type": 0
                    },
                    "id": "user1",
                    "email": "",
                    "permission": {
                        "flags": 15
                    },
                    "name": "user1",
                    "group": 0,
                    "url_spec": ""
                }
            }
        ]
    },
    "owner": {
        "id": "user1",
        "display_name": "user1"
    }
}
```
在未开启versioning的情况下，RGW obj 对应的RADOS obj 的命名规则为

- head：<marker>__<rgw_obj_name>，如 e34456f0-d371-4384-9007-70c60563fb0b.4848.1_20MB

- （multipart上传）mulpart tail: <marker>__multipart_<rgw_obj_name>.<manifest_prefix>.<part_id>，如 e34456f0-d371-4384-9007-70c60563fb0b.4848.1__multipart_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1
- shadow tail：
    - multipart 上传：<marker>__shadow_<rgw_obj_name>.<manifest_prefix>.<part_id>_<stripe_id>，如 e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_20MB.2~PDSpB4F7hIgKhjVwf6NqmJtBuKxm-mx.1_3
    - 非multipart 上传：<marker>__shadow_.<manifest_prefix>_<stripe_id>，如 e34456f0-d371-4384-9007-70c60563fb0b.4848.1__shadow_.lWnArC6BcKPkRjbGf-B0q23BSpxjql1_1
    
其中marker 就是bucket stats 中看到的marker，也是bucket id。

在开启versioning 之后，会增加一个object_instance_id，这里暂不做详细讨论。

# References
- https://docs.ceph.com/docs/master/radosgw/layout/
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html
- https://www.authonet.com/authentication.php
- https://docs.aws.amazon.com/AmazonS3/latest/dev/mpuoverview.html
