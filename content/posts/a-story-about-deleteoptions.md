---
title: "A Story about DeleteOptions"
date: 2019-03-30T09:27:34-07:00
draft: false
---

**Note**: The issue has been fixed in https://github.com/kubernetes/kubernetes/pull/76051.

Today, I want to share a story about object deletion in kubernetes federation.
This leads to a better understanding on how object deletion works in kubernetes,
and I believe this may help others to understand it as well.

It all starts with a support request:

> When I delete my namespace on federation, the same namespace in member cluster
> is not deleted, however, cascade deletion does happen for all other resources
> like configmaps.

Let's see how object deletion is supposed to work on federation firstly. I am
going to create a new namespace on federation below.

```bash
$ cat ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fcp-foo
$ kubectl create -f ns.yaml
namespace/fcp-foo created
$ kubectl get ns fcp-foo -o yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2019-03-31T03:56:32Z"
  finalizers:
  - federation.kubernetes.io/delete-from-underlying-clusters
  - orphan
  name: fcp-foo
  resourceVersion: "52956073"
  selfLink: /api/v1/namespaces/fcp-foo
  uid: f8b3ea79-5368-11e9-bc0b-2e8a8f1a8f3c
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
```

If you pay attention enough, you can find that there are two finalizers added:

```yaml
finalizers:
- federation.kubernetes.io/delete-from-underlying-clusters
- orphan
```

Finalizer is the mechanism to ensure proper cleanup in kubernetes, the object
won't disappear from storage as long as there are finalizers remaining. In
another word, every finalizer has its own responsibility for certain resource
cleanup. What is the responsibility of these two finalizers then? Let's take a
look at the [code][1] where these two finalizers are added.

```go
// Ensures that the given object has both FinalizerDeleteFromUnderlyingClusters
// and FinalizerOrphan finalizers.
// We do this so that the controller is always notified when a federation resource is deleted.
// If user deletes the resource with nil DeleteOptions or
// DeletionOptions.OrphanDependents = true then the apiserver removes the orphan finalizer
// and deletion helper does a cascading deletion.
// Otherwise, deletion helper just removes the federation resource and orphans
// the corresponding resources in underlying clusters.
// This method should be called before creating objects in underlying clusters.
func (dh *DeletionHelper) EnsureFinalizers(obj runtime.Object) (
	runtime.Object, error) {
	finalizers := sets.String{}
	hasFinalizer, err := finalizersutil.HasFinalizer(obj, FinalizerDeleteFromUnderlyingClusters)
	if err != nil {
		return obj, err
	}
	if !hasFinalizer {
		finalizers.Insert(FinalizerDeleteFromUnderlyingClusters)
	}
	hasFinalizer, err = finalizersutil.HasFinalizer(obj, metav1.FinalizerOrphanDependents)
	if err != nil {
		return obj, err
	}
	if !hasFinalizer {
		finalizers.Insert(metav1.FinalizerOrphanDependents)
	}
	if finalizers.Len() != 0 {
		glog.V(2).Infof("Adding finalizers %v to %s", finalizers.List(), dh.objNameFunc(obj))
		return dh.addFinalizers(obj, finalizers)
	}
	return obj, nil
}
```

To summarize, the finalizer `federation.kubernetes.io/delete-from-underlying-clusters`
ensures a proper cleanup in member clusters, aka cascade deletion. However, we
do not always want to do cascade deletion, in some cases, we just want the object
to be deleted from federation while leaving the objects in member clusters intact.

Now the questions is: how does the controller known whether the object needs a
cascade deletion or not? This is where finalizer `orphan` plays the role. If the
finalizer `orphan` remains, it means no cascade deletion.

We can also look into the [code][2] where the controller removes finalizers:

```go
// Deletes the resources corresponding to the given federated resource from
// all underlying clusters, unless it has the FinalizerOrphan finalizer.
// Removes FinalizerOrphan and FinalizerDeleteFromUnderlyingClusters finalizers
// when done.
// Callers are expected to keep calling this (with appropriate backoff) until
// it succeeds.
func (dh *DeletionHelper) HandleObjectInUnderlyingClusters(obj runtime.Object) (
	...
	hasFinalizer, err := finalizersutil.HasFinalizer(obj, FinalizerDeleteFromUnderlyingClusters)
	if err != nil {
		return obj, err
	}
	if !hasFinalizer {
		glog.V(2).Infof("obj does not have %s finalizer. Nothing to do", FinalizerDeleteFromUnderlyingClusters)
		return obj, nil
	}
	hasOrphanFinalizer, err := finalizersutil.HasFinalizer(obj, metav1.FinalizerOrphanDependents)
	if err != nil {
		return obj, err
	}
	if hasOrphanFinalizer {
		glog.V(2).Infof("Found finalizer orphan. Nothing to do, just remove the finalizer")
		// If the obj has FinalizerOrphan finalizer, then we need to orphan the
		// corresponding objects in underlying clusters.
		// Just remove both the finalizers in that case.
		finalizers := sets.NewString(FinalizerDeleteFromUnderlyingClusters, metav1.FinalizerOrphanDependents)
		return dh.removeFinalizers(obj, finalizers)
	}
```

We can see,  the controller removes both `orphan` and
`federation.kubernetes.io/delete-from-underlying-clusters` finalizer at the same
time if `orphan` finalizer is still on that object.

You may ask, why the controller adds these two finalizers at the first time, does
that mean it will always do non-cascade deletion? The logic seems to be weird,
but actually it is not, just keep reading..

Let's try to convince ourselves first, if we want to do cascade deletion, then
someone must remove the `orphan` finalizer before the controller works on that
deletion. That has to happen, and must always happen earlier than the time when
controller gets the deletion event.

This proves to be true, when we send a delete request to api server, the
[registry implementation][3] for namespace decides what to do on those finalizers:

```go
// Delete enforces life-cycle rules for namespace termination
func (r *REST) Delete(ctx context.Context, name string, options *metav1.DeleteOptions) (runtime.Object, bool, error) {
	...
	// upon first request to delete, we switch the phase to start namespace termination
	// TODO: enhance graceful deletion's calls to DeleteStrategy to allow phase change and finalizer patterns
	if namespace.DeletionTimestamp.IsZero() {
		err = r.store.Storage.GuaranteedUpdate(
			ctx, key, out, false, &preconditions,
			storage.SimpleUpdate(func(existing runtime.Object) (runtime.Object, error) {
				...
				// Remove orphan finalizer if options.OrphanDependents = false.
				if options.OrphanDependents != nil && *options.OrphanDependents == false {
					// remove Orphan finalizer.
					newFinalizers := []string{}
					for i := range existingNamespace.ObjectMeta.Finalizers {
						finalizer := existingNamespace.ObjectMeta.Finalizers[i]
						if string(finalizer) != metav1.FinalizerOrphanDependents {
							newFinalizers = append(newFinalizers, finalizer)
						}
					}
					existingNamespace.ObjectMeta.Finalizers = newFinalizers
				}
				return existingNamespace, nil
	...
```

Bingo, now we can see, the namespace store removes `orphan` finalizer when
`*options.OrphanDependents == false`. This can explain how the cascade deletion
works on federation:

1. User sends a delete request via `kubectl delete`.
1. APIServer removes the `orphan` finalizer when `options.orphanDependents == false`.
1. Controller then deletes objects in member clusters and removes
`federation.kubernetes.io/delete-from-underlying-clusters` once all the deletions
are successfully made.

If we do not want cascade deletion, we must unset `options.orphanDependents` or
set it as `true`. And in that case, the `orphan` finalizer remains, and the
controller will remove both finalizers without touching member clusters.

Now let's get back to our problem, why it is broken:

```console
$ kubectl delete ns fcp-foo --v=10
...
I0330 22:22:08.278004   44597 request.go:942] Request Body: {"propagationPolicy":"Background"}
I0330 22:22:08.278049   44597 round_trippers.go:419] curl -k -v -XDELETE  -H "Accept: application/json" -H "Content-Type: application/json" -H "User-Agent: kubectl/v1.14.0 (darwin/amd64) kubernetes/641856d" 'https://<apiserver>/api/v1/namespaces/fcp-foo'
...
```

As observed, the delete options is `{"propagationPolicy":"Background"}`, this is
the default value on kubectl v1.14.0. Since it does not specifying
`orphanDependents`, the finalizer `orphan` is not removed, and thus the cascade
deletion does not happen.

But why other resources like configmaps are still fine, do they have a different
registry store implementation? Well, they do!

```go
// Delete removes the item from storage.
func (e *Store) Delete(ctx context.Context, name string, options *metav1.DeleteOptions) (runtime.Object, bool, error) {
	...
	// Handle combinations of graceful deletion and finalization by issuing
	// the correct updates.
	shouldUpdateFinalizers, _ := deletionFinalizersForGarbageCollection(ctx, e, accessor, options)
	// TODO: remove the check, because we support no-op updates now.
	if graceful || pendingFinalizers || shouldUpdateFinalizers {
		err, ignoreNotFound, deleteImmediately, out, lastExisting = e.updateForGracefulDeletionAndFinalizers(ctx, name, key, options, preconditions, obj)
	}
	...
}

// deletionFinalizersForGarbageCollection analyzes the object and delete options
// to determine whether the object is in need of finalization by the garbage
// collector. If so, returns the set of deletion finalizers to apply and a bool
// indicating whether the finalizer list has changed and is in need of updating.
//
// The finalizers returned are intended to be handled by the garbage collector.
// If garbage collection is disabled for the store, this function returns false
// to ensure finalizers aren't set which will never be cleared.
func deletionFinalizersForGarbageCollection(ctx context.Context, e *Store, accessor metav1.Object, options *metav1.DeleteOptions) (bool, []string) {
	...
	shouldOrphan := shouldOrphanDependents(ctx, e, accessor, options)
	...
	newFinalizers := []string{}

	// first remove both finalizers, add them back if needed.
	for _, f := range accessor.GetFinalizers() {
		if f == metav1.FinalizerOrphanDependents || f == metav1.FinalizerDeleteDependents {
			continue
		}
		newFinalizers = append(newFinalizers, f)
	}

	if shouldOrphan {
		newFinalizers = append(newFinalizers, metav1.FinalizerOrphanDependents)
	}
	...
}

// shouldOrphanDependents returns true if the finalizer for orphaning should be set
// updated for FinalizerOrphanDependents. In the order of highest to lowest
// priority, there are three factors affect whether to add/remove the
// FinalizerOrphanDependents: options, existing finalizers of the object,
// and e.DeleteStrategy.DefaultGarbageCollectionPolicy.
func shouldOrphanDependents(ctx context.Context, e *Store, accessor metav1.Object, options *metav1.DeleteOptions) bool {
	// Get default GC policy from this REST object type
	gcStrategy, ok := e.DeleteStrategy.(rest.GarbageCollectionDeleteStrategy)
	var defaultGCPolicy rest.GarbageCollectionPolicy
	if ok {
		defaultGCPolicy = gcStrategy.DefaultGarbageCollectionPolicy(ctx)
	}

	if defaultGCPolicy == rest.Unsupported {
		// return  false to indicate that we should NOT orphan
		return false
	}

	// An explicit policy was set at deletion time, that overrides everything
	if options != nil && options.OrphanDependents != nil {
		return *options.OrphanDependents
	}
	if options != nil && options.PropagationPolicy != nil {
		switch *options.PropagationPolicy {
		case metav1.DeletePropagationOrphan:
			return true
		case metav1.DeletePropagationBackground, metav1.DeletePropagationForeground:
			return false
		}
	}

	// If a finalizer is set in the object, it overrides the default
	// validation should make sure the two cases won't be true at the same time.
	finalizers := accessor.GetFinalizers()
	for _, f := range finalizers {
		switch f {
		case metav1.FinalizerOrphanDependents:
			return true
		case metav1.FinalizerDeleteDependents:
			return false
		}
	}

	// Get default orphan policy from this REST object type if it exists
	if defaultGCPolicy == rest.OrphanDependents {
		return true
	}
	return false
}
```

Put it simply:

1. It removes `orphan` finalizer first.
1. It adds `orphan` finalizer back if:
    1. `options.orphanDependents` is true.
    1. or `options.propagationPolicy` is orphan, otherwise not.
    1. options is nil, and `orphan` finalizer is present.
    1. the default GC policy in DeleteStrategy is `OrphanDependents`

Since the client passes delete options as `propagationPolicy: background`, the
`orphan` finalizer is not added back, which means the controller will do cascade
deletion.

Now we have figured out how it happened, it needs a lot of patience to go through
the codebase. However it is really enjoyable.

[1]: https://github.com/kubernetes/federation/blob/804bb9f3599b9b4c7eb29384921b0bd31cd66de8/pkg/federation-controller/util/deletionhelper/deletion_helper.go#L67-L98
[2]: https://github.com/kubernetes/federation/blob/804bb9f3599b9b4c7eb29384921b0bd31cd66de8/pkg/federation-controller/util/deletionhelper/deletion_helper.go#L110-L129
[3]: https://github.com/kubernetes/kubernetes/blob/afefc0b2c587302034993d381f1c7cfa5c38aa9f/pkg/registry/core/namespace/storage/storage.go#L124-L223
