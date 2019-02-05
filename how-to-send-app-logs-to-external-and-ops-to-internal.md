How to send Application logs to external destination and Operations to internal
===============================================================================

I want to configure my OpenShift EFK stack Fluentd to send all Application logs
to an external forward, but keep all Operations logs in the internal
Elasticsearch/Kibana.

The product provides a way to send all logs off the cluster using
[Fluentd secure_forward](https://docs.okd.io/3.11/install_config/aggregate_logging.html#aggregated-fluentd)
"Configuring Fluentd to Send Logs to an External Log Aggregator" but this
method is not very flexible in terms of selecting different output
destinations.  It sends all logs, both Operations and logs from all
Applications, to the secure_forward destination.  This document describes a way
to make those log destinations more flexible.

How it Works
============

The `ES_HOST` and `ES_PORT` options used to be used when we supported the *ES
COPY* feature, now unsupported, to have a different Elasticsearch for
Application logs and Operations logs.  However, you can use these to hack the
code into thinking that there are different output destinations for Application
logs and Operations logs.  The fact that `ES_HOST != OPS_HOST` will "trick"
Fluentd into setting up two different output streams for Applications logs and
Operations logs, but will not actually use `ES_HOST` because we will be setting
up the Applications logs destination to *not* use the Elasticsearch plugin.

First Steps
===========

These are the steps you will need to do first steps, before getting into the
actual detailed routing steps.

* Make a working directory for holding your copies of various configmaps used
  by Fluentd.  You will use this directory for subsequent steps below.  The
  steps below assume the current directory is this directory:

    mkdir working_dir
    cd working_dir

* Make sure you are in the logging project.  This will be `openshift-logging`
  for later versions (3.10 or later) or `logging` for earlier versions:

    oc project openshift-logging
    OR
    oc project logging

* Make your own copy of `output-applications.conf` and `output-operations.conf`
  from a running Fluentd pod:

Assuming `$fluentd_pod` is the name of a running Fluentd pod:

    oc exec $fluentd_pod -- \
      cat /etc/fluent/configs.d/openshift/output-applications.conf > \
      output-applications.conf
    oc exec $fluentd_pod -- \
      cat /etc/fluent/configs.d/openshift/output-operations.conf > \
      output-operations.conf

* Make a copy of the current `logging-fluentd` configmap:

    oc extract configmap/logging-fluentd --to=.

* Make a copy of the original `fluent.conf`:

    if [ ! -f fluent.conf.orig ] ; then
      cp fluent.conf fluent.conf.orig
    fi

* Edit `fluent.conf` to use your customized `output-applications.conf` and
  `output-operations.conf`

Where you see the lines for

    @include configs.d/openshift/output-operations.conf
    @include configs.d/openshift/output-applications.conf

Change these to

    @include configs.d/user/output-operations.conf
    @include configs.d/user/output-applications.conf

Application logs to external and Operations logs to internal EFK
================================================================

This assumes you want to forward all Application logs to your external Fluentd
secure_forward listener, and keep all Operations logs in the internal
Elasticsearch.

* Edit `output-applications.conf`

Remove the following lines from the file:

    <match retry_es>
       @type copy
       @include output-es-retry.conf
    </match>
    
       @include output-es-config.conf
       @include ../dynamic/output-remote-syslog.conf
       @include ../user/output-extra-*.conf

That is, the file should look like this:

    <match **>
       @type copy
       @include ../user/secure-forward.conf
    </match>

That is, there is only the output for your secure-forward config.

* Edit `output-operations.conf`

Remove the following lines from the file:
    
       @include ../user/secure-forward.conf

That is, the file should look like this:

    <match retry_es>
       @type copy
       @include output-es-retry.conf
    </match>
    <match **>
       @type copy
       @include output-es-config.conf
       @include ../dynamic/output-remote-syslog.conf
       @include ../user/output-extra-*.conf
    </match>

That is, it should skip your secure-forward config.

Application logs to external and discard/delete Operations logs
===============================================================

This assumes you want to forward all Application logs to your external Fluentd
secure_forward listener, and discard and delete Operations logs (i.e. you are
not going to use Elasticsearch/Kibana at all).

* Edit `output-applications.conf`

Remove the following lines from the file:

    <match retry_es>
       @type copy
       @include output-es-retry.conf
    </match>
    
       @include output-es-config.conf
       @include ../dynamic/output-remote-syslog.conf
       @include ../user/output-extra-*.conf

That is, the file should look like this:

    <match **>
       @type copy
       @include ../user/secure-forward.conf
    </match>

That is, there is only the output for your secure-forward config.

* Edit `output-operations.conf`

It should contain only the following:

    <match output_ops_tag journal.** system.var.log** mux.ops audit.log** %OCP_FLUENTD_TAGS%>
      @type null
    </match>

The `null` plugin will throw away all of these logs.

Application logs from specific namespaces/pods/containers
=========================================================

**NOTE**: This *only* works when using the `docker --log-driver=json-file` or
when using `CRI-O` as the log driver.  Unfortunely, the `--log-driver=journald`
handling code in Fluentd strips all of the useful tagging information.

This assumes you want to forward all Application logs to your external Fluentd
secure_forward listener, and keep all Operations logs in the internal
Elasticsearch.  In addition, you want to forward only logs from specific
projects and/or pods and/or containers.  First, perform the above steps for
`Application logs to external and Operations logs to internal EFK`.  Then
perform these additional steps.

* Create a file called `filter-post-retag-apps.conf`

This file will contain some filters and matches based on the criteria you want
to use for routing.

* Create filters/retaggers

The tag routing is based on field names, and those field names must be top
level fields, not nested fields, so we'll have to create some top level fields
to use.

    <filter kubernetes.**>
      @type record_transformer
      enable_ruby
      <record>
        kubernetes_namespace_name ${kubernetes.namespace_name}
        kubernetes_pod_name ${kubernetes.pod_name}
        kubernetes_container_name ${kubernetes.container_name}
      </record>
    </filter>

This will add 3 new top level fields to the record for the namespace, pod, and
container names, which will be used to retag the records for routing.  Then use
a custom `rewrite_tag_filter`.

    <match kubernetes.**> # match all kubernetes container logs
      @type rewrite_tag_filter
      @label @OUTPUT
      rewriterule1 kubernetes_namespace_name ^this-is-my-project$ output_tag
      rewriterule2 kubernetes_pod_name ^my-app output_tag
      rewriterule3 kubernetes_container_name mongodb$ output_tag
      rewriterule4 message .+ output_ops_tag
    </match>

The rules are applied in order.  Once a rule is matched, no further processing
is done.  The first rule says "If the namespace name exactly matches the string
'this-is-my-project', route to the output destination for Application logs".
The second rule says "If the pod name begins with the string 'my-app', route to
the output destination for Application logs".  The third rule says "If the
container name ends with 'mongodb', route to the output destination for
Application logs".  The fourth rule is the "default" rule which describes what
to do with all logs that didn't match any of the other rules.  It says "If the
record has a 'message' field (all records should have a 'message' field), route
to the destination for Operations logs".

* Edit `fluent.conf` to include your `filter-post-retag-apps.conf`

Find these two lines in `fluent.conf`

      @include configs.d/openshift/filter-viaq-data-model.conf
      @include configs.d/openshift/filter-post-*.conf

Add the following between these two lines:

      @include configs.d/user/filter-post-retag-apps.conf

so that the result looks like this:

      @include configs.d/openshift/filter-viaq-data-model.conf
      @include configs.d/user/filter-post-retag-apps.conf
      @include configs.d/openshift/filter-post-*.conf

Last Steps
==========

These are the last steps to perform after performing all of the First Steps and
Routing Specific steps.

* Recreate the logging-fluentd configmap with your new files:

    oc delete configmap logging-fluentd
    oc create configmap logging-fluentd --from-file=.

* Set `ES_HOST` and restart *all* Fluentd pods:

    oc set env daemonset/logging-fluentd ES_HOST=value-is-not-used

Setting `ES_HOST` to a value other than its current value will also trigger all
Fluentd pods to be restarted.  If you have already changed the value of
`ES_HOST` and just want to restart all Fluentd pods, then relabel:

    oc label nodes -l logging-infra-fluentd=true logging-infra-fluentd=false
    # wait for all pods to stop
    oc label nodes -l logging-infra-fluentd=false logging-infra-fluentd=true
