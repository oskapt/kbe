+++
title = "Service Discovery"
subtitle = "Kubernetes service discovery by example"
date = "2017-05-09"
url = "/sd/"
+++

Service discovery is the process of figuring out how to connect to a [service](/service/).
While there is a service discovery option based on [environment variables](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#environment-variables) available,
the DNS-based service discovery is preferable. Note that DNS is a [cluster add-on](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/README.md) so make sure your Kubernetes distribution provides for one or install it yourself.

Let's create a [service](https://github.com/mhausenblas/kbe/blob/master/specs/sd/svc.yaml) named
`thesvc` and an [RC](https://github.com/mhausenblas/kbe/blob/master/specs/sd/rc.yaml) supervising
some pods along with it:

```bash
$ kubectl create -f https://raw.githubusercontent.com/mhausenblas/kbe/master/specs/sd/rc.yaml

$ kubectl create -f https://raw.githubusercontent.com/mhausenblas/kbe/master/specs/sd/svc.yaml
```

Now we want to connect to the `thesvc` service from within the cluster, say, from another service.
To simulate this, we create a [jump pod](https://github.com/mhausenblas/kbe/blob/master/specs/sd/jumpod.yaml)
in the same namespace (`default`, since we didn't specify anything else):

```bash
$ kubectl create -f https://raw.githubusercontent.com/mhausenblas/kbe/master/specs/sd/jumpod.yaml
```

The DNS add-on will make sure that our service `thesvc` is available via the FQDN
`thesvc.default.svc.cluster.local` from other pods in the cluster. Let's look up
the IP the service is available on using a DNS
lookup tool called `dig` which is available on the jump pod:

```bash
$ kubectl exec jumpod -c shell -i -t -- dig thesvc.default.svc.cluster.local

; <<>> DiG 9.9.4-RedHat-9.9.4-38.el7_3.3 <<>> thesvc.default.svc.cluster.local
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32925
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;thesvc.default.svc.cluster.local. IN   A

;; ANSWER SECTION:
thesvc.default.svc.cluster.local. 30 IN A       172.30.192.196

;; Query time: 2 msec
;; SERVER: 192.168.99.100#53(192.168.99.100)
;; WHEN: Wed May 10 09:15:43 UTC 2017
;; MSG SIZE  rcvd: 66
```

The answer to the DNS query tells us that the service is available via the cluster
IP `172.30.192.196`. We can directly connect to and consume the service (
in the same namespace) like so:

 ```bash
 $ kubectl exec jumpod -c shell -i -t -- curl http://thesvc/info
{"host": "thesvc", "version": "0.5.0", "from": "172.17.0.5"}
```

Note that the IP address `172.17.0.5` above is the cluster-internal IP address
of the jump pod.

To access a service that is deployed in a different namespace than the one you're
accessing it from, use a FQDN in the form `$SVC.$NAMESPACE.svc.cluster.local`.

Let's see how that works by creating:

1. a [namespace](https://github.com/mhausenblas/kbe/blob/master/specs/sd/other-ns.yaml) `other`
1. a [service](https://github.com/mhausenblas/kbe/blob/master/specs/sd/other-svc.yaml) `thesvc` in namespace `other`
1. an [RC](https://github.com/mhausenblas/kbe/blob/master/specs/sd/other-rc.yaml) supervising the pods, also in namespace `other`

If you're not familiar with namespaces, check out the [namespace examples](/ns/) first.

```bash
$ kubectl create -f https://raw.githubusercontent.com/mhausenblas/kbe/master/specs/sd/other-ns.yaml

$ kubectl create -f https://raw.githubusercontent.com/mhausenblas/kbe/master/specs/sd/other-rc.yaml

$ kubectl create -f https://raw.githubusercontent.com/mhausenblas/kbe/master/specs/sd/other-svc.yaml
```

We're now in the position to consume the service `thesvc` in namespace `other` from the
`default` namespace (again via the jump pod):

 ```bash
$ kubectl exec jumpod -c shell -i -t -- curl http://thesvc.other/info
{"host": "thesvc.other", "version": "0.5.0", "from": "172.17.0.5"}
```

Summing up, DNS-based service discovery provides a flexible and generic way to
connect to services across the cluster.
