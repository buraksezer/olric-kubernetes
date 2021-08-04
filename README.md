# olric-kubernetes

This repository provides resources to deploy an Olric based application on Kubernetes. For the sake of simplicity, 
we prefer to use `olricd` to deploy Olric on Kubernetes.

The following guide is tested on a Kubernetes cluster which is created via Minikube.

## Usage

## Install Minikube

Install Minikube on your host system: [https://kubernetes.io/docs/tasks/tools/install-minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)

## Install Kubectl

Install Kubectl on your host system: [https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Deploy Olric on Kubernetes

```bash
$ kubectl apply -f olricd.yaml
```

Alternatively, 

```bash
$ kubectl apply -f https://raw.githubusercontent.com/buraksezer/olric-kubernetes/master/olricd.yaml
```

This command will create a three-node Olric cluster. Check it:

```bash
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
olricd-6cd9496bbc-bg8wf   1/1     Running   0          14s
olricd-6cd9496bbc-nd9gv   1/1     Running   0          14s
olricd-6cd9496bbc-zth59   1/1     Running   0          14s
```

### Logs

Check the logs:

```bash
$ kubectl logs olricd-6cd9496bbc-bg8wf

2020/10/03 17:34:50 [olricd] pid: 1 has been started on 172.18.0.6:3320
2020/10/03 17:34:50 [INFO] Service discovery plugin is enabled, provider: k8s
2020/10/03 17:34:50 [DEBUG] discover: Using provider "k8s"
2020/10/03 17:34:50 [DEBUG] memberlist: Initiating push/pull sync with: 172.18.0.6:3322
2020/10/03 17:34:50 [DEBUG] memberlist: Stream connection from=172.18.0.6:45462
2020/10/03 17:34:50 [DEBUG] memberlist: Initiating push/pull sync with: 172.18.0.5:3322
2020/10/03 17:34:50 [DEBUG] memberlist: Initiating push/pull sync with: 172.18.0.4:3322
2020/10/03 17:34:50 [INFO] Join completed. Synced with 3 initial nodes => olric.go:306
2020/10/03 17:34:50 [INFO] Olric bindAddr: 172.18.0.6, bindPort: 3320 => olric.go:378
2020/10/03 17:34:50 [INFO] Memberlist bindAddr: 172.18.0.6, bindPort: 3322 => olric.go:386
2020/10/03 17:34:50 [INFO] Cluster coordinator: 172.18.0.4:3320 => olric.go:390
2020/10/03 17:34:50 [INFO] Node name in the cluster: 172.18.0.6:3320 => olric.go:603
2020/10/03 17:34:50 [INFO] Node joined: 172.18.0.4:3320 => routing.go:364
2020/10/03 17:34:50 [INFO] Node joined: 172.18.0.5:3320 => routing.go:364
2020/10/03 17:34:50 [INFO] Routing table has been pushed by 172.18.0.4:3320 => routing.go:504
2020/10/03 17:35:38 [DEBUG] memberlist: Initiating push/pull sync with: 172.18.0.5:3322
2020/10/03 17:35:46 [INFO] Routing table has been pushed by 172.18.0.4:3320 => routing.go:504
2020/10/03 17:36:08 [DEBUG] memberlist: Initiating push/pull sync with: 172.18.0.5:3322
```

### Scale up/down

In order to scale up/down your Olric cluster manually:

```bash
$ kubectl scale deployment --replicas=10 olricd
```

This command will create 7 more Olric nodes and some partitions and their backups (with their content, of course) will move to their new owners.

```bash
2020/10/03 11:00:10 [INFO] Routing table has been pushed by 172.18.0.5:3320 => routing.go:504
...
2020/10/03 11:00:08 [INFO] Received dmap (backup:false): olric-load-test on PartID: 242 => rebalancer.go:305
2020/10/03 11:00:08 [INFO] Received dmap (backup:false): olric-load-test on PartID: 266 => rebalancer.go:305
...
2020/10/03 11:00:10 [INFO] Moving dmap: olric-load-test (backup: false) on PartID: 157 to 172.18.0.12:3320 => rebalancer.go:174
```

If you scale down the cluster, you may lose some data. It depends on Olric configuration and how many nodes are removed from the cluster.

### Accessing the cluster

There are two ways to access the cluster.

Firstly, you can use `olric-debug` container to access the cluster from Kubernetes environment. 

```bash
$ kubectl apply -f olric-debug.yaml
```

Alternatively,

```bash
$ kubectl apply -f https://raw.githubusercontent.com/buraksezer/olric-kubernetes/master/olric-debug.yaml
```


Verify whether olric-debug pod works or not:

```bash
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
...
olric-debug               1/1     Running   0          54s
...
```

Get a shell to the running container:

```bash
kubectl exec -it olric-debug -- /bin/sh
```

Now you have a running Alpine Linux setup on Kubernetes. It includes **olric-cli**, **olric-load** and **olric-stats** commands.

```bash
/go/src/github.com/buraksezer/olric # olric-cli -a olricd.default.svc.cluster.local:3320
[olricd.default.svc.cluster.local:3320] » use users
use users
[olricd.default.svc.cluster.local:3320] » put buraksezer {"_id": "06054057", "name": "Burak", "surname": "Sezer", "job": "Engineer"}
[olricd.default.svc.cluster.local:3320] » get buraksezer
{"_id": "06054057", "name": "Burak", "surname": "Sezer", "job": "Engineer"}
[olricd.default.svc.cluster.local:3320] »
```

The second way is exposing an external IP address to access the Olric cluster:

```
$ kubectl expose deployment olricd --type=LoadBalancer --name=olricd-external-lb
```

See the services on your setup:

```bash
$ kubectl get services

NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
kubernetes           ClusterIP      10.96.0.1       <none>        443/TCP                         2d
memberlist           ClusterIP      None            <none>        3322/TCP                        15m
olricd               ClusterIP      10.110.245.55   <none>        3320/TCP                        15m
olricd-external-lb   LoadBalancer   10.98.198.52    <pending>     3320:31429/TCP,3322:32681/TCP   7h36m
```

In order to get the external IP, you need to run the following command:

```bash
$ minikube tunnel
```

Run the `kubectl get services` command and get the external IP. Now, you can access the cluster on your host system.

```bash
$ olric-cli -a $EXTERNAL-IP:3320
```

### Stress tests with olric-load

You can use the following command to create some load on your Olric deployment:

```bash
$ olric-load -a $EXTERNAL-IP:3320 -c put -k 1000000 -s msgpack
```

Retrieve the key/value pairs:

```bash
$ olric-load -a $EXTERNAL-IP:3320 -c get -k 1000000 -s msgpack
```

See [Tooling](https://github.com/buraksezer/olric#tooling) document to learn about how you install Olric tools on your system.

[Kubernetes documentation on how you expose external IP address](https://kubernetes.io/docs/tutorials/stateless-application/expose-external-ip-address/)

### Delete your deployment

```bash
$ kubectl delete -f olricd.yaml
```

## Horizontal Pod Autoscaler

See [Horizontal Pod Autoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) first, if you are not familiar with HPA.

First, install `metrics-server`:

```
$ kubectl apply -f olricd-hpa/metrics-server.yaml
```

Then, create a new Olric cluster on Kubernetes:

```
$ kubectl apply -f olricd-hpa/olricd-hpa.yaml
```

Now, you need to create a horizontal pod autoscaler:

```
$ kubectl autoscale deployment olricd-hpa --cpu-percent=50 --min=1 --max=10
```

In order to access the cluster, you can use `olric-debug` deployment. See [Accessing the cluster](#accessing-the-cluster) section. 

To increase load:

```
$ olric-benchmark -a olricd-hpa.default.svc.cluster.local:3320 -s msgpack -T put -r 1000000
```

See current status of the cluster:

```
$ kubectl get hpa
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
olricd-hpa   Deployment/olricd-hpa   157%/50%   1         10        6          3m10s
```

List the pods:

```
$ kubectl get pods 
NAME                          READY   STATUS    RESTARTS   AGE
olric-debug                   1/1     Running   0          104m
olricd-hpa-5b68655c8b-4dngv   1/1     Running   0          3m50s
olricd-hpa-5b68655c8b-5xcln   1/1     Running   0          24s
olricd-hpa-5b68655c8b-99lsr   1/1     Running   0          84s
olricd-hpa-5b68655c8b-9kf6m   1/1     Running   0          3m50s
olricd-hpa-5b68655c8b-cfwfr   1/1     Running   0          84s
olricd-hpa-5b68655c8b-dnjhd   1/1     Running   0          24s
olricd-hpa-5b68655c8b-j8dqd   1/1     Running   0          84s
olricd-hpa-5b68655c8b-q9nl6   1/1     Running   0          3m50s
olricd-hpa-5b68655c8b-tkf6m   1/1     Running   0          24s
olricd-hpa-5b68655c8b-vfkx2   1/1     Running   0          24s
```

The cluster will be scaled up and balanced automatically. See the logs:

```
$ kubectl logs olricd-hpa-5b68655c8b-vfkx2
2021/08/04 20:11:45 [olricd] pid: 1 has been started
2021/08/04 20:11:45 [INFO] Storage engine has been loaded: kvstore => service.go:129
2021/08/04 20:11:46 [INFO] Service discovery plugin is enabled, provider: k8s
2021/08/04 20:11:46 [DEBUG] discover: Using provider "k8s"
2021/08/04 20:11:46 [DEBUG] discover-k8s: ignoring pod "olricd-567647fb66-zskwr", not running: "Pending"
2021/08/04 20:11:46 [DEBUG] memberlist: Initiating push/pull sync with: 10.1.0.61:3322
2021/08/04 20:11:46 [DEBUG] memberlist: Initiating push/pull sync with: 10.1.0.44:3322
2021/08/04 20:11:46 [DEBUG] memberlist: Initiating push/pull sync with: 10.1.0.62:3322
2021/08/04 20:11:46 [INFO] Join completed. Synced with 3 initial nodes => discovery.go:59
2021/08/04 20:11:46 [INFO] Memberlist bindAddr: 10.1.0.63, bindPort: 3322 => routingtable.go:399
2021/08/04 20:11:46 [INFO] Cluster coordinator: 10.1.0.44:3320 => routingtable.go:400
2021/08/04 20:11:46 [INFO] Node name in the cluster: 10.1.0.63:3320 => olric.go:390
2021/08/04 20:11:46 [INFO] Olric bindAddr: 10.1.0.63, bindPort: 3320 => olric.go:396
2021/08/04 20:11:46 [INFO] Replication count is 1 => olric.go:398
2021/08/04 20:11:46 [INFO] Node joined: 10.1.0.44:3320 => routingtable.go:254
2021/08/04 20:11:46 [INFO] Node joined: 10.1.0.61:3320 => routingtable.go:254
2021/08/04 20:11:46 [INFO] Node joined: 10.1.0.62:3320 => routingtable.go:254
2021/08/04 20:11:46 [INFO] Routing table has been pushed by 10.1.0.44:3320 => operations.go:87
2021/08/04 20:11:46 [INFO] Received DMap (kind: Primary): olric-benchmark-test on PartID: 11 => balance.go:160
2021/08/04 20:12:14 [INFO] Routing table has been pushed by 10.1.0.44:3320 => operations.go:87
2021/08/04 20:12:18 [DEBUG] memberlist: Stream connection from=10.1.0.62:42708
2021/08/04 20:12:39 [DEBUG] memberlist: Initiating push/pull sync with: 10.1.0.61:3322
2021/08/04 20:12:40 [DEBUG] memberlist: Stream connection from=10.1.0.44:51408
2021/08/04 20:12:44 [DEBUG] memberlist: Stream connection from=10.1.0.64:51574
2021/08/04 20:12:44 [INFO] Node joined: 10.1.0.64:3320 => routingtable.go:254
2021/08/04 20:12:44 [INFO] Routing table has been pushed by 10.1.0.44:3320 => operations.go:87
2021/08/04 20:12:44 [INFO] Moving DMap: olric-benchmark-test (kind: Primary) on PartID: 11 to 10.1.0.44:3320 => balancer.go:72
2021/08/04 20:12:45 [INFO] Received DMap (kind: Primary): olric-benchmark-test on PartID: 12 => balance.go:160
2021/08/04 20:12:46 [DEBUG] memberlist: Stream connection from=10.1.0.65:36986
2021/08/04 20:12:46 [INFO] Node joined: 10.1.0.65:3320 => routingtable.go:254
2021/08/04 20:12:47 [INFO] Routing table has been pushed by 10.1.0.44:3320 => operations.go:87
2021/08/04 20:12:47 [INFO] Moving DMap: olric-benchmark-test (kind: Primary) on PartID: 12 to 10.1.0.61:3320 => balancer.go:72
2021/08/04 20:12:49 [DEBUG] memberlist: Stream connection from=10.1.0.66:37434
2021/08/04 20:12:49 [INFO] Node joined: 10.1.0.66:3320 => routingtable.go:254
2021/08/04 20:12:49 [INFO] Routing table has been pushed by 10.1.0.44:3320 => operations.go:87
2021/08/04 20:12:49 [INFO] Received DMap (kind: Primary): olric-benchmark-test on PartID: 10 => balance.go:160
2021/08/04 20:12:51 [DEBUG] memberlist: Stream connection from=10.1.0.67:55788
2021/08/04 20:12:51 [INFO] Node joined: 10.1.0.67:3320 => routingtable.go:254
2021/08/04 20:12:51 [INFO] Routing table has been pushed by 10.1.0.44:3320 => operations.go:87
2021/08/04 20:12:51 [INFO] Moving DMap: olric-benchmark-test (kind: Primary) on PartID: 10 to 10.1.0.44:3320 => balancer.go:72
2021/08/04 20:12:51 [ERROR] Failed to move DMap: olric-benchmark-test on PartID: 10 to 10.1.0.44:3320: storage fragmented => balancer.go:78
2021/08/04 20:12:51 [INFO] Received DMap (kind: Primary): olric-benchmark-test on PartID: 12 => balance.go:160
2021/08/04 20:13:09 [DEBUG] memberlist: Initiating push/pull sync with: 10.1.0.65:3322
2021/08/04 20:13:10 [DEBUG] memberlist: Stream connection from=10.1.0.44:52122
2021/08/04 20:13:14 [INFO] Routing table has been pushed by 10.1.0.44:3320 => operations.go:87
2021/08/04 20:13:14 [INFO] Moving DMap: olric-benchmark-test (kind: Primary) on PartID: 10 to 10.1.0.44:3320 => balancer.go:72
2021/08/04 20:13:14 [INFO] Received DMap (kind: Primary): olric-benchmark-test on PartID: 11 => balance.go:160
```

## Contributions

Please don't hesitate to fork the project and send a pull request or just e-mail me to ask questions and share ideas.

## License

The Apache License, Version 2.0 - see LICENSE for more details.
