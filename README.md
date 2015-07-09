#elasticsearch-kubernetes
Elasticsearch (1.6.0) cluster on top of Kubernetes made easy.

This work is based on [pires/kubernetes-elasticsearch-cluster](https://github.com/pires/kubernetes-elasticsearch-cluster). It is a merged, simpler version that servers as loadbalancer, master and data node.

## Pre-requisites

* Kubernetes cluster (tested with v0.19.0 on top of [Baremetal machines + CoreOS](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/getting-started-guides/coreos/bare_metal_offline.md)
* `kubectl` configured to access your cluster master API Server

## Test

### Deploy

```
kubectl create -f service-account.yaml
kubectl create -f elasticsearch-service.yaml
kubectl create -f elasticsearch-master-controller.yaml
```

Wait until `elasticsearch-master-controller` is provisioned

### Validate

I leave to you the steps to validate the provisioned pods, but first step is to wait for containers to be in ```RUNNING``` state and check the logs of the master (as in Elasticsearch):

```
kubectl get pods
```

You should see something like this:

```
$ kubectl get pods
NAME                         READY     REASON    RESTARTS   AGE
elasticsearch-chlp5          1/1       Running   0          5m
```

Copy pod identifier and check the logs:

```
kubectl logs elasticsearch-chlp5 elasticsearch-master
```

You should see something like this:

```
*$ kubectl logs elasticsearch-chlp5                                                                               
log4j:WARN No such property [maxBackupIndex] in org.apache.log4j.DailyRollingFileAppender.
log4j:WARN No such property [maxBackupIndex] in org.apache.log4j.DailyRollingFileAppender.
log4j:WARN No such property [maxBackupIndex] in org.apache.log4j.DailyRollingFileAppender.
[2015-07-09 17:12:50,164][WARN ][bootstrap                ] Unable to lock JVM memory (ENOMEM). This can result in part of the JVM being swapped out. Increase RLIMIT_MEMLOCK (ulimit).
[2015-07-09 17:12:50,395][INFO ][node                     ] [Shadow-Hunter] version[1.6.0], pid[1], build[cdd3ac4/2015-06-09T13:36:34Z]
[2015-07-09 17:12:50,396][INFO ][node                     ] [Shadow-Hunter] initializing ...
[2015-07-09 17:12:50,985][INFO ][plugins                  ] [Shadow-Hunter] loaded [cloud-kubernetes], sites []
[2015-07-09 17:12:51,099][INFO ][env                      ] [Shadow-Hunter] using [1] data paths, mounts [[/data (/dev/sda1)]], net usable_space [11.8gb], net total_space [15.9gb], types [btrfs]
[2015-07-09 17:12:55,362][INFO ][node                     ] [Shadow-Hunter] initialized
[2015-07-09 17:12:55,362][INFO ][node                     ] [Shadow-Hunter] starting ...
[2015-07-09 17:12:55,733][INFO ][transport                ] [Shadow-Hunter] bound_address {inet[/0:0:0:0:0:0:0:0:9300]}, publish_address {inet[/10.244.1.3:9300]}
[2015-07-09 17:12:55,771][INFO ][discovery                ] [Shadow-Hunter] elasticsearch/7cpzZ8ZPRzifvJhyls7bNg
[2015-07-09 17:13:02,464][INFO ][cluster.service          ] [Shadow-Hunter] new_master [Shadow-Hunter][7cpzZ8ZPRzifvJhyls7bNg][elasticsearch-chlp5][inet[/10.244.1.3:9300]]{master=true}, reason: zen-disco-join (elected_as_master)
[2015-07-09 17:13:02,510][INFO ][http                     ] [Shadow-Hunter] bound_address {inet[/0:0:0:0:0:0:0:0:9200]}, publish_address {inet[/10.244.1.3:9200]}
[2015-07-09 17:13:02,511][INFO ][node                     ] [Shadow-Hunter] started
[2015-07-09 17:13:02,588][INFO ][gateway                  ] [Shadow-Hunter] recovered [0] indices into cluster_state
[2015-07-09 17:14:47,080][INFO ][cluster.service          ] [Shadow-Hunter] added {[Ebenezer Laughton][ReSUZAkPQ768m1LJYKnpwg][elasticsearch-usyb5][inet[/10.244.103.5:9300]]{master=true},}, reason: zen-disco-receive(join from node[[Ebenezer Laughton][ReSUZAkPQ768m1LJYKnpwg][elasticsearch-usyb5][inet[/10.244.103.5:9300]]{master=true}])
[2015-07-09 17:14:47,259][INFO ][cluster.service          ] [Shadow-Hunter] added {[Hannah Levy][9h5NbVAyS6yrmtJiB-DocQ][elasticsearch-rqw9x][inet[/10.244.83.4:9300]]{master=true},}, reason: zen-disco-receive(join from node[[Hannah Levy][9h5NbVAyS6yrmtJiB-DocQ][elasticsearch-rqw9x][inet[/10.244.83.4:9300]]{master=true}])
```

### Scale

Scaling each type of node to handle your cluster is as easy as:

```
kubectl scale --replicas=3 rc elasticsearch
```

### Access the service

*Don't forget* that services in Kubernetes are only acessible from containers in the cluster. For different behavior you should configure the creation of an external-loadbalancer, in your service. That's out of scope of this document, for now.

```
kubectl get service elasticsearch
```

You should see something like this:

```
$ kubectl get service
NAME            LABELS                                               SELECTOR                                             IP(S)          PORT(S)
elasticsearch   clustername=proofofconcept,component=elasticsearch   clustername=proofofconcept,component=elasticsearch   10.100.5.176   9300/TCP
                                                                                                                                         9200/TCP
kubernetes      component=apiserver,provider=kubernetes              <none>                                               10.100.0.2     443/TCP
kubernetes-ro   component=apiserver,provider=kubernetes              <none>                                               10.100.0.1     80/TCP
```

From any host on your cluster (that's running `kube-proxy`):

```
curl http://10.100.5.176:9200
```

This should be what you see:

```json
{
  "status" : 200,
  "name" : "Termagaira",
  "cluster_name" : "elasticsearch-k8s",
  "version" : {
    "number" : "1.6.0",
    "build_hash" : "cdd3ac4dde4f69524ec0a14de3828cb95bbb86d0",
    "build_timestamp" : "2015-06-09T13:36:34Z",
    "build_snapshot" : false,
    "lucene_version" : "4.10.4"
  },
  "tagline" : "You Know, for Search"
}
```

Or if you want to see cluster information:

```
curl http://10.100.5.176:9200/_cluster/health?pretty
```

This should be what you see:

```json
{
  "cluster_name" : "elasticsearch-k8s",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0
}
```
