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
