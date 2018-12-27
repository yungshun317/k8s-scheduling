# Scheduling
* Use label selectors to schedule Pods.
* Understand the role of DaemonSets.
* Understand how resource limits can affect Pod scheduling.
* Understand how to run multiple schedulers and how to configure Pods to use them.
* Manually schedule a pod without a scheduler.
* Display scheduler events.
* Know how to configure the Kubernetes scheduler.

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

## Configure multiple schedulers
Kubernetes ships with a default scheduler. However, you can run multiple schedulers simultaneously alongside the default scheduler and instruct Kubernetes what scheduler to use for each of your pods. 

### Define a Deployment for the scheduler
1. The name of the scheduler specified as an argument to the scheduler command in the container spec should be unique. This is the name that is matched against the value of the optional spec.schedulerName on Pods, to determine whether this scheduler is responsible for scheduling a particular Pod.
2. We created a dedicated service account my-scheduler and bind the cluster role system:kube-scheduler to it so that it can acquire the same privileges as kube-scheduler.
3. Package your scheduler binary into a container image first. In my case, yungshun317/my-kube-scheduler:1.0.

$ kubectl create -f scheduler.yml
serviceaccount/my-scheduler created
clusterrolebinding.rbac.authorization.k8s.io/my-scheduler-as-kube-scheduler created
deployment.apps/my-scheduler created

$ kubectl get pods --namespace=kube-system
NAME                                 READY   STATUS    RESTARTS   AGE
my-scheduler-59dd458d79-nhm59        1/1     Running   0          3m59s

To run multiple-scheduler with leader election enabled, you must do the following:
1. Update the fields commented in scheduler.yml.
2. If RBAC is enabled on the cluster, you must update the system:kube-scheduler cluster role. Add your scheduler name to the resouceNames of the rule applied for endpoints resources. As:

$ kubectl edit clusterrole system:kube-scheduler
- apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    annotations:
      rbac.authorization.kubernetes.io/autoupdate: "true"
    labels:
      kubernetes.io/bootstrapping: rbac-defaults
    name: system:kube-scheduler
  rules:
  - apiGroups:
    - ""
    resourceNames:
    - kube-scheduler
    # Add your scheduler.
    - my-scheduler
    resources:
    - endpoints
    verbs:
    - delete
    - get
    - patch
    - update

clusterrole.rbac.authorization.k8s.io/system:kube-scheduler edited

### Specify schedulers for Pods
$ kubectl create -f pod-without-schedulername.yml 
pod/no-annotation created
$ kubectl create -f pod-with-default-scheduler.yml
pod/annotation-default-scheduler created
$ kubectl create -f pod-with-my-scheduler.yml
pod/annotation-second-scheduler created

## Static Pods
Static Pods are managed directly by kubelet daemon on a specific node, without the API server observing it.

### Static Pod creation
1. Configuration files
  1. (Deprecated) Use $ kubelet --pod-manifest-path=<directory> to start kubelet daemon.
  2. Modify the staticPodPath: <directory> field in the configuration file. By default, the config.yaml file is located under the /var/lib/kubelet directory and the staticPodPath is set to /etc/kubernetes/manifests, so just place the static-pod.yml under this directory and run

$ systemctl restart kubelet

$ kubectl get po
NAME                    READY   STATUS    RESTARTS   AGE
static-web-k8s-master   1/1     Running   0          58s

2. HTTP: kubelet periodically downloads a file specified by --manifest-url=<URL> argument and interprets it as a json/yaml file with a pod definition. It works the same as --pod-manifest-path=<directory>.

### Dynamic addition and removal of static Pods
Note that the static-web-k8s-master Pod in our example is just a mirror Pod created by kubelet. We can't remove it using kubectl and, if stopped by docker stop, kubelet will restart it automatically.

By default, kubelet periodically scans the configured directory.
 
$ sudo rm /etc/kubernetes/manifests/static-pod.yml

$ kubectl get po
No resources found.

# Scheduler events
1. Use get events command.

$ kubectl get events
LAST SEEN   TYPE     REASON                    KIND   MESSAGE
31m         Normal   Starting                  Node   Starting kubelet.
31m         Normal   NodeHasSufficientMemory   Node   Node k8s-master status is now: NodeHasSufficientMemory
31m         Normal   NodeHasNoDiskPressure     Node   Node k8s-master status is now: NodeHasNoDiskPressure
31m         Normal   NodeHasSufficientPID      Node   Node k8s-master status is now: NodeHasSufficientPID
31m         Normal   NodeAllocatableEnforced   Node   Updated Node Allocatable limit across pods
14s         Normal   Scheduled                 Pod    Successfully assigned default/no-annotation to k8s-worker-2
3s          Normal   Pulled                    Pod    Container image "k8s.gcr.io/pause:2.0" already present on machine
3s          Normal   Created                   Pod    Created container
2s          Normal   Started                   Pod    Started container

$ kubectl get events --field-selector involvedObject.name=no-annotation
LAST SEEN   TYPE     REASON      KIND   MESSAGE
3m27s       Normal   Scheduled   Pod    Successfully assigned default/no-annotation to k8s-worker-2
3m16s       Normal   Pulled      Pod    Container image "k8s.gcr.io/pause:2.0" already present on machine
3m16s       Normal   Created     Pod    Created container
3m15s       Normal   Started     Pod    Started container

2. Use kubectl describe to get events only for a Pod.

$ kubectl describe pod no-annotation | grep -A20 Events
Events:
  Type    Reason     Age    From                   Message
  ----    ------     ----   ----                   -------
  Normal  Scheduled  5m37s  default-scheduler      Successfully assigned default/no-annotation to k8s-worker-2
  Normal  Pulled     5m26s  kubelet, k8s-worker-2  Container image "k8s.gcr.io/pause:2.0" already present on machine
  Normal  Created    5m26s  kubelet, k8s-worker-2  Created container
  Normal  Started    5m25s  kubelet, k8s-worker-2  Started container

3. The log file of kube-scheduler is placed under the /var/log/containers directory on the master node.

## Configure the Kubernetes scheduler
The manifest file is /etc/kubernetes/manifests/kube-scheduler.yaml. Configure by modifying flags in the .spec.containers.command field. 
