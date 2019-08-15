# 开启segment的mirror

---
- author：hanson
- date：2019.08.15
- label：greenplum， segment， mirror
---

GP系统的Mirror既可以在初始化时配置，也可以在现有系统重新配置

默认Mirror Segment会在Primary的主机集群中进行配置
### 在现有系统添加SegmentMirror（与Primary相同的主机集群）
1. 在所有Segment主机上分配Mirror的数据存储区域：
1. 必须确保所有Segment主机之间已经建立了互信
2. 运行gpaddmirrors命令启用GP系统的Mirror，例如：
$ gpaddmirrors-p 10000

> -p 选项：
> Optional. This number is used to calculate the database ports and replication ports used
> for mirror segments. The default offset is 1000.
Mirror port assignments are calculated as
> follows:
- > primary port + offset = mirror database port
- > primary port + (2 * offset) = mirror replication port
- > primary port + (3 * offset) = primary replication port
> For example, if a primary segment has port 50001, then its mirror will use a database port
> of 51001, a mirror replication port of 52001, and a primary replication port of 53001 by
> default.

在master上通过“gpstate -m”查看集群的mirror信息。
> -m (list mirrors)
Optional. List the mirror segment instances in the system, their current role, and
synchronization status.
```shell
[hanson@mdw ~]$ gpstate -m
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:-Obtaining Segment details from master...
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:--------------------------------------------------------------
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:--Current GPDB mirror list and status
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:--Type = Group
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:--------------------------------------------------------------
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:-   Mirror   Datadir                           Port    Status    Data Status
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:-   sdw2     /home/hanson/data/mirror/gpseg0   50000   Passive   Synchronized
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:-   sdw2     /home/hanson/data/mirror/gpseg1   50001   Passive   Synchronized
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:-   sdw1     /home/hanson/data/mirror/gpseg2   50000   Passive   Synchronized
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:-   sdw1     /home/hanson/data/mirror/gpseg3   50001   Passive   Synchronized
20190814:06:18:00:003263 gpstate:mdw:hanson-[INFO]:--------------------------------------------------------------
```

也可以通过“select * from gp_segment_configuration”查看

```
[hanson@mdw ~]$ psql -d postgres -c "select * from gp_segment_configuration"
 dbid | content | role | preferred_role | mode | status | port  | hostname | address | replication_port
------+---------+------+----------------+------+--------+-------+----------+---------+------------------
    1 |      -1 | p    | p              | s    | u      |  5432 | mdw      | mdw     |
    2 |       0 | p    | p              | s    | u      | 40000 | sdw1     | sdw1    |            41000
    4 |       2 | p    | p              | s    | u      | 40000 | sdw2     | sdw2    |            41000
    3 |       1 | p    | p              | s    | u      | 40001 | sdw1     | sdw1    |            41001
    5 |       3 | p    | p              | s    | u      | 40001 | sdw2     | sdw2    |            41001
    6 |       0 | m    | m              | s    | u      | 50000 | sdw2     | sdw2    |            51000
    7 |       1 | m    | m              | s    | u      | 50001 | sdw2     | sdw2    |            51001
    8 |       2 | m    | m              | s    | u      | 50000 | sdw1     | sdw1    |            51000
    9 |       3 | m    | m              | s    | u      | 50001 | sdw1     | sdw1    |            51001
(9 rows)
```



### 在现有系统添加Segment Mirror（与Primary不同的主机集群）
1. 确保GP已经在所有主机上安装
2. 在所有Segment主机上分配Mirror的数据存储区域
3. 必须确保所有Segment主机之间已经建立了互信
4. 创建配置文件，该文件包括所有主机名称，端口，数据目录。
可以通过下面的方式来创建配置文件：

```
$ gpaddmirrors-o mirror_config_file
```

例如，

```
filespaceOrder=
mirror0=0:sdw2：50000：51000：52000：/data/mirror/gpseg0
mirror1=1:sdw1：50000：51000：52000：/data/mirror/gpseg1
```


5. 运行gpaddmirrors命令启用GP系统的Mirror

```
$ gpaddmirrors-i mirror_config_file
```

