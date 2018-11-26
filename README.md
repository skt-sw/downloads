FlashBase 설치하기
=================

cluster 구성을 위한 directory를 생성한다.
<pre><code>mkdir ~/tsr2
mkdir ~/tsr2/cluster_1
</code></pre>

FlashBase 설치 파일(e.g. tsr2-installer.bin.flashbase_xxx)을 아래 경로에 복사한다. ('TSR2-package/bin/mac'에 있음))
<pre><code>cp tsr2-installer.bin.flashbase_v1.1.9.mac ~/tsr2/cluster_1/</code></pre>

deploy-flashbase.sh를 '~/tsr2'에 복사한다.('TSR2-package/scripts'에 있음)
<pre><code>cp deploy-flashbase.sh ~/tsr2/ </code></pre>

deploy-flashbase.sh를 열어서 아래와 같이 수정한다.

1. 'line 3'에서 cluster를 구성할 노드(서버)를 지정한다. 여러 대의 서버에 설치할 경우에는 'line 1'과 같이 설정한다.
2. 'line 17'에 cluster 번호를 입력한다. 'cluster_1'의 경우 '1'로 입력하면 된다. '1 2 3' 과 같이 여러 cluster에 동시에 설치하는 것도 가능하다.
3. 설치는 './deploy-flashbase.sh [file name]'와 같이 입력하면 된다.
<pre><code>./deploy-flashbase.sh cluster_1/tsr2-installer.bin.flashbase_v1.1.9.mac </code></pre>

``` console
  1 #nodes=("nv-accel-d01" "nv-accel-d02" "nv-accel-d03")
  2 #nodes=("nv-accel-w01" "nv-accel-w02" "nv-accel-w03" "nv-accel-w04" "nv-accel-w05" "nv-accel-w06")
  3 nodes=( "localhost")
  4
  5 INSTALLER_PATH=$1
  6
  7 [[ $INSTALLER_PATH == "" ]] && echo "NO ARGS" && echo "cmd <path of installer.bin>" && exit 1
  8 [[ ! -e $INSTALLER_PATH ]] && echo "NO FILE: $INSTALLER_PATH" && exit 1
  9
 10 INSTALLER_BIN=$(basename $INSTALLER_PATH)
 11 DATEMIN=`date +%Y%m%d%H%M%S`
 12 TSR2_DIR=~/tsr2
 13 echo "DATEMIN: $DATEMIN"
 14 echo "INSTALLER PATH: $INSTALLER_PATH"
 15 echo "INSTALLER NAME: $INSTALLER_BIN"
 16
 17 for cluster_num in "1";
 18 do
 19     CLUSTER_DIR=$TSR2_DIR/cluster_${cluster_num}
 20     BACKUP_DIR="${CLUSTER_DIR}_bak_$DATEMIN"
 21     CONF_BACKUP_DIR="${CLUSTER_DIR}_conf_bak_$DATEMIN"
 22     SR2_HOME=${CLUSTER_DIR}/tsr2-assembly-1.0.0-SNAPSHOT
 23     SR2_CONF=${SR2_HOME}/conf
 24
 25     echo "======================================================"
 26     echo "DEPLOY CLUSTER $cluster_num"
 27     echo ""
 28     echo "CLUSTER_DIR: $CLUSTER_DIR"
 29     echo "SR2_HOME: $SR2_HOME"
 30     echo "SR2_CONF: $SR2_CONF"
 31     echo "BACKUP_DIR: $BACKUP_DIR"
 32     echo "CONF_BACKUP_DIR: $CONF_BACKUP_DIR"
 33     echo "======================================================"
 34     echo "backup..."
 35     mkdir -p ${CONF_BACKUP_DIR}
 36     cp -rf ${SR2_CONF}/* $CONF_BACKUP_DIR
 37
 38     echo ""
 39
 40     for node in ${nodes[@]};
 41     do
 42         echo "DEPLOY NODE $node"
 43        # ssh $node "mv ${CLUSTER_DIR} ${BACKUP_DIR}"
 44         ssh $node "mkdir -p ${CLUSTER_DIR}"
 45         scp -r $INSTALLER_PATH $node:${CLUSTER_DIR}
 46         ssh $node "PATH=${PATH}:/usr/sbin; ${CLUSTER_DIR}/${INSTALLER_BIN} --full ${CLUSTER_DIR}"
 47         rsync -avr $CONF_BACKUP_DIR/* $node:${SR2_CONF}
 48     done
 49
 50     echo ""
 51 done

```


'.use_cluster' 파일을 이용하여 cluster 경로를 설정한다.('TSR2-package/scripts'에 있음)
<pre><code>cp .use_cluster ~/ </code></pre>

아래와 같이 입력하면 cluster_1 로 경로가 설정된다.
<pre><code>source ~/.use_cluster 1 </code></pre>

아래와 같이 경로가 설정되어 있는지 확인한다.
<pre><code>$ which flashbase
/Users/admin/tsr2cluster_1/tsr2-assembly-1.0.0-SNAPSHOT/sbin/flashbase

$ flashbase version
FlashBase(TM) version '1.1.9'
Last updated on 2018.08.06
</code></pre>

아래와 같이 process 수와 replication, disk 정보를 입력한다.

'line 4~5'에서 master node의 서버와 프로세스 수를 지정한다.(PORT 수가 프로세스 수)

'line 8~9'에서 slave node의 서버와 프로세스 수를 지정한다.

'line 13~15'에서 파일을 저장하기 위한 경로를 지정한다.

'line 18'에서 디스크 수를 지정한다.(mac에서는 directory 수,ssd_1~ssd_3)
<pre><code>$ flashbase edit

  1 #!/bin/bash
  2
  3 ## Master hosts and ports
  4 export SR2_REDIS_MASTER_HOSTS=( "127.0.0.1" )
  5 export SR2_REDIS_MASTER_PORTS=( $(seq 18100 18104) )
  6
  7 ## Slave hosts and ports (optional)
  8 export SR2_REDIS_SLAVE_HOSTS=( "127.0.0.1" )
  9 export SR2_REDIS_SLAVE_PORTS=( $(seq 18600 18604) )
 10
 11 ## only single data directory in redis db and flash db
 12 ## Must exist below variables; 'SR2_REDIS_DATA', 'SR2_REDIS_DB_PATH' and 'SR2_FLASH_DB_PATH'
 13 export SR2_REDIS_DATA="/sata_ssd/ssd_"
 14 export SR2_REDIS_DB_PATH="/sata_ssd/ssd_"
 15 export SR2_FLASH_DB_PATH="/sata_ssd/ssd_"
 16
 17 ## multiple data directory in redis db and flash db
 18 export SSD_COUNT=3
</code></pre>

'redis-master.conf.template'과 'redis-master.conf.template'를 아래와 같이 해당 cluster의 conf directory에 복사한다.('TSR2-package/conf'에 있음)
<pre><code>cp redis-master.conf.template ~/tsr2/cluster_1/tsr2-assembly-1.0.0-SNAPSHOT/conf
cp redis-slave.conf.template ~/tsr2/cluster_1/tsr2-assembly-1.0.0-SNAPSHOT/conf
</code></pre>

FlashBase 실행하기
==============================================================================

아래 순서로 실행한다.

1. 경로 설정
<pre><code>source ~/.use_cluster 1</code></pre>

2. 실행중인 프로세스 중지
<pre><code>flashbase stop</code></pre>

3. 기존 파일 삭제
<pre><code>flashbase clean</code></pre>

4. 'flashbase edit template'을 입력하여, 설정값을 확인 및 변경
<pre><code>flashbase edit template</code></pre>

5. 초기화와 함께 클러스터 구성
<pre><code>flashbase restart --reset --cluster --yes</code></pre>

6. 초기화 및 실행이 안되는 경우 대처

  - 아래 위치로 가서 로그를 확인한다.
<pre><code>cd $SR2_HOME/logs/redis</code></pre>

  - 로그에 아무런 정보가 없으면 아래와 같이 프로세스 하나만 실행해서 발생한 로그 메시지를 확인한다.
  - 'cluster_1' 부분은 클러스터에 맞게 수정 후 사용
<pre><code>DYLD_LIBRARY_PATH=~/tsr2/cluster_1/tsr2-assembly-1.0.0-SNAPSHOT/lib/native/ ~/tsr2/cluster_1/tsr2-assembly-1.0.0-SNAPSHOT/bin/redis-server ~/tsr2/cluster_1/tsr2-assembly-1.0.0-SNAPSHOT/conf/redis/redis-18101.conf</code></pre>

FlashBase에 적재하기
==============================================================================

적재는 아래와 같이 FlashBase package에 포함된 tsr2-tools를 사용한다.
사용 방법은 'tsr2-tools insert -d [적재할 데이터가 있는 파일 또는 directory] -s "|" -t [json file] -p 8 -c 1 -i'와 같다.
이 경우 seperator로 '|'를 사용했다.

``` console
tsr2-tools insert -d ./load -s "|" -t ./json/test.json -p 8 -c 1 -i
```

Test data로 적재하기
------------------------------------------------------------------------------
아래와 같이 'tsr2-test' directory를 생성하고 관련 파일들을 복사한다.
<pre><code>mkdir ~/tsr2-test</code></pre>

적재할 데이터 파일들은 '~/tsr2-test/load'에 복사한다.
<pre><code>mkdir ~/tsr2-test/load</code></pre>

'~/tsr2-test/json'에 test.json 파일을 생성한다.
<pre><code>mkdir ~/tsr2-test/json</code></pre>

``` console
vi test.json

  1   {
  2       "endpoint": "127.0.0.1:18101",
  3       "id": "101",
  4       "columns": 10,
  5       "partitions": [
  6           0, 3, 4
  7       ],
  8       "rowStore" : true
  9   }
```

적재는 아래와 같이 실행한다.
``` console
tsr2-tools insert -d ~/tsr2-test/load -s "|" -t test.json -p 8 -c 1 -i
```

아래와 같이 입력하면 적재 상황을 모니터링할 수 있다.
<pre><code>flashbase cli monitor</code></pre>

적재한 결과는 아래와 같다.
``` console
1534483316.477876 [0 127.0.0.1:49837] "NVWRITE" "D:{101:1:event_date:4:country:5:id}" "3:0:3:4" "1" "0" "event_date\x1dname\x1dcompany\x1dcountry\x1did\x1ddata1\x1ddata2\x1ddata3\x1ddata4\x1ddata5"
1534483316.478114 [0 127.0.0.1:49837] "NVWRITE" "D:{101:1:180818:4:Mexico:5:1681090140399}" "3:0:3:4" "1" "0" "180818\x1dMaite, Christian, Tad, Illiana\x1dUltrices Corp.\x1dMexico\x1d1681090140399\x1dPUY52MAG4EK\x1d6\x1d5\x1dut quam\x1dFusce feugiat. Lorem"
1534483316.478257 [0 127.0.0.1:49837] "NVWRITE" "D:{101:1:180320:4:American Samoa:5:1680111178699}" "3:0:3:4" "1" "0" "180320\x1dTatum, Eliana, Iola, Colby\x1dMi Eleifend Egestas Institute\x1dAmerican Samoa\x1d1680111178699\x1dQYV83PRB0JW\x1d12\x1d8\x1dtortor at risus. Nunc ac sem ut dolor\x1dnon, lobortis quis, pede. Suspendisse dui. Fusce diam"
1534483316.478315 [0 127.0.0.1:49837] "NVWRITE" "D:{101:1:171102:4:Chile:5:1617010935199}" "3:0:3:4" "1" "0" "171102\x1dTheodore, Holly, Carter, Fulton\x1dNisi Nibh Lacinia Industries\x1dChile\x1d1617010935199\x1dZEK46GWB7HN\x1d14\x1d5\x1dlacus. Quisque purus sapien,\x1dtempor lorem, eget mollis lectus pede et risus. Quisque libero"
1534483316.478390 [0 127.0.0.1:49837] "NVWRITE" "D:{101:1:190718:4:Saint Martin:5:1655052520899}" "3:0:3:4" "1" "0" "190718\x1dKenyon, Jeremy, Hedda, Wayne\x1dSit Consulting\x1dSaint Martin\x1d1655052520899\x1dZKV28BVO2UJ\x1d18\x1d1\x1dsed pede nec ante blandit viverra. Donec tempus, lorem\x1da purus. Duis elementum, dui quis accumsan convallis, ante lectus"
1534483316.478481 [0 127.0.0.1:49837] "NVWRITE" "D:{101:1:181125:4:Djibouti:5:1662041755899}" "3:0:3:4" "1" "0" "181125\x1dMarvin, Berk, Connor, Britanney\x1dDonec Dignissim Magna Company\x1dDjibouti\x1d1662041755899\x1dTOY60WWP1BV\x1d22\x1d10\x1dMorbi metus. Vivamus euismod urna. Nullam\x1dmollis vitae, posuere at, velit. Cras lorem lorem, luctus"
1534483316.478565 [0 127.0.0.1:49837] "NVWRITE" "D:{101:1:190808:4:Peru:5:1633051811499}" "3:0:3:4" "1" "0" "190808\x1dAlisa, Vernon, Gregory, Dale\x1dUltrices Foundation\x1dPeru\x1d1633051811499\x1dDKY94BEN1QF\x1d28\x1d6\x1dvitae erat\x1dipsum. Phasellus vitae"
1534483316.478632 [0 127.0.0.1:49837] "NVWRITE" "D:{101:1:180726:4:Chad:5:1637081315399}" "3:0:3:4" "1" "0" "180726\x1dDrew, Adrienne, Blaze, Jade\x1dAt Nisi PC\x1dChad\x1d1637081315399\x1dMKT50ZSU1QN\x1d32\x1d10\x1dsapien. Cras dolor dolor,\x1dNulla eget metus eu erat semper rutrum. Fusce dolor"
...
...

```

적재한 데이터 확인
``` console 
$fb cli
127.0.0.1:18100> metakeys *
 1) "M:{101:1:170807:4:Tuvalu:5:1612091856899}"
 2) "M:{101:1:171102:4:Chile:5:1617010935199}"
 3) "M:{101:1:171214:4:Singapore:5:1668021703999}"
 4) "M:{101:1:171221:4:Korea, South:5:1693112634099}"
 5) "M:{101:1:180320:4:American Samoa:5:1680111178699}"
 6) "M:{101:1:180415:4:Montenegro:5:1636031140599}"
 7) "M:{101:1:180504:4:Dominican Republic:5:1610011098499}"
 8) "M:{101:1:180726:4:Chad:5:1637081315399}"
 9) "M:{101:1:180810:4:Tunisia:5:1608020722999}"
10) "M:{101:1:180818:4:Mexico:5:1681090140399}"
11) "M:{101:1:181020:4:Belgium:5:1648100961899}"
12) "M:{101:1:181125:4:Djibouti:5:1662041755899}"
13) "M:{101:1:181224:4:Zambia:5:1662070325099}"
14) "M:{101:1:190312:4:Chile:5:1652011207799}"
15) "M:{101:1:190323:4:Brazil:5:1624050541999}"
16) "M:{101:1:190401:4:Ecuador:5:1638120251599}"
17) "M:{101:1:190714:4:Nicaragua:5:1642072229899}"
18) "M:{101:1:190718:4:Saint Martin:5:1655052520899}"
19) "M:{101:1:190808:4:Peru:5:1633051811499}"
20) "M:{101:1:event_date:4:country:5:id}"
127.0.0.1:18100> info keyspace
# Keyspace
db0:keys=40,memKeys=20,flashKeys=0,expires=20,avg_ttl=0
127.0.0.1:18100>

```

적재한 데이터를 'thriftserver beeline'으로 질의
아래처럼 Spark executor가 정상적으로 떴으면 thriftserver beeline을 사용해서 질의를 한다.
 ``` console
2500 ResourceManager
6294 CoarseGrainedExecutorBackend
2839 SparkSubmit
2586 NodeManager
6266 CoarseGrainedExecutorBackend
6219 ExecutorLauncher
 ```

'ddl_fb_test_101.sql'를 사용해서 아래와 같이 table을 생성한다.
``` console
thriftserver beeline -f ddl_fb_test_101.sql
```

수행 결과
``` console
CREATE TABLE `fb_test` (`user_id` STRING, `name` STRING, `company` STRING, `country` STRING, `event_date` STRING, `data1` STRING, `data2` STRING, `data3` STRING, `data4` STRING, `data5` STRING)
USING r2
OPTIONS (
  `query_result_partition_cnt_limit` '40000',
  `query_result_task_row_cnt_limit` '10000',
  `host` 'localhost',
  `serialization.format` '1',
  `query_result_total_row_cnt_limit` '100000000',
  `group_size` '10',
  `port` '18102',
  `mode` 'nvkvs',
  `partitions` 'user_id country event_date',
  `table` '101'
)
```

table(fb_test)을 생성하였으므로, 아래와 같이 질의를 하면 된다.
``` console
select event_date, company from fb_test  where event_date > '0';

or

select count(*) from fb_test;

or

select event_date, company, count(*) from fb_test  where event_date > '0' group by event_date, company;
```

FlashBase Commands
==============================================================================

기본적으로 아래와 같이 입력하여 client를 생성한다.
<pre><code>flashbase cli -h [host] -p [port]</code></pre>

'flashbase cli'만 입력하면 첫번째 host의 첫번째 port로 생성한다.
<pre><code>flashbase cli</code></pre>

client에 들어온 후에, 'info'를 입력하면 전체 상태를 확인할 수 있다.

``` console
$flashbase cli -h localhost -p 18500
localhost:18500> info
# Server
redis_version:3.0.7
redis_git_sha1:2d30588f
redis_git_dirty:0
redis_build_id:990d7b12314566d7
redis_mode:cluster
os:Darwin 17.3.0 x86_64
arch_bits:64
multiplexing_api:kqueue
gcc_version:4.2.1
process_id:52490
run_id:7f5dbe653c947acfef05a2ea9b32068fdf1c36b3
tcp_port:18500
uptime_in_seconds:237167
uptime_in_days:2
hz:10
lru_clock:3547335
config_file:/Users/admin/dev/cluster_5/tsr2-assembly-1.0.0-SNAPSHOT/conf/redis/redis-18500.conf

# Clients
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

# Alert Warnings
[EVICTION_WARNING] partition setting is inefficient, 00Rowgroups ratio is 100

# Memory
isOOM:false
used_memory:75585072
used_memory_human:72.08M
used_memory_rss:5165056
used_memory_peak:83956368
used_memory_peak_human:80.07M
used_memory_lua:36864
used_memory_rocksdb_total:114453440
used_memory_rocksdb_block_cache:100663296
used_memory_rocksdb_mem_table:13790144
used_memory_rocksdb_table_readers:0
used_memory_rocksdb_pinned_block:0
meta_data_memory:851760
percent_of_meta_data_memory:1
used_memory_client_buffer_peak:0
mem_fragmentation_ratio:0.07
mem_allocator:libc

# Persistence
loading:0
rdb_changes_since_last_save:288641
rdb_bgsave_in_progress:0
rdb_last_save_time:1534242338
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:1
rdb_current_bgsave_time_sec:-1
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:101291191
aof_base_size:1
aof_pending_rewrite:0
aof_buffer_length:0
aof_rewrite_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0

# Stats
total_connections_received:49
total_commands_processed:2626085
instantaneous_ops_per_sec:0
total_net_input_bytes:2727904217
total_net_output_bytes:2935287875
instantaneous_input_kbps:0.03
instantaneous_output_kbps:0.00
rejected_connections:0
sync_full:1
sync_partial_ok:9
sync_partial_err:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:9228
migrate_cached_sockets:0

# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=18600,state=online,offset=2731230897,lag=0
master_repl_offset:2731230897
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2730182322
repl_backlog_histlen:1048576

# CPU
used_cpu_sys:124.00
used_cpu_user:170.14
used_cpu_sys_children:1.15
used_cpu_user_children:5.19

# Cluster
cluster_enabled:1

# Keyspace
db0:keys=8884,memKeys=1451,flashKeys=0,expires=7433,avg_ttl=0
localhost:18500>
```

'info keyspace'와 같이 '#' 뒤에 오는 이름을 입력하면 해당 항목에 대해서만 확인할 수 있다.
``` console
localhost:18500> info keyspace
# Keyspace
db0:keys=8884,memKeys=1451,flashKeys=0,expires=7433,avg_ttl=0
localhost:18500>

```

