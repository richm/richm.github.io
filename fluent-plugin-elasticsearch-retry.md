# Retry Handling with fluent-plugin-elasticsearch

Thanks to @jcantrill et. al. the retry logic in fluent-plugin-elasticsearch has
been greatly improved.  These changes are in version 1.16.1 and later:
<https://github.com/uken/fluent-plugin-elasticsearch/releases/tag/v1.16.1>

* When a bulk index request is rejected, records that can't be processed at all
  (i.e. would clog the pipe and prevent other records from being processed) are
  redirected to @ERROR label processing:
  <https://docs.fluentd.org/v0.12/articles/config-file#@error-label>

* When a record can be retried, it is re-submitted back to fluentd for
  processing.  The primary use case is when a record is rejected due to a
  `429` - bulk index queue is full, or other resource problem in Elasticsearch.
  In this case, clients are supposed to back-off and resubmit the operation at
  a later time, as described here
  <https://www.elastic.co/guide/en/elasticsearch/guide/2.x/_monitoring_individual_nodes.html#_threadpool_section>
  in the `Bulk Rejections` box.

* When a record is rejected with a `409` - `Conflict` - the duplicate record
  is not resubmitted.  This relies on using a unique ID per record, and setting
  `write_operation create` in the fluent-plugin-elasticsearch configuration
  (see below).


# Unique ID per record

All records must be assigned a unique ID.  OpenShift logging does it with the
new `elasticsearch_genid` plugin type provided by fluent-plugin-elasticsearch.
<https://github.com/openshift/origin-aggregated-logging/blob/master/fluentd/configs.d/openshift/filter-post-genid.conf#L3>

    <filter **>
      @type elasticsearch_genid
      hash_id_key viaq_msg_id
    </filter>

You will have to add a section like this to your fluentd config in order to
assign a unique ID to each record.  The `@type elasticsearch` plugin config
should be changed like this:

    @type elasticsearch
    ...
    id_key viaq_msg_id
    write_operation 'create'

This tells the plugin to use the generated ID for the `_id`
<https://www.elastic.co/guide/en/elasticsearch/guide/2.x/_document_metadata.html#_id>
metadata field associated with the record.  This also tells the plugin to use the
`create`
<https://www.elastic.co/guide/en/elasticsearch/guide/2.x/create-doc.html>
operation to add the new record instead of the default `index`, which will
update the record if it exists.  In this case, we want Elasticsearch to return
`409` so that we will not attempt to resubmit this record.

# Retry handling

When fluent-plugin-elasticsearch resubmits a failed record that is a candidate
for a retry (e.g. the request returned a `429` for the record), the record is
resubmitted back into the fluentd record queue for processing.  By default, it
is submitted back to the very beginning of processing, and will go back through
all of your processing pipeline again.  You will usually want to avoid this.
There are two ways to route records so that you can control the reprocessing -
tagging and/or labeling.  You can use either separately or both in conjunction.

## Retry tagging

There is a new parameter in fluent-plugin-elasticsearch - `retry_tag` -
<https://github.com/uken/fluent-plugin-elasticsearch#retry_tag>

    @type elasticsearch
    ...
    retry_tag retry_es

It is best to use a tag that isn't used by anything else and will not match
other filters, to avoid having records reprocessed.  When records are
resubmitted, they will use this tag instead of whatever tag they were using
before.  Then you can have something like this in your fluentd config:

    # handler for initial attempt and retries
    <match retry_es **my_initial_tag**>
      @type elasticsearch
      ...
      retry_tag retry_es
      ... other config ...
    </match>

This is good for a simple case where you don't have much processing, and you
only have one or two matches that don't use `@type copy`.  You can't do this
for `@type copy` because your retried records would be sent again to all of
your copy destinations, causing duplicate records at those destinations.  In
that case, you will need to define a separate `@type elasticsearch` config to
handle only the Elasticsearch retries:

    # handler for retries
    <match retry_es>
      @type elasticsearch
      ...
      retry_tag retry_es
      ... other config ...
    </match>
    # handler for initial attempt and copy destinations
    <match **my_initial_tag**>
      @type copy
      <store>
        @type elasticsearch
        ...
        retry_tag retry_es
        ... other config ...
      </store>
      <store>
        @type secure_forward
        ...
      </store>
    </match>

This is the method used by OpenShift logging as there may be multiple `copy`
destinations specified by the user.  For example:
<https://github.com/openshift/origin-aggregated-logging/blob/master/fluentd/configs.d/openshift/output-operations.conf>
The configuration of the retry fluent-plugin-elasticsearch should be identical
to the initial fluent-plugin-elasticsearch.  This is config duplication, so
you'll need to use some other method to keep it DRY e.g. move all of the common
config to a separate file and `@include` it.

## Retry labeling

You can also specify a label
<https://docs.fluentd.org/v0.12/articles/life-of-a-fluentd-event#labels> and 
<https://docs.fluentd.org/v0.12/articles/routing-examples#input--%3E-filter--%3E-output-with-label>
to route records to.  To do this, add a `@label @LABEL_NAME` to your
fluent-plugin-elasticsearch config:

    @type elasticsearch
    ...
    @label @RETRY_ES

Then add a `<label @RETRY_ES>` section to your config:

    <label @RETRY_ES>
      <match **>
        @type elasticsearch
        ...
        @label @RETRY_ES
      </match>
    </label>

With labeling, you can ensure that the retried record is sent directly to the
`label` section, avoiding the rest of your processing pipeline.

You can use labeling and tagging both:

    @type elasticsearch
    ...
    @label @RETRY_ES
    retry_tag retry_es

Then add a `<label @RETRY_ES>` section to your config:

    <label @RETRY_ES>
      <match retry_es>
        @type elasticsearch
        ...
        @label @RETRY_ES
        retry_tag retry_es
      </match>
    </label>

The configuration of the retry fluent-plugin-elasticsearch should be identical
to the initial fluent-plugin-elasticsearch.  This is config duplication, so
you'll need to use some other method to keep it DRY e.g. move all of the common
config to a separate file and `@include` it.

## Other parameters related to retries

The parameters `max_retry_wait`, `disable_retry_limit`, `retry_limit`, and
`retry_wait` won't be as important when using the configuration specified
above, because the retries will happen internally to the
fluent-plugin-elasticsearch.  In a sense, that code path will be avoided.  They
will still be used in the case where the error is due to connection or network
issues e.g. cannot connect to Elasticsearch.

# Error record handling

Records that have "hard" errors (schema violations, corrupted data, etc.) that
cannot be retried will be sent to the `@ERROR` handler.  Think of this as a
"dead letter queue", for messages that cannot be delivered.  If you add a
`<label @ERROR>` section to your fluentd config, you can handle these records
how you want.  For example:

    <label @ERROR>
      <match **>
        @type file
        path /var/log/fluent/dlq
        time_slice_format %Y%m%d
        time_slice_wait 10m
        time_format %Y%m%dT%H%M%S%z
        compress gzip
      </match>
    </label>

This will write error records to the `dlq` file.  See
<https://docs.fluentd.org/v0.12/articles/out_file> for more information about the
file output.
