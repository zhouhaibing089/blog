---
title: "two lane queue"
date: 2024-02-18T14:22:00-08:00
draft: false
aliases:
- two lane queue
---

As you may know, a controller in kubernetes is a loop which picks up a key from
workqueue and then reconciles the key into some desired state.
While most of such control loops are reactive (based on external events),
some of them require periodic processing (also known as resync). In general,
**periodic processing** can easily fill up the workqueue when the number of keys
is pretty huge.

I have been involved in such scenarios more than once. For instance, we have an
internal API called `HealthMonitor` which is similar to pod readiness probes,
but describes the health status of an application. This API is often consumed in
higher order deployment API to make sure applications are still healthy before
we move into next colo (region, az or cluster). By nature, application health
checks are periodic, so when we want to check the health status of one specific
application, there is a huge chance that it is going to take long as the
controller might be busy dealing with other applications.

Another common case is when controller initially starts, it is pretty common for
the controller to process all the keys again even when they are already in their
desired state. At this time, any new created or updated keys may have to wait
for a long time if the existing keys are holding the queue too long.

In this post, I'm going to share one of the techniques to tackle this problem.
If you experienced something similar, I hope this is helpful to your case, too.

### The idea

In many of the scenarios, what users expected is to have a priority processing
and that basically means some keys are more important than the others:

* Keys that are more actively consumed should have higher priority. For e.g, the
  `HealthMonitor`s that are tied to an active deployment should be prioritized
  over those that are not.
* When controller initially starts, existing keys usually have lower priority
  than the new or updated keys.

So the idea is: When controller picks up a key from workqueue, can the queue
give back the key that needs more urgency?

### Workqueue

Below is a simplified illustration explaining how workqueue works in its minimal
context (`DelayingInterface` and `RateLimitingInterface` are higher order
abstractions but are not necessary for this discussion).

![workqueue](/images/workqueue.svg)

#### Informer

Informer is the event source. It lists/watches from apiserver and maintains a
local cache store (mostly a mirror of objects in apiserver). When there are
changes detected, it calls the corresponding event handler functions to deliver
this change.

#### Controller

Controller is the loop which listens to the events sent by informers and then
reconciles the object to drive into its desired state. Because a reconciliation
loop can take however long it needs, and the informer needs to deliver events
fast, workqueue is employed to bridge them together.

#### Workqueue

[Workqueue][1] primarily offers three functions:

* `Add`: Accepts a new key.
* `Get`: Retrieves a key.
* `Done`: Marks a key as processed.

It also maintains two sets:

* `dirty`: All the keys that need to be processed.
* `processing`: All the keys that are currently being processed.

By default it uses a slice `[]any` (called `queue`) to store all the keys that
are not processed yet.

* A key is either in `queue` or in `processing`.
* All the items in `queue` must also be in `dirty` (not vice versa).
* `dirty` may have an overlap with `processing` if a key is `Add`-ed while it is
  being processed.

`queue` is a FIFO queue which means, keys that are added first are always going
to be processed earlier.

### FIFO?

It appears that if we somehow can swap out the default `[]any` implementation,
we might be able to implement a different strategy. By looking at the places
where `[]any` is used, all it needs are:

* `Push`: Push a key into the queue.
* `Pop`: Pop out a key from the queue.
* `Len`: The number of keys in the queue.

But wait, if a key is already in the queue, we may want to reset its priority on
its next `Add`. This could happen when:

* `dirty` has this key.
* `processing` doesn't have this key.

In order to offer such callback, another function is needed:

* `Touch`: The key is already in queue, but we still want to do something with
  it.

In short, we need something as done in [kubernetes/kubernetes#123347][2] (or use
this [fork][3]).

### TwoLaneQueue

Once it is possible to swap out the default FIFO implementation, the idea of
priortizing something over others is possible. While it is fine to go crazy and
assign keys with an integer priority, reality is that only two are sufficient:
fast or slow.

[knative/pkg][4] started the concept of two lane queue, but its implementation
is at workqueue level (combining two workqueues into a consumer queue). The
problem with that approach is that slow keys are still going to block fast keys
when slow keys take over the whole consumer queue.

Things become much easier at one level down by swapping the underlying queue of
workqueue. This is implemented [here][5]. The idea is: instead of using one
single slice, we can use two lists (`fast` and `slow` accordingly):

1. As long as `fast` is not empty, the items in `fast` is returned first.
1. Each `Add` can change item priority via `Touch` function (move from `fast` to
  `slow`, or from `slow` to `fast`).
1. `list` (linked list) is chosen because of the exact reason above (remove and
  insert).

### The end

While this post doesn't show you an end to end example, I hope it still offers a
good amount of information. Let me know if you disagree or have a different
idea. Thank you for reading.

[1]: https://github.com/kubernetes/client-go/blob/306b201a2d292fd78483fdf6147131926ed25a78/util/workqueue/queue.go#L115
[2]: https://github.com/kubernetes/kubernetes/pull/123347
[3]: https://github.com/zhouhaibing089/workqueue/tree/main
[4]: https://github.com/knative/pkg/blob/main/controller/two_lane_queue.go
[5]: https://github.com/zhouhaibing089/workqueue/tree/main/twolanequeue
