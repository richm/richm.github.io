# Problem - unassigned_shards and cluster status RED

Using OpenShift origin-aggregated-logging 1.2, Elasticsearch 1.5.2, the cluster status is RED.

    oc exec logging-es-xxx-N-yyy -n logging -- curl -s \
      --key /etc/elasticsearch/keys/admin-key \
      --cert /etc/elasticsearch/keys/admin-cert \
      --cacert /etc/elasticsearch/keys/admin-ca \
      https://localhost:9200/_cluster/health | \
      python -mjson.tool
    {
        "active_primary_shards": 12345,
        "active_shards": 12345,
        "cluster_name": "logging-es",
        "initializing_shards": 0,
        "number_of_data_nodes": 3,
        "number_of_nodes": 3,
        "number_of_pending_tasks": 0,
        "relocating_shards": 0,
        "status": "red",
        "timed_out": false,
        "unassigned_shards": 7
    }

The problem is the unassigned_shards.  We need to identify those shards and
figure out how to deal with them so the cluster can move to `yellow` or
`green`.

# Solution - identify and delete problematic indices

Use the `/_cluster/health?level=indices` to get a list of the indices status:

    oc exec logging-es-xxx-N-yyy -n logging -- curl -s \
      --key /etc/elasticsearch/keys/admin-key \
      --cert /etc/elasticsearch/keys/admin-cert \
      --cacert /etc/elasticsearch/keys/admin-ca \
      https://localhost:9200/_cluster/health?level=indices | \
      python -mjson.tool > indices.json

The report will list each index and its state:

    "my-index.2017.03.15":{
       "active_primary_shards": 4,
       "active_shards": 4,
       "initializing_shards": 0,
       "number_of_replicas": 0,
       "number_of_shards": 5,
       "relocating_shards": 0,
       "status": "red",
       "unassigned_shards": 1
     },
     ...

Look for records that have `"status": "red"` and an `"unassigned_shards"` with
a value of `1` or higher.  IF YOU DON'T NEED THE DATA ANYMORE, AND ARE SURE
THAT THIS DATA CAN BE LOST, then it might be easiest to just delete these using
the REST API:

    oc exec logging-es-xxx-N-yyy -n logging -- curl -s \
      --key /etc/elasticsearch/keys/admin-key \
      --cert /etc/elasticsearch/keys/admin-cert \
      --cacert /etc/elasticsearch/keys/admin-ca \
      -XDELETE https://localhost:9200/my-index.2017.03.15

If you need to recover this data, or deletion is not working, then use the
recovery procedure documented at
[indices recovery](https://www.elastic.co/guide/en/elasticsearch/reference/1.5/indices-recovery.html)

