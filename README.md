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

In order to get the exterbal IP, you need to run the following command:

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

### Delete your deployment

```bash
$ kubectl delete -f olricd.yaml
```

## Contributions

Please don't hesitate to fork the project and send a pull request or just e-mail me to ask questions and share ideas.

## License

The Apache License, Version 2.0 - see LICENSE for more details.
