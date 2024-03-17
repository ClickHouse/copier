
## Introduction

[Consider removal of clickhouse-copier](https://github.com/ClickHouse/ClickHouse/issues/60734)

Clickhouse-copier is a tool for copying data between clusters and one of the easy ways for resharding. For ease of use, you can run it not only on a local machine, but also on a KS cluster. Use a ready-made Helm Hart. Enter settings for unique values in settings from table directly into the configmap.

## Install

```console
helm install my-database -n my-namespace -f values.yaml .
```

## Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```console
helm install my-database -n my-namespace -f values.yaml .
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Parameters

| Name                     | Description                                                                             | Value           |
| ------------------------ | --------------------------------------------------------------------------------------- | --------------- |
| `replicaCount`           | the number of workers, depending on the power of your cluster and the number of shards. Selected individually. In practice, often the number of shards is equal to the number of workers, but more is possible          | `"1"`            |
| `host_replica_pull`           | Replica names. It is understood that only unique shards are indicated (without replicas), for example, if there are 12 shards, then we register 12 shards (only one replica from the shard)                | `"[]"`            |
| `user_pull`       | Username with rights to connect to the source cluster  (Grant select)                          | `"user_select"`            |
| `password_user_pull`      | user password with rights to connect to the source cluster                             | `"password"`            |
| `host_replica_push`           | name of the destination cluster replicas. We also indicate only one replica from the shard   | `[]`            |
| `user_push`      | Username with rights to connect to the destination cluster (Grant insert)                       | `{}`            |
| `password_user_push`          | user password with rights to connect to the destination cluster                                                           | `cluster.local` |
| `zookeeper_host`            | Zookeeper host ip or dns name                                       | `192.168.160.58`            |
| `path_copier` | path for task in zookeeper|clickhouse-keepeer | `my_baza_part1`         |
| `database_pull` | source base                                   | `database`     |
| `table_pull`    | source table                                      | `table`  |
| `database_push` | destination base                                    | `database`     |
| `table_push`    | destination table                                     | `table`  |
| `engine_table`    | engine table and settings for create                                  | ``  |
| `sharding_key`    | for sharding key it is better to use cityHash64(object_order_id), etc.                                  | ``  |
| `image.tag`    | The image used to run the clickhouse-copier. From 24.3 will be absent, whether there will be a separate one is unknown                                  | ``  |


## My experience

On large clusters with multiple resharding, transferring small tables can take a long time when using resharding, so it is better to use this solution not for 12 shards moving to 40. But for data from one replica to several shards. Practice has also shown that using replicas in the configuration file brings additional problems. Use one replica at a time, and first create the tables yourself using clickhouse, your data is replicated where necessary.

Optionally use enable_partitions:
````
                    <sharding_key>rand()</sharding_key>
                    # <enabled_partitions>
                    #     <partition>'2018-02-26'</partition>
                    #     <partition>'2018-03-05'</partition>
                    #     ...
                    # </enabled_partitions>
````

helm-chart is aimed at the fact that the server configuration has already been configured, you need to specify in values the path to task-zookeeper (there should be a separate one for each task), the names of databases and tables

The solution is simple, you need to fill out the configuration file. Install zookeeper and run copying tasks.

zookeeper can be used existing or installed separately using popular charts


### Additionally, consider the section configuration parameters in configmaps settings:

````
<settings>
        <max_execution_time>10000000</max_execution_time> 
        <send_timeout>50000000</send_timeout>
        <receive_timeout>5000000</receive_timeout>
        <max_result_bytes>0</max_result_bytes>
        <max_rows_to_read>0</max_rows_to_read>
        <max_bytes_to_read>0</max_bytes_to_read>
        <max_columns_to_read>1000</max_columns_to_read>
        <max_temporary_columns>1000</max_temporary_columns>
        <max_temporary_non_const_columns>1000</max_temporary_non_const_columns>
        <min_execution_speed>0</min_execution_speed>
        <max_result_rows>0</max_result_rows>
        <max_result_bytes>0</max_result_bytes>
        <connect_timeout>30</connect_timeout>
        <insert_distributed_sync>1</insert_distributed_sync>
        <allow_to_drop_target_partitions>1</allow_to_drop_target_partitions>
</settings>
````

### What is the maximum speed when using clickhouse-copier?

When using a small number of workers, the speed is within 400 Mbit. On the test bench, when using a configuration of 250 workers and 12 running processes (4 in each container), it was possible to achieve a speed of 1.67 Gbit/sec, but at the same time the delays for writing from disks reach 3-4 seconds - metric in prometheus ``irate(node_disk_io_time_weighted_seconds_total{instance=~"$instance"}[5m])``

###  [insert_distributed_sync](https://github.com/ClickHouse/ClickHouse/issues/20053#issuecomment-773069116)
````
insert_distributed_sync = 1 has the following advantages:

data is available to SELECT from replica immediately;
if all replicas are unavailable, you get the error instead of buffering data on local filesystem;
better resource usage due to less data copying;
no risk of data loss when server with Distributed table will fail;
no buffering of data in local filesystem - no concerns on the size of buffered data, queue size and delays;


insert_distributed_sync = 1 has the following disadvantages:

when there are large number of shards (in order of 100), the latency of INSERT will increase significantly;
when there are large number of shards (in order of 100), synchronous INSERT will be less reliable, you will have more transient errors;
does not work if insert_quorum is set but insert_quorum_parallel is disabled;
it does not batch large number of small INSERTs - every INSERT goes to the shards immediately;
````

### Error SquashingTransform

We are checking the settings data from the two clusters; there may be a discrepancy on version 22.8 compared to 23.3. It is necessary to bring these settings into the same form. Or recreate the tables with block settings as in the original.

Check settings:

```
SELECT
    name,
    value
FROM system.settings
WHERE name LIKE '%block_size%'
```

Settings at the inventory_group level for a new cluster (this is information for admins):

```
clickhouse_merge_tree_config:
    cleanup_delay_period: 300
    enable_mixed_granularity_parts: 1
    max_delay_to_insert: 2
    merge_max_block_size: 65505
    merge_selecting_sleep_ms: 600000
    parts_to_delay_insert: 150
    parts_to_throw_insert: 300
clickhouse_profiles_default:
    default:
        max_block_size: 65505
        max_insert_block_size: 1.048545e+06
        max_joined_block_size_rows: 65505
        min_insert_block_size_bytes: 2.6842752e+08
        min_insert_block_size_rows: 1.048545e+06
    system:
        max_block_size: 65505
        max_insert_block_size: 1.048545e+06
        max_joined_block_size_rows: 65505
        min_insert_block_size_bytes: 2.6842752e+08
        min_insert_block_size_rows: 1.048545e+06
clickhouse_version: 23.3.18.15
```



