Elasticsearch - change number of shards for index template
==========================================================

Intro
-----

These instructions are primarily for OpenShift logging but should apply to any
Elasticsearch installation by removing the OpenShift specific bits.  They also
apply to Elasticsearch 2.x for OpenShift 3.4 -> 3.10, so may require
some tweaking to work with ES 5.x.  The instructions assume your logging
namespace is `logging` - use `openshift-logging` with OpenShift 3.10 and later.

The default number of shards per index for OpenShift logging is 1, which is by
design not to break very large deployments with a large number of indices,
where the problem is having too many shards.  However, for deployments with a
small number of very large indices, this can be problematic.  Elasticsearch
recommends keeping shard size under 50GB, so increasing the number of shards
per index can help with that.

Steps
-----

Identify the index pattern you want to increase sharding for.  For
OpenShift logging this will be `.operations.*` or `project.*`.  If there are
specific projects that typically generate much more data than others, and you
need to keep the number of shards down, you can shard very specific patterns
e.g. `project.this-project-generates-too-many-logs.*`.  If you don't anticipate
having many namespaces/project/indices, you can just use `project.*`.

Create a JSON file for each index pattern, like this:

    {
        "order": 20,
        "settings": {
            "index": {
                "number_of_shards": 3
            }
        },
        "template": ".operations.*"
    }

Call this one `more-shards-for-operations-indices.json`.

    {
        "order": 20,
        "settings": {
            "index": {
                "number_of_shards": 3
            }
        },
        "template": "project.*"
    }

Call this one `more-shards-for-project-indices.json`.

Load these into Elasticsearch.  You'll need the name of one of the Elasticsearch
pods:

    oc get -n logging pods -l component=es

Pick one and call it `$espod`.  If you have a separate OPS cluster, you'll need
to identify one of the es-ops Elasticsearch pods too, for the `.operations.*`
indices:

    oc get -n logging pods -l component=es-ops

Pick one and call it `$esopspod`.

Load the file `more-shards-for-project-indices.json` into `$espod`:

    file=more-shards-for-project-indices.json
    cat $file | \
    oc exec -n logging -i -c elasticsearch $espod -- \
        curl -s -k --cert /etc/elasticsearch/secret/admin-cert \
        --key /etc/elasticsearch/secret/admin-key \
        https://localhost:9200/_template/$file -XPUT -d@- | \
    python -mjson.tool

Load the file `more-shards-for-operations-indices.json` into `$esopspod`, or
`$espod` if you do not have a separate OPS cluster:

    file=more-shards-for-operations-indices.json
    cat $file | \
    oc exec -n logging -i -c elasticsearch $esopspod -- \
        curl -s -k --cert /etc/elasticsearch/secret/admin-cert \
        --key /etc/elasticsearch/secret/admin-key \
        https://localhost:9200/_template/$file -XPUT -d@- | \
    python -mjson.tool

**NOTE** The settings will not apply to existing indices.  You will need to
perform a reindexing for that to work.  However, it is usually not a problem,
as the settings will apply to new indices, and curator will eventually delete
the old ones.

Results
-------

To see if this is working, wait until new indices are created, and use the
`_cat` endpoints to view the new indices/shards:

    oc exec -c elasticsearch $espod -- \
    curl -s -k --cert /etc/elasticsearch/secret/admin-cert \
        --key /etc/elasticsearch/secret/admin-key \
        https://localhost:9200/_cat/indices?v
    health status index                                                                        pri rep docs.count docs.deleted store.size pri.store.size 
    green  open   project.kube-service-catalog.d5dbe052-903c-11e8-8c22-fa163e6e12b8.2018.07.26   3   0       1395            0      2.2mb          2.2mb

The `pri` value is now `3` instead of the default `1`.  This means there are 3
shards for this index.  You can also check the `shards` endpoint:

    oc exec -c elasticsearch $espod -- \
    curl -s -k --cert /etc/elasticsearch/secret/admin-cert \
        --key /etc/elasticsearch/secret/admin-key \
        https://localhost:9200/_cat/shards?v
    index                                                                        shard prirep state     docs   store ip         node                            

    project.kube-service-catalog.d5dbe052-903c-11e8-8c22-fa163e6e12b8.2018.07.26 1     p      STARTED    596 683.3kb 10.131.0.8 logging-es-data-master-vksc2fwe 
    project.kube-service-catalog.d5dbe052-903c-11e8-8c22-fa163e6e12b8.2018.07.26 2     p      STARTED    590 652.6kb 10.131.0.8 logging-es-data-master-vksc2fwe 
    project.kube-service-catalog.d5dbe052-903c-11e8-8c22-fa163e6e12b8.2018.07.26 0     p      STARTED    602 628.1kb 10.131.0.8 logging-es-data-master-vksc2fwe

This lists the 3 shards for the index.  If you have multiple Elasticsearch
nodes, you should see more than one node listed in the `node` column of the
`_cat/shards` output.
