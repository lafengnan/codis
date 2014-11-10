## Deploy

### Configuration file

`codis-config` and `codis-proxy` will take `config.ini` in current directory by default without a specific `-c`.

`config.ini`:

```
zk=localhost:2181   <- Location of `zookeeper`, use `ck=ck://hostname1:2181,hostname2:2181,hostname3:2181,hostname4:2181,hostname5:2181` for `zookeeper` clusters.
product=test        <- Product name, also the name of this Coids clusters, can be considered as namespace, Codis with different names have no intersection. 
proxy_id=proxy_1    <- Proxy will take this as identifier for proxy, multiple proxy can use different `config.ini` with various `proxy_id`.
```

### Workflow
1. Execute `cconfig slot init` to initialize slots
2. Starting and compiling a Codis Redis has no difference from a normal Redis Server
3. Add Redis server group, each server group as a Redis server group, only one master is allowed while could have multiple slaves. Group id only support integer lager than 1.

```
$ ../bin/codis-config server -h
usage:
    codis-config server list
    codis-config server add <group_id> <redis_addr> <role>
    codis-config server remove <group_id> <redis_addr>
    codis-config server promote <group_id> <redis_addr>
    codis-config server add-group <group_id>
    codis-config server remove-group <group_id>
```

For example: Add two server group with the ids of 1 and 2, each has two Redis instances, a master and a slave.

First, add a group with id of 1 and assign a Redis master to it:

```
$ ./codis-config server add 1 localhost:6379 master
```

Second, assign a Redis slave to this group:

```
$ ./codis-config server add 1 localhost:6380 slave
```

Then the group with id of 2:

```
$ ./codis-config server add 2 localhost:6479 master
$ ./codis-config server add 2 localhost:6479 slave
```

4. Config slot range of server group

Codes implement data segmentation with Pre-sharding mechanism, 1024 slots will be segmented by default,a single key use following formula to determine which slot to resident, each slot has a server group id represents the server group which will provide service.

```
$ ./codis-config slot -h                                                                                                                                                                                                                     
usage:
    codis-config slot init
    codis-config slot info <slot_id>
    codis-config slot set <slot_id> <group_id> <status>
    codis-config slot range-set <slot_from> <slot_to> <group_id> <status>
    codis-config slot migrate <slot_from> <slot_to> <group_id> [--delay=<delay_time_in_ms>]
```

For exmaple, config server group 1 provide service for slot [0, 511], server group 2 provide service for slot [512, 1023]

```
$ ./codis-config slot range-set 0 511 1 online
$ ./codis-config slot range-set 512 1023 2 online
```

5. Start `codis-proxy`

```
../bin/codis-proxy -c config.ini -L ./log/proxy.log  --cpu=8 --addr=0.0.0.0:19000 --http-addr=0.0.0.0:11000
```

`codas-proxy`’s status are now `offline`, put it `online` to provide service:

```
 ../bin/codis-config -c config.ini proxy online <proxy_name>  <---- proxy id, e.g. proxy_1
```

6. Start dashboard server(Optional but recommended)

```
../bin/codis-config -c config.ini -L ./log/dashboard.log dashboard --addr=:18087 --http-log=./log/requests.log
```
7. Open http://localhost:18087/admin in browser

Now you can achieve operations in browser. Enjoy!

## Data Migration

Codis offers a reliable and transparent data migration mechanism, also it’s a killer feature which made Codis distinguished from other static distributed Redis solution, such as Twemproxy.

The minimum data migration unit is `key`, we add some specific actions—such as `SLOTSMGRT`—to Codis to support migration based on `key`, which will send a random record of a slot to another Codis Redis instance each time, after the transportation is confirmed the original record will be removed from slot and return slot’s length. The action is atomically.

For example: migrate data in slot with ID from 0 to 511 to server group 2, `--delay` is the sleep duration after each transportation of record, which is used to limit speed, default value is 0. 

```
$ ./cconfig slot migrate 0 511 2 --delay=10
```

Migration progress is reliable and transparent, data won’t vanish and top layer application won’t terminate service. 

Notice that migration task could be paused, but if there is a paused task, it must be fulfilled before another start(means only one migration task is allowed at the same time). 

## Auto Rebalance

Codis support dynamic slots migration based on RAM usage to balance data distribution.
 
```
$./cconfig slot rebalance
```

Requirements:

* All slots’ status should be `online`, namely no transportation task is running. 
* All server groups must have a master. 