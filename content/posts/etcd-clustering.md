---
title: "etcd clustering step by step"
date: 2023-06-18T14:27:00-07:00
draft: false
---

This is an attempt to document the steps that I tried to understand etcd
clustering. I think by following these steps, we are better suited to understand
how etcd disaster recovery looks like in practice.

## Prerequisites

I added the following into `/etc/hosts` so that I can assign dns names to each
of the etcd instances that I'm about to create.

```console
$ cat /etc/hosts
# Static table lookup for hostnames.
# See hosts(5) for details.
127.0.0.1 etcd-1
127.0.0.2 etcd-2
127.0.0.3 etcd-3
```

## The first instance

We can run the following command to start our first etcd instance:

```console
$ etcd --name=etcd-1 \
    --listen-peer-urls http://127.0.0.1:2380 \
    --listen-client-urls http://127.0.0.1:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380 \
    --initial-advertise-peer-urls http://etcd-1:2380 \
    --advertise-client-urls http://etcd-1:2379
```

The default value for `--name` is `default` which doesn't look great because I
will need to create `n` instances, so I decided to name it as `etcd-{x}`.

`--listen-peer-urls` specifies the address to listen on requests received from
peers (other etcd instances), and `--listen-client-urls` specifies the address
to listen on requests received from clients (for instance, `etcdctl`). I could
have just skipped these two flags, but for consistency and better clarities, I
decided to add them explicitly.

The default value for `--initial-cluster` is `'default=http://localhost:2380'`.
Since I changed the name and also wanted to use a specific dns name, I need to
override it properly.

`--initial-advertise-peer-urls` and `--advertise-client-urls` specifies how this
instance expects other peers and clients to talk to it.

Now let's store some data into it:

```console
$ etcdctl --endpoints=http://etcd-1:2379 put foo bar
OK
```

## Add the second instance

It is natural to just try the same command with few modifications. Let's try it
out:

```console
$ etcd --name=etcd-2 \
    --listen-peer-urls http://127.0.0.2:2380 \
    --listen-client-urls http://127.0.0.2:2379 \
    --initial-cluster etcd-2=http://etcd-2:2380 \
    --initial-advertise-peer-urls http://etcd-2:2380 \
    --advertise-client-urls http://etcd-2:2379
```

The command runs fine, but it just started another new etcd instance which is
not connected to the first instance at all:

```console
$ etcdctl --endpoints=http://etcd-2:2379 get foo
```

This command above gives nothing, which we would expect to see `bar`.

In order to connect to the first instance, we should specify its peer address in
`--initial-cluster`. Let's add `etcd-1` as well:

> You may have to run `rm -drf etcd-2.etcd` to clean up first.

```console
$ etcd --name=etcd-2 \
    --listen-peer-urls http://127.0.0.2:2380 \
    --listen-client-urls http://127.0.0.2:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380 \
    --initial-advertise-peer-urls http://etcd-2:2380 \
    --advertise-client-urls http://etcd-2:2379
```

This takes us further, but it errors crazily with message below:

```
{"level":"warn","ts":"2023-06-17T21:26:46.212008-0700","caller":"rafthttp/stream.go:653","msg":"request sent was ignored by remote peer due to cluster ID mismatch","remote-peer-id":"a3959071884acd0c","remote-peer-cluster-id":"572a811fac16a854","local-member-id":"97ecdd3706cc2d90","local-member-cluster-id":"b2698d4259f33e40","error":"cluster ID mismatch"}
```

It means that this instance tried to create another cluster which isn't expected.
We can fix this by add `--initial-cluster-state existing`.

```console
$ etcd --name=etcd-2 \
    --listen-peer-urls http://127.0.0.2:2380 \
    --listen-client-urls http://127.0.0.2:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380 \
    --initial-advertise-peer-urls http://etcd-2:2380 \
    --advertise-client-urls http://etcd-2:2379 \
    --initial-cluster-state existing
```

This command still failed to bring up the second instance. The error message is:

```console
{"level":"fatal","ts":"2023-06-17T21:30:36.560232-0700","caller":"etcdmain/etcd.go:204","msg":"discovery failed","error":"error validating peerURLs {ClusterID:572a811fac16a854 Members:[&{ID:a3959071884acd0c RaftAttributes:{PeerURLs:[http://etcd-1:2380] IsLearner:false} Attributes:{Name:etcd-1 ClientURLs:[http://etcd-1:2379]}}] RemovedMemberIDs:[]}: member count is unequal","stacktrace":"go.etcd.io/etcd/server/v3/etcdmain.startEtcdOrProxyV2\n\tgo.etcd.io/etcd/server/v3/etcdmain/etcd.go:204\ngo.etcd.io/etcd/server/v3/etcdmain.Main\n\tgo.etcd.io/etcd/server/v3/etcdmain/main.go:40\nmain.main\n\tgo.etcd.io/etcd/server/v3/main.go:31\nruntime.main\n\truntime/proc.go:250"}
```

This basically means that the member count doesn't match. We specified 2 in our
flag `--initial-cluster`, but there is only one member which is `etcd-1`:

```console
$ etcdctl --endpoints=http://etcd-1:2379 member list
a3959071884acd0c, started, etcd-1, http://etcd-1:2380, http://etcd-1:2379, false
```

To fix this, we can add `etcd-2` manually with:

```console
$ etcdctl --endpoints=http://etcd-1:2379 member add etcd-2 --peer-urls http://etcd-2:2380
```

> This is going to disrupt the service status of `etcd-1` due to loss of quorum
> because `etcd-2` is not yet up.

Afterwards, we should be able to bring up the second instance by re-running the
command that failed previously. And we should be able to query from `etcd-2` as
well:

```console
$ etcdctl --endpoints=http://etcd-2:2379 get foo
foo
bar
```

## Add another instance

The step should now be straightforward:

```console
$ etcdctl --endpoints=http://etcd-1:2379 member add etcd-3 --peer-urls http://etcd-3:2380
$ etcd --name=etcd-3 \
    --listen-peer-urls http://127.0.0.3:2380 \
    --listen-client-urls http://127.0.0.3:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-3:2380 \
    --advertise-client-urls http://etcd-3:2379 \
    --initial-cluster-state existing
$ etcdctl --endpoints=http://etcd-3:2379 get foo
foo
bar
```

### Avoid adding members manually

The steps demonstrated abvoe seem to be tedious because we have to add each
member manually before we can start the instance. If we know all the members
beforehand, we can avoid doing that manually. To run the same setup, we can just
do the following:

> You may need to clean up everything we did above by running `rm -drf etcd-*`.

```console
$ etcd --name=etcd-1 \
    --listen-peer-urls http://127.0.0.1:2380 \
    --listen-client-urls http://127.0.0.1:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-1:2380 \
    --advertise-client-urls http://etcd-1:2379
$ etcd --name=etcd-2 \
    --listen-peer-urls http://127.0.0.2:2380 \
    --listen-client-urls http://127.0.0.2:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-2:2380 \
    --advertise-client-urls http://etcd-2:2379
$ etcd --name=etcd-3 \
    --listen-peer-urls http://127.0.0.3:2380 \
    --listen-client-urls http://127.0.0.3:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-3:2380 \
    --advertise-client-urls http://etcd-3:2379
```

Here are the differences:

* Now all the instances are started with a full list of `--initial-cluster`.
* They all start with `--initial-cluster-state new` (the default value).

As part of this change, the first instance won't be able to take traffic right
away because the cluster isn't healthy yet. The cluster is only able to start
serving traffic once the second instance is also up.

But wait, why `--initial-cluster-state new` works here? In our previous attempt,
when we join the second instance without setting it to `existing`, it complains
about `clusterID mismatch`. It turns out that cluster id is actually generated
based on `--initial-cluster`. This means that each instance, when they have the
same value of `--initial-cluster`, they'll come up with the same member id as
well as cluster id. We can examine the cluster id by trying the command above
multiple time and we can expect that cluster id is always the same:

```console
$ etcdctl --endpoints=http://etcd-1:2379 member list -w json | jq '.header.cluster_id'
1176270058202747400
```

> You actually should see the same value if you followed the same steps above.

Fore more information, you may look at [computeMemberId][1] and [genID][2].

## Failures

There are different failure scenarios:

* Minority failure with data.
* Minority failure without data.
* Majority failure with data.
* Majority failure without data.

### Minority failure with data

Let's imagine a case when `etcd-1` (`127.0.0.1`) failed, but its local storage
is still available because it uses remote storage. We can simply provision
another instance and let `etcd-1` resolves to the new IP address. In my
experiment, let's assume the new IP address is `127.0.0.4`. We can make this
change by updating `/etc/hosts`:

```console
$ cat /etc/hosts
# Static table lookup for hostnames.
# See hosts(5) for details.
127.0.0.4 etcd-1
127.0.0.2 etcd-2
127.0.0.3 etcd-3
```

Note that `etcd-1` now points to `127.0.0.4`. We can now start this instance by:

```console
$ etcd --name=etcd-1 \
    --listen-peer-urls http://127.0.0.4:2380 \
    --listen-client-urls http://127.0.0.4:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-1:2380 \
    --advertise-client-urls http://etcd-1:2379
```

### Minority failure without data

In this case, not only the instance is gone, but its data is also lost. To
simulate this case, let's wipe out the data via:

```console
$ rm -drf etcd-1.etcd
```

Now if we try the same command as mentioned above:

```console
$ etcd --name=etcd-1 \
    --listen-peer-urls http://127.0.0.4:2380 \
    --listen-client-urls http://127.0.0.4:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-1:2380 \
    --advertise-client-urls http://etcd-1:2379
```

It is going to fail with:

```console
{"level":"fatal","ts":"2023-06-18T13:16:48.878283-0700","caller":"etcdmain/etcd.go:204","msg":"discovery failed","error":"member a3959071884acd0c has already been bootstrapped","stacktrace":"go.etcd.io/etcd/server/v3/etcdmain.startEtcdOrProxyV2\n\tgo.etcd.io/etcd/server/v3/etcdmain/etcd.go:204\ngo.etcd.io/etcd/server/v3/etcdmain.Main\n\tgo.etcd.io/etcd/server/v3/etcdmain/main.go:40\nmain.main\n\tgo.etcd.io/etcd/server/v3/main.go:31\nruntime.main\n\truntime/proc.go:250"}
```

What this message means is that `etcd-1` has already been bootstrapped in the
past, it won't succeed if we try to bootstrap it again. In order to bootstrap it
successfully, we have to reset its state by removing this member:

```console
$ memberid=$(etcdctl --endpoints=http://etcd-2:2379 member list --hex -w json | jq -r '.members[] | select(.name == "etcd-1") | .ID')
$ etcdctl --endpoints=http://etcd-2:2379 member remove $memberid
```

Now we are back to the original state, which we need to add this member manually
and then start the instance with `--initial-cluster-state existing`:

```console
$ etcdctl --endpoints=http://etcd-2:2379 member add etcd-1 --peer-urls http://etcd-1:2380
$ etcd --name=etcd-1 \
    --listen-peer-urls http://127.0.0.4:2380 \
    --listen-client-urls http://127.0.0.4:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-1:2380 \
    --advertise-client-urls http://etcd-1:2379 \
    --initial-cluster-state existing
```

### Majority failure with data

Let's continue the experimentation, and assume we lost `etcd-2` and `etcd-3`. In
this case, since we still have the data for `etcd-2` and `etcd-3`, we can simply
launch two new instances. In this experiment, let's assume the new IP addresses
for them are `127.0.0.5` and `127.0.0.6` respectively:

```console
$ cat /etc/hosts
# Static table lookup for hostnames.
# See hosts(5) for details.
127.0.0.4 etcd-1
127.0.0.5 etcd-2
127.0.0.6 etcd-3
$ etcd --name=etcd-2 \
    --listen-peer-urls http://127.0.0.5:2380 \
    --listen-client-urls http://127.0.0.5:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-2:2380 \
    --advertise-client-urls http://etcd-2:2379
$ etcd --name=etcd-3 \
    --listen-peer-urls http://127.0.0.6:2380 \
    --listen-client-urls http://127.0.0.6:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-3:2380 \
    --advertise-client-urls http://etcd-3:2379
```

### Majority failure without data

It is clear now that we need to remove the previous dead members when we don't
have its previous data, however we won't be able to do such operations because
we lost too many instances, and at that point of time, no `etcdctl` operations
are possible anymore. This can further be dividied into three different cases:

* There is snapshot saved.
* There is no snapshot, but there is still at least one live instance.
* No snapshot, and no any live instances

#### With snapshot

Let's save a snapshot before we simulate failures:

```console
$ etcdctl --endpoints=http://etcd-3:2379 snapshot save snapshot.db
```

Now let's stop all instances and also clean the data directories:

```console
$ rm -drf etcd-1.etcd
$ rm -drf etcd-2.etcd
$ rm -drf etcd-3.etcd
```

Now we can restore the data directory for each of the instance via:

```console
$ etcdctl snapshot restore snapshot.db \
    --name etcd-1 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-1:2380
$ etcdctl snapshot restore snapshot.db \
    --name etcd-2 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-2:2380
$ etcdctl snapshot restore snapshot.db \
    --name etcd-3 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-3:2380
```

Then we can follow the same steps above just like we have data available:

```console
$ etcd --name=etcd-1 \
    --listen-peer-urls http://127.0.0.4:2380 \
    --listen-client-urls http://127.0.0.4:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-1:2380 \
    --advertise-client-urls http://etcd-1:2379
$ etcd --name=etcd-2 \
    --listen-peer-urls http://127.0.0.5:2380 \
    --listen-client-urls http://127.0.0.5:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-2:2380 \
    --advertise-client-urls http://etcd-2:2379
$ etcd --name=etcd-3 \
    --listen-peer-urls http://127.0.0.6:2380 \
    --listen-client-urls http://127.0.0.6:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-3:2380 \
    --advertise-client-urls http://etcd-3:2379
```

#### With live instance

If we don't have snapshots, but we do still have at least one live instance, we
can still restore this cluster by initializing the state from one of the live
instance. In this experiment, I'm going to stop `etcd-2` and `etcd-3` and also
remove its data directories:

```console
$ rm -drf etcd-2.etcd
$ rm -drf etcd-3.etcd
```

Now we can restart `etcd-1` with `--force-new-cluster`:

```console
$ etcd --name=etcd-1 \
    --listen-peer-urls http://127.0.0.4:2380 \
    --listen-client-urls http://127.0.0.4:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-1:2380 \
    --advertise-client-urls http://etcd-1:2379 \
    --force-new-cluster
```

With this step completed, we can bring back `etcd-2` and `etcd-3` just like what
we experimented earlier - when we set up clustering manually:

```console
$ etcdctl --endpoints=http://etcd-1:2379 member add etcd-2 --peer-urls http://etcd-2:2380
$ etcd --name=etcd-2 \
    --listen-peer-urls http://127.0.0.5:2380 \
    --listen-client-urls http://127.0.0.5:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380 \
    --initial-advertise-peer-urls http://etcd-2:2380 \
    --advertise-client-urls http://etcd-2:2379 \
    --initial-cluster-state existing
$ etcdctl --endpoints=http://etcd-1:2379 member add etcd-3 --peer-urls http://etcd-3:2380
$ etcd --name=etcd-3 \
    --listen-peer-urls http://127.0.0.6:2380 \
    --listen-client-urls http://127.0.0.6:2379 \
    --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380 \
    --initial-advertise-peer-urls http://etcd-3:2380 \
    --advertise-client-urls http://etcd-3:2379 \
    --initial-cluster-state existing
```

If there is no snapshot, nor there is any live instances, then I think we don't
really have any possibilities there.

## Summary

I hope you enjoyed this experimentation so far. This post examined several cases
from manual clustering setup as well as different disaster recovery scenarios. I
wish this offered few insights and also be helpful when you actually need it.

[1]: https://github.com/etcd-io/etcd/blob/6f2a5b710fc13fb427128d09939671560edf680b/server/etcdserver/api/membership/member.go#L63
[2]: https://github.com/etcd-io/etcd/blob/6f2a5b710fc13fb427128d09939671560edf680b/server/etcdserver/api/membership/cluster.go#L232

