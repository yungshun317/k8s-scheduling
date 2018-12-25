# Scheduling
* Use label selectors to schedule Pods.
* Understand the role of DaemonSets.
* Understand how resource limits can affect Pod scheduling.
* Understand how to run multiple schedulers and how to configure Pods to use them.

## Labels & Selectors
Labels are key/value pairs. Unlike names and UIDs, labels do not provide uniquesness. In general, Kubernetes expects many objects to carry the same label(s). Via a label selector, the client/user can identify a set of objects. The label selector is the core grouping primitive in Kubernetes.

### Equality-based selectors: =, ==, !=
Allow filtering by label keys and values.

environment = production
tier != frontend
environment=production,tier!=frontend

### Set-based selectors: in, notin, exists (only the key identifier)
Allow filtering keys according to a set of values.

environment in (production, qa)
tier notin (frontend, backend)
partition
!partition

### LIST & WATCH filtering
$ kubectl get pods -l environment=production,tier=frontend
$ kubectl get pods -l 'environment in (production), tier in (frontend)'

### Set references in API object
* Some Kubernetes object, such as service and replicationcontrollers, also use label selectors to specify sets of other resources, such as pods. Only equality-based requirement selectors are supported.

selector:
  component: redis

* Newer resources, such as Job, Deployment, Replica Set, and Daemon Set, support set-based requirements.

selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}

## DaemonSet
A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.

Typical uses are:
* Running a cluster storage daemon, such as glusterd, ceph, on each node.
* Running a logs collection daemon on every node, such as fluentd or logstash.
* Running a node monitoring daemon on every node, such as Prometheus Node Explorer, collectd.

### Create a DaemonSet 
* Required fields: apiVersion, kind, metadata. A DaemonSet also needs a .spec section. 
* Pod template: the .spec.template is the required fields in .spec. It's a Pod template, exactly having the same schema as a Pod, except it is nested and does not have an apiVersion or kind.
* Pod selector: you must specify a Pod selector that matches the labels of the .spec.template. .spec.selector is immutable and not compatible with kubectl apply. The .spec.selector contains two fields: matchLabels and matchExpressions. If the .spec.selector is specified, it must match the .spec.template.metadata.labels. 
* Node selector: .spec.template.spec.nodeSelector.
* Node affinity: .spec.template.spec.affinity.
