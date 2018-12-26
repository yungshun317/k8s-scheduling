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

$ kubectl create -f daemonset.yml
daemonset.apps/fluentd-elasticsearch created

$ kubectl get daemonsets --namespace=kube-system 
NAME                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
fluentd-elasticsearch     3         3         3       3            3           <none>                            7m45s

$ kubectl describe ds fluentd-elasticsearch --namespace=kube-system
Name:           fluentd-elasticsearch
Selector:       name=fluentd-elasticsearch
Node-Selector:  <none>
Labels:         k8s-app=fluentd-logging
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  name=fluentd-elasticsearch
  Containers:
   fluentd-elasticsearch:
    Image:      k8s.gcr.io/fluentd-elasticsearch:1.20
    Port:       <none>
    Host Port:  <none>
    Limits:
      memory:  200Mi
    Requests:
      cpu:        100m
      memory:     200Mi
    Environment:  <none>
    Mounts:
      /var/lib/docker/containers from varlibdockercontainers (ro)
      /var/log from varlog (rw)
  Volumes:
   varlog:
    Type:          HostPath (bare host directory volume)
    Path:          /var/log
    HostPathType:  
   varlibdockercontainers:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/docker/containers
    HostPathType:  
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  9m51s  daemonset-controller  Created pod: fluentd-elasticsearch-2b8v5
  Normal  SuccessfulCreate  9m51s  daemonset-controller  Created pod: fluentd-elasticsearch-vv9x8
  Normal  SuccessfulCreate  9m51s  daemonset-controller  Created pod: fluentd-elasticsearch-gvgxn

### Taints & tolerations
The following tolerations are added to DaemonSet Pods automatically according to the related features:
* node.kubernetes.io/not-ready
* node.kubernetes.io/unreachable
* node.kubernetes.io/disk-pressure
* node.kubernetes.io/memory-pressure
* node.kubernetes.io/unschedulable
* node.kubernetes.io/network-unavailable

### Alternatives
* Init scripts: it's certainly possible to run daemon processes by directly starting them on a node (e.g. using init, upstartd, or systemd). This is perfectly fine. However, there are several advantages to running such processes via a DaemonSet:
  * Ability to monitor and manage logs for daemons in the same way as applications.
  * Same config language and tools (e.g. Pod templates, kubectl) for daemons and applications.
  * Running daemons in containers with resource limits increases isolation between daemons from app containers. This can also be achieved by running the daemons in a container but not in a Pod (e.g. start directly via Docker).
* Bare Pods
* Static Pods
* Deployments

In conclusion, use a Deployment for stateless services, like frontends, where scaling up and down the number of replicas and rolling out updates are more important than controlling exactly which host the Pod runs on. Use a DaemonSet when it is important that a copy of a Pod always run on all or certain hosts, and when it needs to start before other Pods.

## Compute resource
CPU and memory are each a resource type. A resource type has a base unit. CPU is specified in units of cores, and memory is specified in units of bytes.

CPU and memory are collectively referred to as compute resources, or just resources. Compute resources are measurable quantities that can be requested, allocated, and consumed. They are distinct from API resources, such as Pods and Services, which are objects that can be read and modified through the Kubernetes API server.

### Resource requests & limits
* spec.containers[].resources.limits.cpu
* spec.containers[].resources.limits.memory
* spec.containers[].resources.requests.cpu
* spec.containers[].resources.requests.memory

### CPU
Used as the value of the --cpu-shares flag in the docker run command.

The expression 0.1 is equivalent to the expression 100m.

### Memory
Used as the value of the --memory flag in the docker run command.

128974848, 129e6, 129M, 123Mi are roughly the same value.

If a container exceeds its memory limit, it might be terminated. If it is restartable, the kubelet will restart it, as with any other type of runtime failure. If a container exceeds its memory request, it is likely that its Pod will be evicted whenever the node runs out of memory.
