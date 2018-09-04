Elasticsearch - add fields to index template
============================================

Intro
-----

Problem: How to add fields to the default index template so that they are
parsed using a different syntax than `string`?

These instructions are primarily for OpenShift logging but should apply to any
Elasticsearch installation by removing the OpenShift specific bits.  They also
apply to Elasticsearch 2.x for OpenShift 3.4 -> 3.10, so may require
some tweaking to work with ES 5.x.  The instructions assume your logging
namespace is `logging` - use `openshift-logging` with OpenShift 3.10 and later.

OpenShift logging Elasticsearch 2.x uses the following default index templates for
`.operations.*` and `project.*` indices:
https://github.com/ViaQ/elasticsearch-templates/releases/download/0.0.16/com.redhat.viaq-openshift-operations.2.4.4.template.json
and
https://github.com/ViaQ/elasticsearch-templates/releases/download/0.0.16/com.redhat.viaq-openshift-project.2.4.4.template.json
Any undefined field not specified here will be treated as a string:

    "mappings": {
        "_default_": {
          "dynamic_templates": [
            {
              "string_fields": {
                "mapping": {
                  "index": "analyzed",
                  "norms": {
                    "enabled": true
                  },
                  "type": "string"
                },
                "match": "*",
                "match_mapping_type": "string"

If you have time valued fields that you want to use for sorting and collation, you
will need to define these separately in separate index templates.

Steps
-----

Identify the index pattern for the indices you want to add the fields to.  For OpenShift
logging this will be `.operations.*` or `project.*`.  If there is a specific project that
will only have these fields, you can specify a specific index pattern for the indices for
this project e.g. `project.this-project-has-time-fields.*`.

Create a JSON file for each index pattern, like this:

    {
        "order": 20,
        "mappings": {
          "_default_": {
            "properties": {
              "mytimefield1": {
                "doc_values": true,
                "format": "yyyy-MM-dd HH:mm:ss,SSSZ||yyyy-MM-dd'T'HH:mm:ss.SSSSSSZ||yyyy-MM-dd'T'HH:mm:ssZ||dateOptionalTime",
                "index": "not_analyzed",
                "type": "date"
              },
              "mytimefield2": {
                "doc_values": true,
                "format": "yyyy-MM-dd HH:mm:ss,SSSZ||yyyy-MM-dd'T'HH:mm:ss.SSSSSSZ||yyyy-MM-dd'T'HH:mm:ssZ||dateOptionalTime",
                "index": "not_analyzed",
                "type": "date"
              }
            }
          }
        }
        "template": ".operations.*"
    }

Call this one `add-time-fields-to-operations-indices.json`.

    {
        "order": 20,
        "mappings": {
          "_default_": {
            "properties": {
              "mytimefield1": {
                "doc_values": true,
                "format": "yyyy-MM-dd HH:mm:ss,SSSZ||yyyy-MM-dd'T'HH:mm:ss.SSSSSSZ||yyyy-MM-dd'T'HH:mm:ssZ||dateOptionalTime",
                "index": "not_analyzed",
                "type": "date"
              },
              "mytimefield2": {
                "doc_values": true,
                "format": "yyyy-MM-dd HH:mm:ss,SSSZ||yyyy-MM-dd'T'HH:mm:ss.SSSSSSZ||yyyy-MM-dd'T'HH:mm:ssZ||dateOptionalTime",
                "index": "not_analyzed",
                "type": "date"
              }
            }
          }
        }
        "template": "project.*"
    }


Call this one `add-time-fields-to-project-indices.json`.

Load these into Elasticsearch.  You'll need the name of one of the Elasticsearch
pods:

    oc get -n logging pods -l component=es

Pick one and call it `$espod`.  If you have a separate OPS cluster, you'll need
to identify one of the es-ops Elasticsearch pods too, for the `.operations.*`
indices:

    oc get -n logging pods -l component=es-ops

Pick one and call it `$esopspod`.

Load the file `add-time-fields-to-project-indices.json` into `$espod`:

    file=add-time-fields-to-project-indices.json
    cat $file | \
    oc exec -n logging -i -c elasticsearch $espod -- \
        curl -s -k --cert /etc/elasticsearch/secret/admin-cert \
        --key /etc/elasticsearch/secret/admin-key \
        https://localhost:9200/_template/$file -XPUT -d@- | \
    python -mjson.tool

Load the file `add-time-fields-to-operations-indices.json` into `$esopspod`, or
`$espod` if you do not have a separate OPS cluster:

    file=add-time-fields-to-operations-indices.json
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
`oc exec` command as above to query:

    oc exec -n logging -i -c elasticsearch $esopspod -- \
        curl -s -k --cert /etc/elasticsearch/secret/admin-cert \
        --key /etc/elasticsearch/secret/admin-key \
        https://localhost:9200/project.*/_search?sort=mytimefield2:desc | \
    python -mjson.tool

You should see your records sorted in descending order by the `mytimefield2` field
with the latest time listed first.
