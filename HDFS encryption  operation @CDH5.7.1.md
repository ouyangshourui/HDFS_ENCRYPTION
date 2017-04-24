

# CDH KMS 测试

## 0、用户说明


- [x] lidachao1用户是key admin user
- [x] hdfs 用 户是 hdfs super user
- [x] wumei10 、 ganjianling 是HDFS普通用户

##  1、创建keytab
安装下面的办法创建keytab
```
addprinc -randkey ourui
xst -norandkey -k ourui.keytab ourui
```
##  2、到key admin 用户创建给wumei10创建 key


```
kinit -kt lidachao1.keytab   lidachao1
hadoop key create wumei10_key2
```
结果如下：

```
[root@**rcw1 ~]# kinit -kt lidachao1.keytab   lidachao1
[root@**rcw1 ~]# hadoop key create wumei10_key2
wumei10_key2 has been successfully created with options Options{cipher='AES/CTR/NoPadding', bitLength=128, description='null', attributes=null}.
org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider@6221a451 has been updated.
```

##  3、到hdfs用户给wumei10 创建目录并赋权、创建zone


```
kinit  -kt hdfs.keytab  hdfs
hadoop  fs -mkdir /tmp/wumei10_kms4test
hadoop  fs -chown wumei10:idc_analysis_group /tmp/wumei10_kms4test
hdfs crypto -createZone -keyName wumei10_key2 -path /tmp/wumei10_kms4test
```
结果如下

```
[root@**rcw1 ~]# kinit  -kt hdfs.keytab  hdfs
[root@**rcw1 ~]# hadoop  fs -mkdir /tmp/wumei10_kms4test
[root@**rcw1 ~]# hadoop  fs -chown wumei10:idc_analysis_group /tmp/wumei10_kms4test
[root@**rcw1 ~]# hdfs crypto -createZone -keyName wumei10_key2 -path /tmp/wumei10_kms4test
Added encryption zone /tmp/wumei10_kms4test
```


##  4、到wumei10用户上传文件、并测试可读性

```
kinit -kt wumei10.keytab wumei10
echo "Hello World" > /tmp/helloWorld.txt
hadoop fs -put /tmp/helloWorld.txt /tmp/wumei10_kms4test
hadoop fs -cat /tmp/wumei10_kms4test/helloWorld.txt
rm /tmp/helloWorld.txt
```
结果如下：

```
[root@**rcw1 ~]# hadoop fs -put /tmp/helloWorld.txt /tmp/wumei10_kms4test
17/04/11 18:18:45 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://**rcn1.lfidc**.cn:16000/kms/v1/] threw an IOException [User [wumei10] is not authorized to perform [DECRYPT_EEK] on key with ACL name [wumei10_key2]!!]!!
17/04/11 18:18:45 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://**rcn2.lfidc**.cn:16000/kms/v1/] threw an IOException [User [wumei10] is not authorized to perform [DECRYPT_EEK] on key with ACL name [wumei10_key2]!!]!!
17/04/11 18:18:45 WARN kms.LoadBalancingKMSClientProvider: Aborting since the Request has failed with all KMS providers in the group. !!
put: User [wumei10] is not authorized to perform [DECRYPT_EEK] on key with ACL name [wumei10_key2]!!
17/04/11 18:18:45 ERROR hdfs.DFSClient: Failed to close inode 1404823
```

从结果看2 wumei10对wumei10_key2没有 DECRYPT_EEK权限，这时候就设计到可以的白名单设置了。下面我们到kms-acl.xml文件里面配置该key的权限

```
<property>
    <name>key.acl.wumei10_key2.DECRYPT_EEK</name>
    <value>wumei10</value>
    <description>
      ACL for decryptEncryptedKey operations.
    </description>
  </property>
```
滚动重启KMS server，
我们继续写入数据

```
[root@**rcw1 ~]# hadoop fs -put /tmp/helloWorld.txt /tmp/wumei10_kms4test
[root@**rcw1 ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: wumei10@LFDC.**-GROUP.NET

Valid starting       Expires              Service principal
04/11/2017 18:18:18  04/12/2017 18:18:18  krbtgt/LFDC.**-GROUP.NET@LFDC.**-GROUP.NET
        renew until 04/18/2017 18:18:18
```

数据写入成功，测试读数据

```
[root@**rcw1 ~]# hadoop fs -cat /tmp/wumei10_kms4test/helloWorld.txt                    
Hello World
```
读数据成功。

[link](http://note.youdao.com/noteshare?id=7ac0ffe7860ae692a56c549d9419f71b)

##  5、到gangjianling用户读取上传数据


```
[root@**rcw1 ~]# kinit -kt ganjianling.keytab ganjianling
[root@**rcw1 ~]# hadoop fs -cat /tmp/wumei10_kms4test/helloWorld.txt 
17/04/11 18:40:10 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://**rcn1.lfidc**.cn:16000/kms/v1/] threw an IOException [User [ganjianling] is not authorized to perform [DECRYPT_EEK] on key with ACL name [wumei10_key2]!!]!!
17/04/11 18:40:10 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://**rcn2.lfidc**.cn:16000/kms/v1/] threw an IOException [User [ganjianling] is not authorized to perform [DECRYPT_EEK] on key with ACL name [wumei10_key2]!!]!!
17/04/11 18:40:10 WARN kms.LoadBalancingKMSClientProvider: Aborting since the Request has failed with all KMS providers in the group. !!
cat: User [ganjianling] is not authorized to perform [DECRYPT_EEK] on key with ACL name [wumei10_key2]!!
```


##  6、到hdfs用户读取上传数据


```
[root@**rcw1 ~]# kinit -kt hdfs.keytab hdfs
[root@**rcw1 ~]# hadoop fs -cat /tmp/wumei10_kms4test/helloWorld.txt 
17/04/11 18:40:31 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://**rcn1.lfidc**.cn:16000/kms/v1/] threw an IOException [User:hdfs not allowed to do 'DECRYPT_EEK' on 'wumei10_key2']!!
17/04/11 18:40:31 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://**rcn2.lfidc**.cn:16000/kms/v1/] threw an IOException [User:hdfs not allowed to do 'DECRYPT_EEK' on 'wumei10_key2']!!
17/04/11 18:40:31 WARN kms.LoadBalancingKMSClientProvider: Aborting since the Request has failed with all KMS providers in the group. !!
cat: User:hdfs not allowed to do 'DECRYPT_EEK' on 'wumei10_key2'
```

## 7、到暴力磁盘读取文件
获取/tmp/wumei10_kms4test/helloWorld.txt的block分区

```
[root@**rcw1 ~]# hdfs fsck /tmp/wumei10_kms4test/helloWorld.txt -files -blocks -locations
Connecting to namenode via http://**rcm2.lfidc**.cn:50070
FSCK started by wumei10 (auth:KERBEROS_SSL) from /**.192.22 for path /tmp/wumei10_kms4test/helloWorld.txt at Tue Apr 11 20:08:13 CST 2017
/tmp/wumei10_kms4test/anaconda-ks.cfg 1794 bytes, 1 block(s): OK
0. BP-1364822025-**.192.19-1483423637421:blk_1074038523_297830 len=1794 Live_repl=3 [DatanodeInfoWithStorage[**.192.44:1004,DS-e8d0aba4-e9d7-4cd2-87bb-82b361a7a91a,DISK], DatanodeInfoWithStorage[**.192.35:1004,DS-2929b784-382e-4c9b-91cc-609d3e7e0bfd,DISK], DatanodeInfoWithStorage[**.192.43:1004,DS-bd15f7a5-ac53-484d-ae55-5273ef1aa6c5,DISK]]
Status: HEALTHY
Total size: 1794 B
Total dirs: 0
Total files: 1
Total symlinks: 0
Total blocks (validated): 1 (avg. block size 1794 B)
Minimally replicated blocks: 1 (100.0 %)
Over-replicated blocks: 0 (0.0 %)
Under-replicated blocks: 0 (0.0 %)
Mis-replicated blocks: 0 (0.0 %)
Default replication factor: 3
Average block replication: 3.0
Corrupt blocks: 0
Missing replicas: 0 (0.0 %)
Number of data-nodes: 25
Number of racks: 1
FSCK ended at Tue Apr 11 20:08:13 CST 2017 in 1 milliseconds

The filesystem under path '/tmp/wumei10_kms4test/helloWorld.txt' is HEALTHY
[root@**rcw1 ~]# ssh **.192.44
Last login: Wed Apr 5 10:30:44 2017 from **rcm1.lfidc**.cn
[root@**rd21 ~]# find / -name blk_1074038523*
/data/d8/dn/current/BP-1364822025-**.192.19-1483423637421/current/finalized/subdir4/subdir134/blk_1074038523_297830.meta
/data/d8/dn/current/BP-1364822025-**.192.19-1483423637421/current/finalized/subdir4/subdir134/blk_1074038523
find: ‘/proc/26759’: No such file or directory
[root@**rd21 ~]# cat /data/d8/dn/current/BP-1364822025-**.192.19-1483423637421/current/finalized/subdir4/subdir134/blk_1074038523
-ڍ{yvF?[lUuc
4k[
HB(;9&lWKNX#骇6׸(\DrB}Ͽxח=AzjޑT!NHt=ZJVIڕa4E?_z�` 'μXX<_Ӆ/0P9ٹ&/@VH焺d4hZA
```

# 7、拷贝文件到Encryption Zone

可以将存在的数据拷贝到encryption zone，在HDFS内部可以可以使用DistCp命令
测试过程

- hdfs上面普通的数据目录/data/big_data/staff/wumei10
- 加密去区/tmp/wumei10_kms4test


```
hadoop distcp /data/big_data/staff/wumei10 /tmp/wumei10_kms4test -skipcrccheck -update



[root@**rcw1 ~]# hadoop distcp  -skipcrccheck -update /data/big_data/staff/wumei10 /tmp/wumei10_kms4test 
17/04/12 15:07:21 INFO tools.DistCp: Input Options: DistCpOptions{atomicCommit=false, syncFolder=true, deleteMissing=false, ignoreFailures=false, maxMaps=20, sslConfigurationFile='null', copyStrategy='uniformsize', sourceFileListing=null, sourcePaths=[/data/big_data/staff/wumei10], targetPath=/tmp/wumei10_kms4test, targetPathExists=true, preserveRawXattrs=false, filtersFile='null'}
17/04/12 15:07:21 INFO hdfs.DFSClient: Created HDFS_DELEGATION_TOKEN token 3212 for wumei10 on ha-hdfs:nameservice1
17/04/12 15:07:22 INFO security.TokenCache: Got dt for hdfs://nameservice1; Kind: HDFS_DELEGATION_TOKEN, Service: ha-hdfs:nameservice1, Ident: (HDFS_DELEGATION_TOKEN token 3212 for wumei10)
17/04/12 15:07:22 WARN token.Token: Cannot find class for token kind kms-dt
17/04/12 15:07:22 INFO security.TokenCache: Got dt for hdfs://nameservice1; Kind: kms-dt, Service: **.192.21:16000, Ident: 00 07 77 75 6d 65 69 31 30 04 79 61 72 6e 00 8a 01 5b 60 fc f3 bc 8a 01 5b 85 09 77 bc 1f 10
17/04/12 15:07:22 INFO tools.SimpleCopyListing: Paths (files+dirs) cnt = 509; dirCnt = 73
17/04/12 15:07:22 INFO tools.SimpleCopyListing: Build file listing completed.
17/04/12 15:07:22 INFO Configuration.deprecation: io.sort.mb is deprecated. Instead, use mapreduce.task.io.sort.mb
17/04/12 15:07:22 INFO Configuration.deprecation: io.sort.factor is deprecated. Instead, use mapreduce.task.io.sort.factor
17/04/12 15:07:22 INFO tools.DistCp: Number of paths in the copy list: 509
17/04/12 15:07:22 INFO tools.DistCp: Number of paths in the copy list: 509
17/04/12 15:07:22 WARN token.Token: Cannot find class for token kind kms-dt
17/04/12 15:07:22 INFO security.TokenCache: Got dt for hdfs://nameservice1; Kind: kms-dt, Service: **.192.20:16000, Ident: 00 07 77 75 6d 65 69 31 30 04 79 61 72 6e 00 8a 01 5b 60 fc f6 c7 8a 01 5b 85 09 7a c7 20 0e
17/04/12 15:07:22 INFO mapreduce.JobSubmitter: number of splits:22
17/04/12 15:07:23 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1491908827433_0004
17/04/12 15:07:23 WARN token.Token: Cannot find class for token kind kms-dt
17/04/12 15:07:23 WARN token.Token: Cannot find class for token kind kms-dt
Kind: kms-dt, Service: **.192.20:16000, Ident: 00 07 77 75 6d 65 69 31 30 04 79 61 72 6e 00 8a 01 5b 60 fc f6 c7 8a 01 5b 85 09 7a c7 20 0e
17/04/12 15:07:23 INFO mapreduce.JobSubmitter: Kind: HDFS_DELEGATION_TOKEN, Service: ha-hdfs:nameservice1, Ident: (HDFS_DELEGATION_TOKEN token 3212 for wumei10)
17/04/12 15:07:23 WARN token.Token: Cannot find class for token kind kms-dt
17/04/12 15:07:23 WARN token.Token: Cannot find class for token kind kms-dt
Kind: kms-dt, Service: **.192.21:16000, Ident: 00 07 77 75 6d 65 69 31 30 04 79 61 72 6e 00 8a 01 5b 60 fc f3 bc 8a 01 5b 85 09 77 bc 1f 10
17/04/12 15:07:23 INFO impl.YarnClientImpl: Submitted application application_1491908827433_0004
17/04/12 15:07:23 INFO mapreduce.Job: The url to track the job: http://**rcm2.lfidc**.cn:8088/proxy/application_1491908827433_0004/
17/04/12 15:07:23 INFO tools.DistCp: DistCp job-id: job_1491908827433_0004
17/04/12 15:07:23 INFO mapreduce.Job: Running job: job_1491908827433_0004
17/04/12 15:07:30 INFO mapreduce.Job: Job job_1491908827433_0004 running in uber mode : false
17/04/12 15:07:30 INFO mapreduce.Job:  map 0% reduce 0%
17/04/12 15:07:42 INFO mapreduce.Job:  map 5% reduce 0%
17/04/12 15:07:43 INFO mapreduce.Job:  map 17% reduce 0%
17/04/12 15:07:44 INFO mapreduce.Job:  map 23% reduce 0%
17/04/12 15:07:45 INFO mapreduce.Job:  map 24% reduce 0%
17/04/12 15:07:49 INFO mapreduce.Job:  map 56% reduce 0%
17/04/12 15:07:50 INFO mapreduce.Job:  map 87% reduce 0%
17/04/12 15:07:51 INFO mapreduce.Job:  map 95% reduce 0%
17/04/12 15:07:52 INFO mapreduce.Job:  map 100% reduce 0%
17/04/12 15:07:52 INFO mapreduce.Job: Job job_1491908827433_0004 completed successfully
17/04/12 15:07:52 INFO mapreduce.Job: Counters: 35
        File System Counters
                FILE: Number of bytes read=0
                FILE: Number of bytes written=2807558
                FILE: Number of read operations=0
                FILE: Number of large read operations=0
                FILE: Number of write operations=0
                HDFS: Number of bytes read=39835898
                HDFS: Number of bytes written=39727874
                HDFS: Number of read operations=4241
                HDFS: Number of large read operations=0
                HDFS: Number of write operations=1016
        Job Counters 
                Launched map tasks=22
                Other local map tasks=22
                Total time spent by all maps in occupied slots (ms)=345263
                Total time spent by all reduces in occupied slots (ms)=0
                Total time spent by all map tasks (ms)=345263
                Total vcore-seconds taken by all map tasks=345263
                Total megabyte-seconds taken by all map tasks=353549312
        Map-Reduce Framework
                Map input records=509
                Map output records=1
                Input split bytes=2574
                Spilled Records=0
                Failed Shuffles=0
                Merged Map outputs=0
                GC time elapsed (ms)=1707
                CPU time spent (ms)=90730
                Physical memory (bytes) snapshot=7748816896
                Virtual memory (bytes) snapshot=63901360128
                Total committed heap usage (bytes)=13378256896
        File Input Format Counters 
                Bytes Read=105520
        File Output Format Counters 
                Bytes Written=70
        org.apache.hadoop.tools.mapred.CopyMapper$Counter
                BYTESCOPIED=39727804
                BYTESEXPECTED=39727804
                BYTESSKIPPED=1794
                COPY=508
                SKIP=1
```
检查数据结果

```
[root@**rcw1 ~]# hadoop fs -ls  /data/big_data/staff/wumei10 /tmp/wumei10_kms4test                                     
Found 12 items
-rw-r--r--   3 wumei10 idc_analysis_group          0 2017-04-12 14:59 /data/big_data/staff/wumei10/.autorelabel
-rw-r--r--   3 wumei10 idc_analysis_group      81239 2017-04-12 14:59 /data/big_data/staff/wumei10/.readahead
-rw-r--r--   3 wumei10 idc_analysis_group       1794 2017-04-12 14:59 /data/big_data/staff/wumei10/anaconda-ks.cfg
drwxr-xr-x   - wumei10 idc_analysis_group          0 2017-04-12 14:59 /data/big_data/staff/wumei10/app
drwxr-xr-x   - wumei10 idc_analysis_group          0 2017-04-12 15:00 /data/big_data/staff/wumei10/bin
-rw-r--r--   3 wumei10 idc_analysis_group        586 2017-04-12 14:59 /data/big_data/staff/wumei10/ganjianling.keytab
-rw-r--r--   3 wumei10 idc_analysis_group       1066 2017-04-12 14:59 /data/big_data/staff/wumei10/hdfs.keytab
-rw-r--r--   3 wumei10 idc_analysis_group       1346 2017-04-12 14:59 /data/big_data/staff/wumei10/hive.keytab
-rw-r--r--   3 wumei10 idc_analysis_group        570 2017-04-12 14:59 /data/big_data/staff/wumei10/lidachao1.keytab
-rw-r--r--   3 wumei10 idc_analysis_group        538 2017-04-12 14:59 /data/big_data/staff/wumei10/ourui.keytab
drwxr-xr-x   - wumei10 idc_analysis_group          0 2017-04-12 14:59 /data/big_data/staff/wumei10/root
-rw-r--r--   3 wumei10 idc_analysis_group        554 2017-04-12 14:59 /data/big_data/staff/wumei10/wumei10.keytab
Found 14 items
drwxrwxrwt   - hdfs    supergroup                  0 2017-04-11 18:11 /tmp/wumei10_kms4test/.Trash
-rw-r--r--   3 wumei10 idc_analysis_group          0 2017-04-12 15:07 /tmp/wumei10_kms4test/.autorelabel
-rw-r--r--   3 wumei10 idc_analysis_group      81239 2017-04-12 15:07 /tmp/wumei10_kms4test/.readahead
-rw-r--r--   3 wumei10 idc_analysis_group       1794 2017-04-11 20:06 /tmp/wumei10_kms4test/anaconda-ks.cfg
drwxr-xr-x   - wumei10 idc_analysis_group          0 2017-04-12 15:07 /tmp/wumei10_kms4test/app
drwxr-xr-x   - wumei10 idc_analysis_group          0 2017-04-12 15:07 /tmp/wumei10_kms4test/bin
-rw-r--r--   3 wumei10 idc_analysis_group        586 2017-04-12 15:07 /tmp/wumei10_kms4test/ganjianling.keytab
-rw-r--r--   3 wumei10 idc_analysis_group       1066 2017-04-12 15:07 /tmp/wumei10_kms4test/hdfs.keytab
-rw-r--r--   3 wumei10 idc_analysis_group         12 2017-04-11 18:30 /tmp/wumei10_kms4test/helloWorld.txt
-rw-r--r--   3 wumei10 idc_analysis_group       1346 2017-04-12 15:07 /tmp/wumei10_kms4test/hive.keytab
-rw-r--r--   3 wumei10 idc_analysis_group        570 2017-04-12 15:07 /tmp/wumei10_kms4test/lidachao1.keytab
-rw-r--r--   3 wumei10 idc_analysis_group        538 2017-04-12 15:07 /tmp/wumei10_kms4test/ourui.keytab
drwxr-xr-x   - wumei10 idc_analysis_group          0 2017-04-12 15:07 /tmp/wumei10_kms4test/root
-rw-r--r--   3 wumei10 idc_analysis_group        554 2017-04-12 15:07 /tmp/wumei10_kms4test/wumei10.keytab
```


# 8 spark 读取encryption zone数据权限验证

```
scala> sc.textFile("/tmp/wumei10_kms4test/anaconda-ks.cfg").collect()
17/04/12 18:49:24 WARN token.Token: Cannot find class for token kind kms-dt
[Stage 0:>                                                          (0 + 2) / 2]17/04/12 18:49:33 WARN scheduler.TaskSetManager: Lost task 0.0 in stage 0.0 (TID 0, **rd12.lfidc**.cn): java.io.IOException: org.apache.hadoop.security.authentication.client.AuthenticationException: GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)
        at org.apache.hadoop.crypto.key.kms.KMSClientProvider.createConnection(KMSClientProvider.java:491)
        at org.apache.hadoop.crypto.key.kms.KMSClientProvider.decryptEncryptedKey(KMSClientProvider.java:771)
        at org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider$3.call(LoadBalancingKMSClientProvider.java:185)
        at org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider$3.call(LoadBalancingKMSClientProvider.java:181)
        at org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider.doOp(LoadBalancingKMSClientProvider.java:94)
        at org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider.decryptEncryptedKey(LoadBalancingKMSClientProvider.java:181)
        at org.apache.hadoop.crypto.key.KeyProviderCryptoExtension.decryptEncryptedKey(KeyProviderCryptoExtension.java:388)
        at org.apache.hadoop.hdfs.DFSClient.decryptEncryptedDataEncryptionKey(DFSClient.java:1420)
        at org.apache.hadoop.hdfs.DFSClient.createWrappedInputStream(DFSClient.java:1490)
        at org.apache.hadoop.hdfs.DistributedFileSystem$3.doCall(DistributedFileSystem.java:311)
        at org.apache.hadoop.hdfs.DistributedFileSystem$3.doCall(DistributedFileSystem.java:305)
        at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
        at org.apache.hadoop.hdfs.DistributedFileSystem.open(DistributedFileSystem.java:305)
        at org.apache.hadoop.fs.FileSystem.open(FileSystem.java:778)
        at org.apache.hadoop.mapred.LineRecordReader.<init>(LineRecordReader.java:109)
        at org.apache.hadoop.mapred.TextInputFormat.getRecordReader(TextInputFormat.java:67)
        at org.apache.spark.rdd.HadoopRDD$$anon$1.<init>(HadoopRDD.scala:237)
        at org.apache.spark.rdd.HadoopRDD.compute(HadoopRDD.scala:208)
        at org.apache.spark.rdd.HadoopRDD.compute(HadoopRDD.scala:101)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:306)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:270)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:306)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:270)
        at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
        at org.apache.spark.scheduler.Task.run(Task.scala:89)
        at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)
Caused by: org.apache.hadoop.security.authentication.client.AuthenticationException: GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)
        at org.apache.hadoop.security.authentication.client.KerberosAuthenticator.doSpnegoSequence(KerberosAuthenticator.java:306)
        at org.apache.hadoop.security.authentication.client.KerberosAuthenticator.authenticate(KerberosAuthenticator.java:196)
        at org.apache.hadoop.security.token.delegation.web.DelegationTokenAuthenticator.authenticate(DelegationTokenAuthenticator.java:127)
        at org.apache.hadoop.security.authentication.client.AuthenticatedURL.openConnection(AuthenticatedURL.java:215)
        at org.apache.hadoop.security.token.delegation.web.DelegationTokenAuthenticatedURL.openConnection(DelegationTokenAuthenticatedURL.java:322)
        at org.apache.hadoop.crypto.key.kms.KMSClientProvider$1.run(KMSClientProvider.java:485)
        at org.apache.hadoop.crypto.key.kms.KMSClientProvider$1.run(KMSClientProvider.java:480)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:422)
        at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1693)
        at org.apache.hadoop.crypto.key.kms.KMSClientProvider.createConnection(KMSClientProvider.java:480)
        ... 29 more
Caused by: GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)
        at sun.security.jgss.krb5.Krb5InitCredential.getInstance(Krb5InitCredential.java:147)
        at sun.security.jgss.krb5.Krb5MechFactory.getCredentialElement(Krb5MechFactory.java:122)
        at sun.security.jgss.krb5.Krb5MechFactory.getMechanismContext(Krb5MechFactory.java:187)
        at sun.security.jgss.GSSManagerImpl.getMechanismContext(GSSManagerImpl.java:224)
        at sun.security.jgss.GSSContextImpl.initSecContext(GSSContextImpl.java:212)
        at sun.security.jgss.GSSContextImpl.initSecContext(GSSContextImpl.java:179)
        at org.apache.hadoop.security.authentication.client.KerberosAuthenticator$1.run(KerberosAuthenticator.java:285)
        at org.apache.hadoop.security.authentication.client.KerberosAuthenticator$1.run(KerberosAuthenticator.java:261)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:422)
        at org.apache.hadoop.security.authentication.client.KerberosAuthenticator.doSpnegoSequence(KerberosAuthenticator.java:261)
        ... 39 more

[Stage 0:>                                                          (0 + 1) / 2]17/04/12 18:49:33 WARN scheduler.TaskSetManager: Lost task 0.1 in stage 0.0 (TID 2, **rd12.lfidc**.cn): 
17/04/12 18:49:33 ERROR scheduler.TaskSetManager: Task 0 in stage 0.0 failed 4 times; aborting job
17/04/12 18:49:33 WARN spark.ExecutorAllocationManager: No stages are running, but numRunningTasks != 0
org.apache.spark.SparkException: Job aborted due to stage failure: Task 0 in stage 0.0 failed 4 times, most recent failure: Lost task 0.3 in stage 0.0 (TID 4, **rd12.lfidc**.cn): 
Driver stacktrace:
        at org.apache.spark.scheduler.DAGScheduler.org$apache$spark$scheduler$DAGScheduler$$failJobAndIndependentStages(DAGScheduler.scala:1431)
        at org.apache.spark.scheduler.DAGScheduler$$anonfun$abortStage$1.apply(DAGScheduler.scala:1419)
        at org.apache.spark.scheduler.DAGScheduler$$anonfun$abortStage$1.apply(DAGScheduler.scala:1418)
        at scala.collection.mutable.ResizableArray$class.foreach(ResizableArray.scala:59)
        at scala.collection.mutable.ArrayBuffer.foreach(ArrayBuffer.scala:47)
        at org.apache.spark.scheduler.DAGScheduler.abortStage(DAGScheduler.scala:1418)
        at org.apache.spark.scheduler.DAGScheduler$$anonfun$handleTaskSetFailed$1.apply(DAGScheduler.scala:799)
        at org.apache.spark.scheduler.DAGScheduler$$anonfun$handleTaskSetFailed$1.apply(DAGScheduler.scala:799)
        at scala.Option.foreach(Option.scala:236)
        at org.apache.spark.scheduler.DAGScheduler.handleTaskSetFailed(DAGScheduler.scala:799)
        at org.apache.spark.scheduler.DAGSchedulerEventProcessLoop.doOnReceive(DAGScheduler.scala:1640)
        at org.apache.spark.scheduler.DAGSchedulerEventProcessLoop.onReceive(DAGScheduler.scala:1599)
        at org.apache.spark.scheduler.DAGSchedulerEventProcessLoop.onReceive(DAGScheduler.scala:1588)
        at org.apache.spark.util.EventLoop$$anon$1.run(EventLoop.scala:48)
        at org.apache.spark.scheduler.DAGScheduler.runJob(DAGScheduler.scala:620)
        at org.apache.spark.SparkContext.runJob(SparkContext.scala:1843)
        at org.apache.spark.SparkContext.runJob(SparkContext.scala:1856)
        at org.apache.spark.SparkContext.runJob(SparkContext.scala:1869)
        at org.apache.spark.SparkContext.runJob(SparkContext.scala:1940)
        at org.apache.spark.rdd.RDD$$anonfun$collect$1.apply(RDD.scala:927)
        at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:150)
        at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:111)
        at org.apache.spark.rdd.RDD.withScope(RDD.scala:316)
        at org.apache.spark.rdd.RDD.collect(RDD.scala:926)
        at $iwC$$iwC$$iwC$$iwC$$iwC$$iwC$$iwC$$iwC.<init>(<console>:28)
        at $iwC$$iwC$$iwC$$iwC$$iwC$$iwC$$iwC.<init>(<console>:33)
        at $iwC$$iwC$$iwC$$iwC$$iwC$$iwC.<init>(<console>:35)
        at $iwC$$iwC$$iwC$$iwC$$iwC.<init>(<console>:37)
        at $iwC$$iwC$$iwC$$iwC.<init>(<console>:39)
        at $iwC$$iwC$$iwC.<init>(<console>:41)
        at $iwC$$iwC.<init>(<console>:43)
        at $iwC.<init>(<console>:45)
        at <init>(<console>:47)
        at .<init>(<console>:51)
        at .<clinit>(<console>)
        at .<init>(<console>:7)
        at .<clinit>(<console>)
        at $print(<console>)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:497)
        at org.apache.spark.repl.SparkIMain$ReadEvalPrint.call(SparkIMain.scala:1045)
        at org.apache.spark.repl.SparkIMain$Request.loadAndRun(SparkIMain.scala:1326)
        at org.apache.spark.repl.SparkIMain.loadAndRunReq$1(SparkIMain.scala:821)
        at org.apache.spark.repl.SparkIMain.interpret(SparkIMain.scala:852)
        at org.apache.spark.repl.SparkIMain.interpret(SparkIMain.scala:800)
        at org.apache.spark.repl.SparkILoop.reallyInterpret$1(SparkILoop.scala:857)
        at org.apache.spark.repl.SparkILoop.interpretStartingWith(SparkILoop.scala:902)
        at org.apache.spark.repl.SparkILoop.command(SparkILoop.scala:814)
        at org.apache.spark.repl.SparkILoop.processLine$1(SparkILoop.scala:657)
        at org.apache.spark.repl.SparkILoop.innerLoop$1(SparkILoop.scala:665)
        at org.apache.spark.repl.SparkILoop.org$apache$spark$repl$SparkILoop$$loop(SparkILoop.scala:670)
        at org.apache.spark.repl.SparkILoop$$anonfun$org$apache$spark$repl$SparkILoop$$process$1.apply$mcZ$sp(SparkILoop.scala:997)
        at org.apache.spark.repl.SparkILoop$$anonfun$org$apache$spark$repl$SparkILoop$$process$1.apply(SparkILoop.scala:945)
        at org.apache.spark.repl.SparkILoop$$anonfun$org$apache$spark$repl$SparkILoop$$process$1.apply(SparkILoop.scala:945)
        at scala.tools.nsc.util.ScalaClassLoader$.savingContextLoader(ScalaClassLoader.scala:135)
        at org.apache.spark.repl.SparkILoop.org$apache$spark$repl$SparkILoop$$process(SparkILoop.scala:945)
        at org.apache.spark.repl.SparkILoop.process(SparkILoop.scala:1064)
        at org.apache.spark.repl.Main$.main(Main.scala:31)
        at org.apache.spark.repl.Main.main(Main.scala)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:497)
        at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:731)
        at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:181)
        at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:206)
        at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:121)
        at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: org.apache.hadoop.security.authorize.AuthorizationException: User:hdfs not allowed to do 'DECRYPT_EEK' on 'wumei10_key2'
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:422)
        at org.apache.hadoop.util.HttpExceptionUtils.validateResponse(HttpExceptionUtils.java:157)
        at org.apache.hadoop.crypto.key.kms.KMSClientProvider.call(KMSClientProvider.java:548)
        at org.apache.hadoop.crypto.key.kms.KMSClientProvider.call(KMSClientProvider.java:506)
        at org.apache.hadoop.crypto.key.kms.KMSClientProvider.decryptEncryptedKey(KMSClientProvider.java:773)
        at org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider$3.call(LoadBalancingKMSClientProvider.java:185)
        at org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider$3.call(LoadBalancingKMSClientProvider.java:181)
        at org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider.doOp(LoadBalancingKMSClientProvider.java:94)
        at org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider.decryptEncryptedKey(LoadBalancingKMSClientProvider.java:181)
        at org.apache.hadoop.crypto.key.KeyProviderCryptoExtension.decryptEncryptedKey(KeyProviderCryptoExtension.java:388)
        at org.apache.hadoop.hdfs.DFSClient.decryptEncryptedDataEncryptionKey(DFSClient.java:1420)
        at org.apache.hadoop.hdfs.DFSClient.createWrappedInputStream(DFSClient.java:1490)
        at org.apache.hadoop.hdfs.DistributedFileSystem$3.doCall(DistributedFileSystem.java:311)
        at org.apache.hadoop.hdfs.DistributedFileSystem$3.doCall(DistributedFileSystem.java:305)
        at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
        at org.apache.hadoop.hdfs.DistributedFileSystem.open(DistributedFileSystem.java:305)
        at org.apache.hadoop.fs.FileSystem.open(FileSystem.java:778)
        at org.apache.hadoop.mapred.LineRecordReader.<init>(LineRecordReader.java:109)
        at org.apache.hadoop.mapred.TextInputFormat.getRecordReader(TextInputFormat.java:67)
        at org.apache.spark.rdd.HadoopRDD$$anon$1.<init>(HadoopRDD.scala:237)
        at org.apache.spark.rdd.HadoopRDD.compute(HadoopRDD.scala:208)
        at org.apache.spark.rdd.HadoopRDD.compute(HadoopRDD.scala:101)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:306)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:270)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:306)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:270)
        at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
        at org.apache.spark.scheduler.Task.run(Task.scala:89)
        at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)


scala>
[root@**rcw1 ~]# kinit -kt wumei10.keytab wumei10
[root@**rcw1 ~]# spark-shell --master yarn       
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel).
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 1.6.0
      /_/

Using Scala version 2.10.5 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_60)
Type in expressions to have them evaluated.
Type :help for more information.
Spark context available as sc (master = yarn-client, app id = application_1491908827433_0010).
SQL context available as sqlContext.

scala> sc.textFile("/tmp/wumei10_kms4test/anaconda-ks.cfg").collect()
17/04/12 18:50:23 WARN token.Token: Cannot find class for token kind kms-dt
res0: Array[String] = Array(#version=DEVEL, # System authorization information, auth --enableshadow --passalgo=sha512, # Use CDROM installation media, cdrom, # Use graphical install, graphical, # Run the Setup Agent on first boot, firstboot --enable, ignoredisk --only-use=sda, # Keyboard layouts, keyboard --vckeymap=us --xlayouts='us', # System language, lang en_US.UTF-8 --addsupport=zh_CN.UTF-8, "", # Network information, network  --bootproto=dhcp --device=eno1 --onboot=off --ipv6=auto, network  --bootproto=dhcp --device=eno2 --onboot=off --ipv6=auto, network  --bootproto=dhcp --device=eno3 --onboot=off --ipv6=auto, network  --bootproto=dhcp --device=eno4 --onboot=off --ipv6=auto, network  --hostname=localhost.localdomain, "", # Root password, rootpw --iscrypted $6$H.qsCwrYBD.ATgZH$lP3...
scala> 
```


# 9 hiveserver2 读取encryption zone数据权限验证

操作过程


```
!connect jdbc:hive2://**.192.20:10000/default;principal=hive/**rcn1.lfidc**.cn@LFDC.**-GROUP.NET
CREATE DATABASE  encryption_test_db  location   '/tmp/wumei10_kms4test/encryption_test_db';
describe database  encryption_test_db;
CREATE ROLE   encryption_all_role; 
GRANT ALL ON DATABASE encryption_test_db TO ROLE  encryption_all_role;
GRANT ROLE  encryption_all_role   TO GROUP idc_analysis_group;
SHOW GRANT ROLE  encryption_all_role ;
GRANT ALL ON URI   '/tmp/wumei10_kms4test/encryption_test_db'   TO ROLE encryption_all_role;
beeline -u jdbc:hive2://**.192.22:10000  -n wumei10   -p  123456  -d org.apache.hive.jdbc.HiveDriver -e "show databases"
drop table pokes;
CREATE TABLE pokes1 (foo INT, bar STRING) row format delimited fields terminated by ','; 
 
```



```
[root@**rcw1 ~]# kinit -kt wumei10.keytab wumei10
[root@**rcw1 ~]# beeline -u jdbc:hive2://**.192.22:10000  -n wumei10   -p  123456  -d org.apache.hive.jdbc.HiveDriver
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=512M; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: Using incremental CMS is deprecated and will likely be removed in a future release
17/04/12 19:26:40 WARN mapreduce.TableMapReduceUtil: The hbase-prefix-tree module jar containing PrefixTreeCodec is not present.  Continuing without it.
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=512M; support was removed in 8.0
Connecting to jdbc:hive2://**.192.22:10000
Connected to: Apache Hive (version 1.1.0-cdh5.7.1)
Driver: Hive JDBC (version 1.1.0-cdh5.7.1)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 1.1.0-cdh5.7.1 by Apache Hive
0: jdbc:hive2://**.192.22:10000> use encryption_test_db;
INFO  : Compiling command(queryId=hive_20170412192626_6d25d854-2bb2-45a0-9dfd-906b4e75a74e): use encryption_test_db
INFO  : Semantic Analysis Completed
INFO  : Returning Hive schema: Schema(fieldSchemas:null, properties:null)
INFO  : Completed compiling command(queryId=hive_20170412192626_6d25d854-2bb2-45a0-9dfd-906b4e75a74e); Time taken: 0.161 seconds
INFO  : Executing command(queryId=hive_20170412192626_6d25d854-2bb2-45a0-9dfd-906b4e75a74e): use encryption_test_db
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hive_20170412192626_6d25d854-2bb2-45a0-9dfd-906b4e75a74e); Time taken: 0.04 seconds
INFO  : OK
No rows affected (0.285 seconds)
0: jdbc:hive2://**.192.22:10000> select * from pokes1;
INFO  : Compiling command(queryId=hive_20170412192626_5a433375-00b2-4b67-8846-c7b97ffbd4a5): select * from pokes1
INFO  : Semantic Analysis Completed
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:pokes1.foo, type:int, comment:null), FieldSchema(name:pokes1.bar, type:string, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20170412192626_5a433375-00b2-4b67-8846-c7b97ffbd4a5); Time taken: 0.22 seconds
INFO  : Executing command(queryId=hive_20170412192626_5a433375-00b2-4b67-8846-c7b97ffbd4a5): select * from pokes1
INFO  : Completed executing command(queryId=hive_20170412192626_5a433375-00b2-4b67-8846-c7b97ffbd4a5); Time taken: 0.009 seconds
INFO  : OK
Error: java.io.IOException: org.apache.hadoop.security.authorize.AuthorizationException: User [hive] is not authorized to perform [DECRYPT_EEK] on key with ACL name [wumei10_key2]!! (state=,code=0)
0: jdbc:hive2://**.192.22:10000> 



[root@**rcw1 ~]# hive shell
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=512M; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: Using incremental CMS is deprecated and will likely be removed in a future release
17/04/12 19:42:57 WARN mapreduce.TableMapReduceUtil: The hbase-prefix-tree module jar containing PrefixTreeCodec is not present.  Continuing without it.
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=512M; support was removed in 8.0

Logging initialized using configuration in jar:file:/opt/cloudera/parcels/CDH-5.7.1-1.cdh5.7.1.p0.11/jars/hive-common-1.1.0-cdh5.7.1.jar!/hive-log4j.properties
WARNING: Hive CLI is deprecated and migration to Beeline is recommended.
hive> use encryption_test_db;
OK
Time taken: 1.569 seconds
hive> select * from pokes1;
OK
3       4
5       6
7       8
Time taken: 0.831 seconds, Fetched: 3 row(s)
hive> ^C[root@**rcw1 ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: wumei10@LFDC.**-GROUP.NET

Valid starting       Expires              Service principal
04/12/2017 19:26:25  04/13/2017 19:26:25  krbtgt/LFDC.**-GROUP.NET@LFDC.**-GROUP.NET
        renew until 04/19/2017 19:26:25
```






