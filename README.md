# Repairing ElasticSearch

**by default *dry_run=True* when you set it to false you are potetially chosing to overwrite primary shards ** 

Basically we managed to delete the same disk on each of 12 nodes. This had the effect of killing a lot of primary shards and leaving the cluster in an unrecoverable state. This is the script i used to bring it back to green.

We've lost data - but I don't want to loose the whole index because of 1 lost primary shard. 

It basically uses the **_cat/shards** to work out unassigned primary and replica shards and **_cluster/reroute** to assign shards to data nodes. It works because of the **allocate_primary** flag in [**_cluster/reroute**](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-reroute.html)

```
The allow_primary parameter will force a new empty primary shard to be allocated without any data. If a node which has a copy of the original shard (including data) rejoins the cluster later on, that data will be deleted: the old shard copy will be replaced by the new live shard copy.
```
This allows me to replace the lost primary shard with and empty one, then reasigning the replicas copies the new empty primary.    

There is an extra piece that is needed, on a production cluster ElasticSearch will aggressively try to recover primary shards. You get to a point at which you cannot **reroute** primary shards because they are active.

## The process

- Wait for the number of allocated primary shards to stop increasing 
  ( **Warning**: reroute will result in data loss if you start too early )
- Disable primary shard recoveries
```
curl -XPUT localhost:9200/_cluster/settings -d '{
    "transient" : {
        "cluster.routing.allocation.node_initial_primaries_recoveries" : 0
    }
}'
```
- _custer/reroute missing primary shards with **allocate_primary** set to 1, these should now all be in "UNASSIGNED" state
- Optional: reassign replicas. Once the cluster is back in "yellow" these will proceed and be done automatically but it can be faster if you do them. In my case the nodes were repeatedly cycling through restarting shards but hitting FileNotFound exceptions. 
- Re-enable primary shard recoveries ( the default value is 4 )
```
curl -XPUT localhost:9200/_cluster/settings -d '{
    "transient" : {
        "cluster.routing.allocation.node_initial_primaries_recoveries" : 4
    }
}'
```

**NOTE: following script only tested with version 1.3.6 of ElasticSearch**

### Further reading 
 - [T37: same method but doesn't stop the primary recoveries](https://t37.net/how-to-fix-your-elasticsearch-cluster-stuck-in-initializing-shards-mode.html)
 - [ElasticSearch: Shard allocation settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/shards-allocation.html)
 
### Shard allocation settings / rolling restart
 
Trying to resolve these issues I kept trying to disable shard allocation, as per the [rolling restart instructions](https://www.elastic.co/guide/en/elasticsearch/guide/current/_rolling_restarts.html). The problem is you cannot reroute if this setting is none.  
