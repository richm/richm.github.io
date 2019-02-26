How to parse JSON strings with embedded JSON strings
====================================================

Many times I have to deal with JSON documents in which the values are still
more embedded JSON documents.  For example, from (Kubernetes operator framework
registry)[https://github.com/operator-framework/operator-registry#example]

    {
      "csvName": "etcdoperator.v0.9.2",
      "csvJson": "{\"apiVersion\":\"operators.coreos.com/v1alpha1\",...}"
      "object": [
        "{\"apiVersion\":\"apiextensions.k8s.io/v1beta1\",...}",
        "{\"apiVersion\":\"apiextensions.k8s.io/v1beta1\",...}",
        "{\"apiVersion\":\"operators.coreos.com/v1alpha1\",...}",
        "{\"apiVersion\":\"apiextensions.k8s.io/v1beta1\",...}"
      ]
    }

Where the values for `"csvJson"` and the `"object"` array are large JSON
encoded blobs.  I would rather view this as "exploded" JSON:

    {
      "csvJson": {
        "kind": "ClusterServiceVersion",
        "spec": {
          "replaces": "etcdoperator.v0.9.0",
          "displayName": "etcd",
          ...
        },
      },
      "object": [
        {
          "kind": "CustomResourceDefinition",
          "spec": {
            "scope": "Namespaced",
            ...
          },
        },
        {
          "kind": "CustomResourceDefinition",
          "spec": {
            "scope": "Namespaced",
            "version": "v1beta2",
            ...
          }
        },
        ...
      ]
    }

Here is a short python program to print this format:

    import sys,json
    def obj_hook(dd):
      for k,v in dd.iteritems():
        if isinstance(v,basestring) and (v.startswith("{") or v.startswith("[")):
          v = json.loads(v, object_hook=obj_hook)
          dd[k] = v
        elif isinstance(v,list):
          for ii, v1 in enumerate(v):
            if isinstance(v1,basestring) and (v1.startswith("{") or v1.startswith("[")):
              v1 = json.loads(v1, object_hook=obj_hook)
              v[ii] = v1
      return dd
    print json.dumps(json.load(sys.stdin, object_hook=obj_hook), indent=2)

This assumes you are feeding the JSON document from `stdin`.
