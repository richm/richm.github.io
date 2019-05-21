# How to add custom fields to Kibana

This will allow you to do the following with custom log fields added by your
applications:

* Have the fields show up by default in the Discover tab Available Fields
* Use your fields in custom visualizations and dashboards
* Control what data type Kibana uses for your field
* Control the searchable, aggregatable, and other attributes of your fields

The first step is to follow the instructions to
[add your fields to Elasticsearch](add-fields-to-index-template)

[Kibana index pattern documentation](https://www.elastic.co/guide/en/kibana/5.6/index-patterns.html)

Kibana is configured using the undocumented index patterns API.

### Get the existing index pattern file

Grab the existing index pattern file from your Elasticsearch container:

```bash
mkdir index_patterns
cd index_patterns
oc project openshift-logging
for espod in $( oc get pods -l component=es -o jsonpath='{.items[*].metadata.name}' ) ; do
  for ff in $( oc exec -c elasticsearch $espod -- ls /usr/share/elasticsearch/index_patterns ) ; do
    oc exec -c elasticsearch $espod -- cat /usr/share/elasticsearch/index_patterns/$ff > $ff
  done
  break
done
```

### Edit the file to add your field definitions

Edit the files.  You will need to add the definitions of your custom fields to
the `"fields"` field value.  Unfortunately, the format is quite unwieldly - the
`"fields"` field value is a multi-kilobyte, single line string in escaped JSON
format:

    "fields": "[{\"count\": 0, \"name\": \"@timestamp\", ...

It doesn't matter where you add your fields.  It might be easier to add them at
the beginning, depending on your text editor.  A field definition looks like this:

    {\"count\": 0, \"name\": \"hostname\", \"searchable\": true, \"aggregatable\": true, \"readFromDocValues\": true, \"type\": \"string\", \"scripted\": false}

This is the definition of the `hostname` field.  This is a `string` valued field
which is `searchable` and `aggregatable` (can be used in visualizations, aggregations).

    {\"count\": 0, \"name\": \"offset\", \"searchable\": true, \"aggregatable\": true, \"readFromDocValues\": true, \"type\": \"number\", \"scripted\": false}

This is the definition of the `offset` field.  This is a `number` valued field.

You can define your custom types based on these examples and the examples of the
other field definitions in the file.  Note that the data type must correspond to
your Elasticsearch field definition you added above.  For example, if you added a
field `myfield` in Elasticsearch which is a `number` or `integer` type, you cannot
add the field to Kibana as a `string` type.

### Find your index patterns

Identify the index patterns for which you want to add these fields.  The
index patterns will be listed in the Kibana UI on the left hand side of the
`Management -> Index Patterns` page.  Admin users will have `.operations.*`,
`.all`, `.orphaned.*`, and `projects.*`.  Regular users will typically have
one for each namespace/project to which the user is a member of.  If you do
not have an index pattern, create one for your project/namespace by clicking
on `+ Create Index Pattern` and specifying the pattern as `project.MYNAMESPACE.*`
where `MYNAMESPACE` is the name of the project/namespace you want to view
the logs for.

### Add the index pattern to the file

Go back and edit the index pattern file.  There is a field called `"title"`.
The value of this field is the index pattern you want to apply these new
fields to.  For example, if you want to use `project.MYNAMESPACE.*`, the
file should look like this:

    "title": "project.MYNAMESPACE.*"

If you want to apply these fields to multiple index patterns, you will need
to edit the file to change the `"title"` value to the index pattern you want
to apply the fields to.  Or create multiple files, one for each index pattern.
You may want to do that anyway if you will have different fields for
different index patterns.

### Identify the username and get the hash value

Identify the hash of the username.  The index patterns are stored in an
Elasticsearch index for Kibana corresponding to the hash of the username.
For example, for a user named `admin`, for the index pattern `.operations*`,
the field definitions are stored in

    .kibana.d033e22ae348aeb5660fc2140aec35850c4da997/index-pattern/.operations.*

The hash value `d033e22ae348aeb5660fc2140aec35850c4da997` can be obtained
like this:

```bash
get_hash() {
    printf "%s" "$1" | sha1sum | awk '{print $1}'
}

get_hash admin

d033e22ae348aeb5660fc2140aec35850c4da997
```

Similarly, `get_hash richm` produces `e867464f5346bad44cd6d81dce4e69835a47f6f0`

### Apply the file to Elasticsearch

Apply the index pattern file to Elasticsearch.  Use a command line like this:

```bash
cat $fieldfile | \
  oc exec -i -c elasticsearch $espod -- es_util \
    --query=".kibana.$userhash/index-pattern/$indexpattern*" -XPUT --data-binary @- | \
  python -mjson.tool
```

Where the parameters are described below:

* `$fieldfile` - the name of your file containing your field definitions -
  for example `com.redhat.viaq-openshift.index-pattern.json`
* `$espod` - the name of one of your Elasticsearch pods - for example
  `logging-es-data-master-ynokmbwh-4-26dck`
* `$userhash` - the hash of the username from `get_hash` - for example
  `d033e22ae348aeb5660fc2140aec35850c4da997`
* `$indexpattern` - the index pattern you want to apply the fields to - for
  example `project.MYNAMESPACE.*`

Once you do this, you may have to exit Kibana and restart before your fields will
show up in the Available Fields list, and in the `fields` list on the `Management`
`->` `Index Patterns` page.
