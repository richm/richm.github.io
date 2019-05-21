When writing data to Elasticsearch, the first time it sees a field that isn't already in the [field mapping](https://github.com/openshift/origin-aggregated-logging/blob/master/elasticsearch/index_templates/com.redhat.viaq-openshift-operations.template.json), it will try to detect the data type of the field based on the JSON type and some rules as specified here: [dynamic field mapping rules](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/dynamic-field-mapping.html), and it will add that field to the field mapping for the index.  For example, if Elasticsearch gets `"field1":9999`, it will create a field mapping which maps `"field1"` to a `long` type:
```json
...
  "field2" : {
    "type" : "long"
  },
...
```
Also, in Elasticsearch, this data is immutable in the index - it can be created but not changed without reindexing.  When using origin-aggregated-logging with `MERGE_JSON_LOG=true`, this can cause problems when different applications create fields with the same name but with incompatible data types.  For example, suppose another application writes a log with `"field2"` having a different data type `"field2":{"field21":"string","field22":1000}` (`"field2"` is a JSON hash).  You will get an error like this from Elasticsearch:
```json
{
  "error" : {
    "root_cause" : [
      {
        "type" : "mapper_parsing_exception",
        "reason" : "failed to parse [field2]"
      }
    ],
    "type" : "mapper_parsing_exception",
    "reason" : "failed to parse [field2]",
    "caused_by" : {
      "type" : "json_parse_exception",
      "reason" : "Current token (START_OBJECT) not numeric, can not use numeric value accessors\n at [Source: org.elasticsearch.common.bytes.BytesReference$MarkSupportingStreamInputWrapper@6340a27e; line: 1, column: 12]"
    }
  },
  "status" : 400
}
```
The error message is a bit verbose, but it means the field value `{"field21":"string","field22":1000}` is not a numeric value (because it is a Hash).

## How to view the dynamic mappings

If you want to see what are the mappings that have been added dynamically, use the [field mapping API](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/indices-get-field-mapping.html).  For example, with origin-aggregated-logging:
```bash
oc exec -c elasticsearch $espod -- es_util --query=_all/_mapping/*/field/field2?pretty
```
```json
{
  ".kibana" : {
    "mappings" : { }
  },
  ".operations.2019.03.21" : {
    "mappings" : {
      "com.redhat.viaq.common" : {
        "field2" : {
          "full_name" : "field2",
          "mapping" : {
            "field2" : {
              "type" : "long"
            }
          }
        }
      }
    }
  },
  ".searchguard" : {
    "mappings" : { }
  }
}
```
Where `$espod` is the name of one of your Elasticsearch pods.  You may have multiple definitions for a field, one for each index in which the field is defined. In origin-aggregated-logging, we create a new index for each day and for each namespace, so you may have many such definitions, and you may have different definitions, if some other index has `"field2"` with a different type.  With the `_mapping` command above, the fields are listed by index.  You can use a tool like `jq` to parse apart the JSON returned.

## Can I force Elasticsearch to store everything as a string?

Not exactly.  Using the [dynamic templates API](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/dynamic-templates.html) you might consider adding a dynamic mapping that forces every value to be a `string`:
```json
{
    "order": 20,
    "mappings": {
      "_default_": {
        "dynamic_templates": [
        {
          "force_all_to_string": {
            "match_mapping_type": "*",
            "mapping": {
              "type": "text",
              "fields": {
                "raw": {
                  "type":  "keyword",
                  "ignore_above": 256
                }
              }
            }
          }
        }
        ]
      }
    },
    "template": ".operations.*"
}
```
```bash
cat force_string_template.json | oc exec -i -c elasticsearch $espod -- es_util --query=_template/force_all_to_string -X PUT -d@-
```
And that seems to work for some types:
```bash
oc exec -c elasticsearch $espod -- es_util --query=.operations.2019.03.23/com.redhat.viaq.common?pretty -XPOST -d '{"field1":"stringval"}'
```
```json
{
  "_index" : ".operations.2019.03.23",
  "_type" : "com.redhat.viaq.common",
  "_id" : "AWmiLVg1uPjkpmElglqW",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "created" : true
}
```
Now I can add a numeric `field1` value which will be converted to a string:
```bash
oc exec -c elasticsearch $espod -- es_util --query=.operations.2019.03.23/com.redhat.viaq.common?pretty -XPOST -d '{"field1":1000}'
```
```json
{
  "_index" : ".operations.2019.03.23",
  "_type" : "com.redhat.viaq.common",
  "_id" : "AWmiLdOnuPjkpmElglqY",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "created" : true
}
```
However, if `field1` is a Hash or Array:
```bash
oc exec -c elasticsearch $espod -- es_util --query=.operations.2019.03.23/com.redhat.viaq.common?pretty -XPOST -d '{"field1":{"field11":"value"}}'
```
```json
{
  "error" : {
    "root_cause" : [
      {
        "type" : "mapper_parsing_exception",
        "reason" : "failed to parse [field1]"
      }
    ],
    "type" : "mapper_parsing_exception",
    "reason" : "failed to parse [field1]",
    "caused_by" : {
      "type" : "illegal_state_exception",
      "reason" : "Can't get text on a START_OBJECT at 1:11"
    }
  },
  "status" : 400
}
```
Elasticsearch cannot convert the value to a string.
