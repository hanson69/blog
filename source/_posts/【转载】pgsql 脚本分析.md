---
title: 【转载】pgsql 脚本分析
date: 2020-05-10 19:51:24
categories: DBA
tags:
- 高可用
- pacemaker
- corosync
- RA
---


Pacemaker + Cronsync可以说是目前为止搭建PostgreSQL HA集群最简单靠谱的方式。而完成这一任务的主要是PostgreSQL的RA脚本文件pgsql。关于pgsql的使用说明请参考：http://clusterlabs.org/wiki/PgSQL_Replicated_Cluster
本文探讨一下pgsql的实现逻辑。

# 1. 功能概述

- 支持同步复制和异步复制
- 支持单机，双机集群和多机集群
- Master发生故障时自动failover
- 同步Slave发生故障时自动切换Master的同步复制到异步复制


# 2. 参数解释
下面只介绍几个比较重要的

**rep_mode**

```
有4种可选的复制模式
      none:缺省值，即单机模式无复制。
      async:异步复制
      sync:同步复制。实际上有一个Salve是同步复制，其它Slave是异步复制（即设置
      synchronous_standby_names为同步复制Slave的节点名）。
      当同步复制的Slave出现故障，Master将自动切换到异步复制（即设置  synchronous_standby_names = ''），
      否则Master会阻塞导致服务中断；如果集群中还有其它Slave，又会很快将其中一个Slave切换到同步复制。
      slave:一直处于hot standby状态从另外的主服务器复制数据。
```


**xlog_check_count**

检查xlog位置(即last_xlog_replay_location和last_xlog_receive_location中的较大者)的次数，默认值是3。

当集群还没有选定Master时，pgsql需要比较各个节点xlog位置，找到日志最新的那个作为Master。但是，xlog位置是各个节点分别更新到集群CIB的，可能存在延迟，当连续xlog_check_count次检查期间，所有节点的log位置都没有变更，则认为此时的xlog位置已经固定了，进行xlog位置比较，xlog位置最新的那个节点作为Master。如果有两个或多个节点的xlog位置都是最新的，那么选哪个节点作为Master效果都一样，此时交由Pacemaker决定选哪个。

**restart_on_promote**

提升一个Slave为Master时RA restart而不是promote PostgreSQL，因为提升会导致时间线加1。默认值是false。
官网wiki上的例子中restart_on_promote=true，尽管官网也指出这样做是为了举例方便，但还是容易让人觉得设置为true更合适。实际上restart_on_promote设为true是不安全的，如果某个节点的xlog位置在提升位置点的后面，那么PostgreSQL无法知道这个xlog位点是在提升前产生的还是提升后产生的。这样的节点重新去连新Master时的结果是不确定的（虽然实际测试发现PostgreSQL几乎总是报WAL记录错而中止复制，而没有出现更恶劣的数据不一致却隐瞒不报的问题），而且也无法利用pg_rewind进行数据修复。所以建议保持restart_on_promote为默认值false。
                                                      
# 3. RA脚本
pgsql脚本有2000多行，可以说设计非常精巧，下面是脚本的概要(按照rep_mode=sync的流程书写)
https://github.com/ClusterLabs/resource-agents/blob/master/heartbeat/pgsql


```
 pgsql_replication_start
    change_pgsql_status "$NODENAME" "STOP"
    delete_master_baseline
    exec_with_retry 0 $CRM_MASTER -v $CAN_NOT_PROMOTE
    make_recovery_conf
    delete_xlog_location
    set_async_mode_all
    if [ -f $PGSQL_LOCK ]; then
        ocf_exit_reason "My data may be inconsistent. You have to remove $PGSQL_LOCK file to force start."
        return $OCF_ERR_GENERIC
    fi
    pgsql_real_start
    change_pgsql_status "$NODENAME" "HS:alone"
    
pgsql_real_start
    pg_ctl start
    pgsql_real_monitor
  
pgsql_monitor
   pgsql_real_monitor
   pgsql_replication_monitor
   
pgsql_real_monitor
   判断postgres进程生死
   根据"select pg_is_in_recovery()"返回$OCF_RUNNING_MASTER 或 $OCF_SUCCESS
   
pgsql_replication_monitor
   对Master节点
       change_data_status "$NODENAME" "LATEST"
       change_pgsql_status "$NODENAME" "PRI"
       control_slave_status
   对Slave节点
        如果当前没有Master
            change_pgsql_status "$NODENAME" "HS:alone"
            have_master_right
        如果DATA_STATUS是DISCONNECT
            change_pgsql_status "$NODENAME" "HS:alone"
   
control_slave_status(run on Master)
    执行"select application_name,upper(state),upper(sync_state) from pg_stat_replication"获取所有salve的状态(data_status，master_score和pgsql_status)
    遍历所有节点并更新salve的状态到CIB:          
        change_data_status "$target" "$data_status"
        如果DATA_STATUS是"STREAMING|ASYNC"  //promote后，Slave刚连上Master，Master第一次调用pgsql_replication_monitor->control_slave_status
            change_master_score "$target" "$CAN_NOT_PROMOTE"
            set_sync_mode "$target"         //set_sync_mode会临时设置RE_CONTROL_SLAVE=true
        如果DATA_STATUS是"STREAMING|SYNC"   //RE_CONTROL_SLAVE=true导致pgsql_replication_monitor第二次调用control_slave_status，加速状态更新。
            change_master_score "$target" "$CAN_PROMOTE" //给同步Slave的设置一个较高的master分数，之后Master挂了，Pacemaker就会选它为新的Master。                                       
            change_pgsql_status "$target" "HS:sync"
        如果DATA_STATUS是"STREAMING|POTENTIAL"
            change_master_score "$target" "$CAN_NOT_PROMOTE"
            change_pgsql_status "$target" "HS:potential"
        如果DATA_STATUS是"DISCONNECT"
            change_master_score "$target" "$CAN_NOT_PROMOTE"
            set_async_mode "$target"
        如果DATA_STATUS是其它
            change_master_score "$target" "$CAN_NOT_PROMOTE"
            set_async_mode "$target"
            change_pgsql_status "$target" "HS:connected"


        
have_master_right(run on Slave)
    data_status不为空且不是"STREAMING|SYNC"或"LATEST"(rep_mode=async时还包括"STREAMING|ASYNC")错误返回（即不能作为候选Master）。
    show_xlog_location
    如果当前slave节点的xlog位置最新  //xlog位置即last_xlog_replay_location和last_xlog_receive_location中的较大者
        exec_with_retry 5 $CRM_MASTER -v $PROMOTE_ME  //强烈要求Pacemaker提升自己为新Master                                       
    否则
        change_data_status "$NODENAME" "DISCONNECT"


pgsql_notify
    pgsql_pre_promote
        如果当前节点的master-baseline比候选Master还新,设置当前节点的failcount为无穷大使其停止。
            ocf_log err "My data is newer than new master's one. New master's location : $master_baseline"
            exec_with_retry 0 $CRM_FAILCOUNT -r $OCF_RESOURCE_INSTANCE -U $NODENAME -v INFINITY
    pgsql_post_demote
        对于非demote对象的slave节点
            show_master_baseline
            change_pgsql_status "$NODENAME" "HS:alone"
    post start|stop 
        对于Master节点                                
            control_slave_status //Slave启动或停止后，Master重新检查Slave状态。
        
pgsql_promote
    touch $PGSQL_LOCK
    show_master_baseline
    
    for target in $NODE_LIST; do
        [ "$target" = "$NODENAME" ] && continue
        change_data_status "$target" "DISCONNECT"
        change_master_score "$target" "$CAN_NOT_PROMOTE"
    done
    
    if ocf_is_true ${OCF_RESKEY_restart_on_promote}; then
        pgsql_real_stop slave
        rm -f $RECOVERY_CONF
        pgsql_real_start
    else
        pgctl promote
        pgsql_real_monitor
        
    change_data_status "$NODENAME" "LATEST"
    exec_with_retry 0 $CRM_MASTER -v $PROMOTE_ME
    change_pgsql_status "$NODENAME" "PRI"
    


pgsql_demote
    exec_with_retry 0 $CRM_MASTER -v $CAN_NOT_PROMOTE
    pgsql_real_stop master
    change_pgsql_status "$NODENAME" "STOP"
    
pgsql_replication_stop
    exec_with_retry 5 $CRM_MASTER -v $CAN_NOT_PROMOTE
    pgsql_real_stop slave
    change_pgsql_status "$NODENAME" "STOP"
    set_async_mode_all
    
pgsql_real_stop
    pgctl stop
    if  [ "$1" = "master" -a "$OCF_RESKEY_CRM_meta_notify_slave_uname" = " " ]; then 
        rm -f $PGSQL_LOCK  //当Master是仅有的节点时才删除lock文件，条件苛刻，所以lock文件几乎总是要人工删除。                                     
    fi

```
#  4. 处理流程
根据RA的脚本，当rep_mode=sync时的处理流程如下
## 4.1 启动集群
1. start(所有节点)
创建recovery.conf，并设置synchronous_standby_names = '',这样每个节点开始时都处于异步的hot standby状态。
启动postgres进程

2. monitor(所有节点)
检查data_status，如果data_status不为空或是"STREAMING|SYNC"或"LATEST"（rep_mode=async还包括"STREAMING|ASYNC"），进一步比较各个节点的xlog位置，拥有最新位置的被选为Master(通过$CRM_MASTER -v $PROMOTE_ME)。
集群初始启动时，data_status为空，以后的data_status就是之前启动时残留的值。

3.pgsql_pre_promote(所有节点)
比较当前节点和候选Master的master-baseline，如果当前节点的baseline比候选Master还新,使当前节点的postgres停止。
因为Pacemaker会选择同步Slave作为新的Master，但同步Slave未必拥有最新的xlog，这时pgsql组织拥有更新xlog的异步Slave上线。

4.pgsql_promote(候选Master)
touch $PGSQL_LOCK
设置master_baseline
如果restart_on_promote=true，删除recovery.conf再重启postgres，否则通过pgctl promote提升Slave为Master。


## 4.2 failover(Master上postgres进程crash)
1. pgsql_demote(原Master)
2. pgsql_post_demote(所有节点)
  非demote对象的Slave通过show_master_baseline记录自己的xlog位置，这一步的目的是为了后面pgsql_pre_promote做比较。
3.pgsql_pre_promote(所有节点)
  检查本节点和候选Master的xlog位置，如果比Master的xlog位置还新，停止本节点上的pgsql。
4. pgsql_promote(候选Master)


## 4.3 failover(Master的物理机，OS或网络故障 )
1. pgsql_pre_promote(所有节点)
2. pgsql_promote(候选Master)

## 4.4 同步异步切换
（rep_mode为"sync"的场合，刚启动或同步Slave发生故障时触发）

pgsql_monitor->pgsql_replication_monitor->control_slave_status(Master)
如果某Slave的DATA_STATUS是"STREAMING|ASYNC"，且synchronous_standby_names = '',将其设置到synchronous_standby_names中成为同步Slave。

如果某Slave的DATA_STATUS是"DISCONNECT"，且synchronous_standby_names就是该Salve,将synchronous_standby_names设置为''，切换到异步复制。

## 4.5 停止集群
1. pgsql_demote(Master)
停止postgres进程
如果当前没有Salve节点，删除$PGSQL_LOCK文件
2. pgsql_post_demote(所有节点)
非demote对象的Slave通过show_master_baseline记录自己的xlog位置


## 4.6 使用上的注意
**1. 脑裂防护**

双机集群在网络断开时会出现脑裂，可通过配置fence设备解决。
另一个办法是加入第3个节点只做仲裁，不配置资源，并设置no-quorum-policy=stop。
比如:用node3充当仲裁节点

```


    pcs resource update pgsql node_list="node1 node2"
    pcs constraint location msPostgresql avoids node3
    pcs property set no-quorum-policy=stop


```

如果是对等的3节点，就更简单了

```


    pcs resource update pgsql node_list="node1 node2 node3"
    pcs property set no-quorum-policy=stop


```
**2. restart_on_promote的设置**

前面已经提到，安全起见restart_on_promote应该设置为true。除此以外设置为true还有另一个目的，恢复旧Master的时候可以使用pg_rewind。但需要注意，每次集群重启都会导致时间线加1，次数多了时间线可能会比较大，其实这可以改进。


## 4.7 集群操作

1. 初始化集群
参照官方wiki的内容初始化集群。
2. 停止服务

```


    pcs resource disable msPostgresql
    rm -f /var/lib/pgsql/tmp/PGSQL.lock


```
既然pgsql不能自己删PGSQL.lock，我们正常停止时，可以帮它删。                                          

3. 启动服务

```
pcs resource enable msPostgresql 
```

4. 在线迁移
当前Master是node2，希望迁移到任意节点。
执行命令：

```
crm_attribute -l reboot -N node2 -n master-pgsql -v -INFINITY
```
因为pgsql在demote的时候提前把postgres进程停掉了，所以monitor捕获pgsql停止后会认为发生了一次故障，可能导致原Master无法作为Slave上线，需要再做一次cleanup。当然还要删除旧Master上的 PGSQL.lock文件。

```


    rm -f /var/lib/pgsql/tmp/PGSQL.lock
    pcs resource cleanup msPostgresql


```
# 5. 故障恢复
## 5.1 通过pg_rewind修复原Master

```


    su - postgres
    rm -f /var/lib/pgsql/tmp/PGSQL.lock
    rm -f /data/postgresql/data/recovery.conf
    killall -9 postgres
    pg_ctl -D /data/postgresql/data start
    pg_ctl -D /data/postgresql/data stop
    pg_rewind -D /data/postgresql/data --source-server="host=192.168.0.216" -P
    exit
    pcs resource cleanup msPostgresql


```

注：上面的192.168.0.216为Master VIP

## 5.2 Slave的修复
如果仅仅是posgres进程crash而数据没有损坏，执行下面的命令即可。

```
pcs resource cleanup msPostgresql 
```
## 5.3 重建基数备份恢复

```


    su - postgres
    killall -9 postgres
    rm -rf /data/postgresql/data/*
    pg_basebackup -h 192.168.0.216 -U postgres -D /data/postgresql/data -X stream -P
    rm -f /var/lib/pgsql/tmp/PGSQL.lock
    exit
    pcs resource cleanup msPostgresql


```

#  6. 其它
## 6.1 候选Master
rep_mode的sync时，同步Slave作为候选Master。
rep_mode的async时，通过xlog位置比较确定候选Master。这个过程时间会比较长（有10几秒），通过xlog_check_count控制。

## 6.2 PGSQL.lock文件
PGSQL.lock文件的目的是防止旧Master上线，因为旧Master和新Master的数据不一致。但这里存在一个问题，pgsql正常停止时往往不能把PGSQL.lock文件删除，导致下次这个节点无法起来。其实有了时间线（restart_on_promote=false）的防护，应该可以不要PGSQL.lock文件的。

## 6.3 xlog位置
xlog位置取的是pg_last_xlog_replay_location()和pg_last_xlog_receive_location()中较大的那个值。
但这里也有一个问题，Master是没有这个值的。这里，pgsql RA利用了PG的一个行为，PG在进入recovery状态时是有pg_last_xlog_replay_location()值的，并且当它被提升为Master或删除recovery.conf重启后，这个值仍然存在(再重启一次就没有了)。
而pgsql RA启动服务的过程是先将所有节点都进recovery模式，再选一个提升为Master，这使它可以拿到Master的xlog位置。

## 6.4 migration-threshold的设置
官方wiki的例子中migration-threshold被设为1，这过于严格。当Slave上的postgres进程crash后本来可以直接拉起来的，但现在需要人工干预了。设成1的好处也是有的，就是Master crash后直接做failover而不在原地尝试重启postgres，原地重启走的是crash 恢复的流程，时间可能会不确定，而且重启成功了数据还没有预热，性能可能受影响。不过目前的pgsql中有PGSQL.lock的防护，原地重启Master上的 的postgres是不会成功的，但会浪费failover的时间。        


# 原文：
http://blog.chinaunix.net/uid-20726500-id-5689969.html

# 其它资料：
https://github.com/ClusterLabs/resource-agents/blob/master/heartbeat/pgsql

https://wiki.clusterlabs.org/wiki/PgSQL_Replicated_Cluster

[The OCF Resource Agent Developer's Guide](http://www.linux-ha.org/doc/dev-guides/ra-dev-guide.html)

https://documentation.suse.com/zh-cn/sle-ha/15-SP1/html/SLE-HA-all/book-sleha-guide.html






