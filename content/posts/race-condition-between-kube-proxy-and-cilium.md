---
title: "Race condition between kube-proxy and cilium"
date: 2023-05-27T16:00:00-07:00
draft: false
---

This is an investigation story where all pods on one host lost connectivity to
external service. It looks like something below:

```console
$ nsenter --net=/proc/29080/ns/net curl docs.cilium.io -v
> GET / HTTP/1.1
> Host: docs.cilium.io
> User-Agent: curl/7.79.1
> Accept: */*
>
```

Here **29080** is the pid of some pod process, and it appeared that TCP
connection has been established, but somehow it couldn't receive any response.

## About the cluster

The cluster was set up like something below:

```console
$ minikube start --driver=virtualbox --cpus=4 --memory=8g --network-plugin=cni \
    --cni=false --nodes=3
```

and then cilium was installed with:

```console
$ cilium install --kube-proxy-replacement=probe \
    --helm-set ipam.operator.clusterPoolIPv4PodCIDR=172.16.0.0/16 \
    --helm-set bpf.masquerade=true \
    --version=v1.11.17
```

In short, this is a cluster created with minikube, and then cilium was
installed as the CNI plugin, and as kube-proxy replacement whenever possible.

## About the pod network

Below is a basic representation of the network setup for the pod which had
connectivity issues:

![pod-node-network](/images/kube-proxy-cilium.svg)

There is a veth pair with one end in pod and the other end in host. It basically
connects the pod network namespace to host network namespace. There is another
veth pair with both ends in host (`cilium_net` and `cilium_host`).

## tcpdump

To understand why the pod couldn't receive anything back, we can run `tcpdump`
to analyze the traffic flow. To start with, we can try pod network namespace
first:

```console
$ nsenter --net=/proc/29080/ns/net tcpdump -i any host 104.17.32.82 -nn
23:43:46.468314 eth0  Out IP 172.16.0.30.50880 > 104.17.32.82.80: Flags [S], seq 2499115150, win 64860, options [mss 1410,sackOK,TS val 239543031 ecr 0,nop,wscale 7], length 0
23:43:46.478847 eth0  In  IP 104.17.32.82.80 > 172.16.0.30.50880: Flags [S.], seq 881408001, ack 2499115151, win 65535, options [mss 1460], length 0
23:43:46.478867 eth0  Out IP 172.16.0.30.50880 > 104.17.32.82.80: Flags [.], ack 1, win 64860, length 0
23:43:46.478892 eth0  Out IP 172.16.0.30.50880 > 104.17.32.82.80: Flags [P.], seq 1:79, ack 1, win 64860, length 78: HTTP: GET / HTTP/1.1
23:43:46.693746 eth0  Out IP 172.16.0.30.50880 > 104.17.32.82.80: Flags [P.], seq 1:79, ack 1, win 64860, length 78: HTTP: GET / HTTP/1.1
23:43:47.117837 eth0  Out IP 172.16.0.30.50880 > 104.17.32.82.80: Flags [P.], seq 1:79, ack 1, win 64860, length 78: HTTP: GET / HTTP/1.1
23:43:54.143858 eth0  In  IP 104.17.32.82.80 > 172.16.0.30.50880: Flags [S.], seq 881408001, ack 2499115151, win 65535, options [mss 1460], length 0
23:43:54.143877 eth0  Out IP 172.16.0.30.50880 > 104.17.32.82.80: Flags [.], ack 1, win 64860, length 0
```

To make it more readable:

1. 172.16.0.30.50880 (pod) -> 104.17.32.82.80 (server): SYN
2. 104.17.32.82.80 (server) -> 172.16.0.30.50880 (pod): SYN and ACK
3. 172.16.0.30.50880 (pod) -> 104.17.32.82.80 (server): ACK
4. 172.16.0.30.50880 (pod) -> 104.17.32.82.80 (server): PUSH and ACK

Step 4 was retried several times, until there is another SYN and ACK replied
from server. This mostly means that the server didn't see the ACK reply from
step 3, and thus retried step 2, but then what happened to the ACK in step 3?

We can then run the same tcpdump command from host, and see what happened in
host network namespace:

```console
$ tcpdump -i any host 104.17.32.82 -nn
23:53:50.103850 lxc3055a222b7f1 In  IP 172.16.0.30.50992 > 104.17.32.82.80: Flags [S], seq 2399693262, win 64860, options [mss 1410,sackOK,TS val 240146666 ecr 0,nop,wscale 7], length 0
23:53:50.103919 eth0  Out IP 10.0.2.15.50992 > 104.17.32.82.80: Flags [S], seq 2399693262, win 64860, options [mss 1410,sackOK,TS val 240146666 ecr 0,nop,wscale 7], length 0
23:53:50.115561 eth0  In  IP 104.17.32.82.80 > 10.0.2.15.50992: Flags [S.], seq 958144001, ack 2399693263, win 65535, options [mss 1460], length 0
23:53:50.115586 lxc3055a222b7f1 In  IP 172.16.0.30.50992 > 104.17.32.82.80: Flags [.], ack 958144002, win 64860, length 0
23:53:50.115677 lxc3055a222b7f1 In  IP 172.16.0.30.50992 > 104.17.32.82.80: Flags [P.], seq 0:78, ack 1, win 64860, length 78: HTTP: GET / HTTP/1.1
```

From host perspective, we can observe traffic coming from `lxc3055a222b7f1`
which is the host end of pod veth pair. Here the first SYN packet actually was
sent out from `eth0` interface, but none of the other packets were sent out via
`eth0`. The `ACK` packet replied by pod clearly was lost somewhere.

## iptables

Here we can make few guesses:

1. It is more likely dropped by iptables than bpf.
1. It is more likely related to conntrack iptables rules.

The second guess is based on the fact that first SYN packet was sent out without
any issues. To examine this guess, we can find out all the iptables rules which
can drop packets based on conntrack states via:

```console
$ iptables -S | grep DROP | grep "ctstate"
-A KUBE-FIREWALL ! -s 127.0.0.0/8 -d 127.0.0.0/8 -m comment --comment "block incoming localnet connections" -m conntrack ! --ctstate RELATED,ESTABLISHED,DNAT -j DROP
-A KUBE-FORWARD -m conntrack --ctstate INVALID -j DROP
```

The first rule doesn't matter because `-d 127.0.0.0/8` isn't our case, so let's
focus on the second rule.

```console
-A KUBE-FORWARD -m conntrack --ctstate INVALID -j DROP
```

This rule says that if `ctstate` is INVALID, then the packet should be dropped.
We can find the place of this rule fairly straightforward:

```console
# iptables -t filter -nvL KUBE-FORWARD
Chain KUBE-FORWARD (1 references)
 pkts bytes target     prot opt in     out     source               destination
    5   512 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate INVALID
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding rules */ mark match 0x4000/0x4000
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding conntrack rule */ ctstate RELATED,ESTABLISHED
```

Here we can inspect the dropped `pkts/bytes` by running the tests again, and it
isn't difficult to confirm that it is indeed this rule which dropped our
packets.

Starting from here, we have two questions to ask:

1. How does it look like from a good host?
2. Why the conntrack state is INVALID?

## Compare with good host

This is also straightforward. `KUBE-FOREWARD` chain can only be jumpped from
`FORWARD` chain, so let's just compare how does it look like in `FORWARD` chain:

On a bad host:

```console
$ iptables -t filter -nvL FORWARD
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
   11   660 KUBE-PROXY-FIREWALL  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes load balancer firewall */
  169 14804 KUBE-FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding rules */
   11   660 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes service portals */
   11   660 KUBE-EXTERNAL-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes externally-visible service portals */
   11   660 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
   11   660 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0
    7   420 CILIUM_FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cilium-feeder: CILIUM_FORWARD */
```

and on a good host:

```console
$ iptables -t filter -nvL FORWARD
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
 1419  148K CILIUM_FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cilium-feeder: CILIUM_FORWARD */
    0     0 KUBE-PROXY-FIREWALL  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes load balancer firewall */
    0     0 KUBE-FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding rules */
    0     0 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes service portals */
    0     0 KUBE-EXTERNAL-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes externally-visible service portals */
    0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0
```

The difference here is that `CILIUM_FORWARD` is matched first. We can see what's
in `CILIUM_FORWARD` with:

```console
$ iptables -t filter -nvL CILIUM_FORWARD
Chain CILIUM_FORWARD (1 references)
 pkts bytes target     prot opt in     out     source               destination
  642 93820 ACCEPT     all  --  *      cilium_host  0.0.0.0/0            0.0.0.0/0            /* cilium: any->cluster on cilium_host forward accept */
    0     0 ACCEPT     all  --  cilium_host *       0.0.0.0/0            0.0.0.0/0            /* cilium: cluster->any on cilium_host forward accept (nodeport) */
  794 56164 ACCEPT     all  --  lxc+   *       0.0.0.0/0            0.0.0.0/0            /* cilium: cluster->any on lxc+ forward accept */
    0     0 ACCEPT     all  --  cilium_net *       0.0.0.0/0            0.0.0.0/0            /* cilium: cluster->any on cilium_net forward accept (nodeport) */
```

Here, the packet is accepted if the packet is sent from `lxc+`, and thus the
packet won't continue matching with `KUBE-FORWARD`.

## How did it happen

We can look at the code in kube-proxy and cilium, and it won't take too long to
realize that the order of `CILIUM_FORWARD` and `KUBE-FORWARD` relies on which
component starts first initially. In a typical case, if cilium starts later than
kube-proxy, `CILIUM_FORWARD` is going to be prepended and thus takes higher
precedence. However if kube-proxy starts after cilium, then `KUBE-FORWARD` will
take precedence, and in this case, it breaks pod connectivity.

## Why ctstate is INVALID

iptables tracks connection states, but if only part of traffic goes through
iptables, then it won't recognize the connection states correctly. In this case
here, we have `bpf.masquerade=true` configured in cilium. It basically means
cilium actually does snat via bpf directly (in contrast to doing it in
iptables). With that said, when iptables sees there is an ACK without knowing
there was a SYN before, it is considered as INVALID.
