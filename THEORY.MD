# Theory

The upstream problem is not a narrow optimization bug. It is that the library currently expresses two different observer contracts through one implementation. The existing `model_observer` contract supports dynamic group membership changes by diffing old and new groups across saves and deletes. To do that, it hooks `post_init` and snapshots groups while Django is hydrating model instances. In real applications, `groups_for_signal` often touches deferred fields or relations, so queryset evaluation can degrade into observer-induced N+1 queries before application code ever reads those instances.

That is a structural fragility, not just an app-specific misuse. The right upstream fix is therefore additive rather than breaking: keep `model_observer` exactly as the dynamic contract, and add a second explicit mode for observers whose group set is immutable for the lifetime of an instance. That mode has now been implemented locally as `static_model_observer`, backed by `StaticModelObserver(ModelObserver)`.

The implementation keeps the existing serializer and channel-send paths intact. It only changes the parts that encode the contract difference:

1. `StaticModelObserver._connect()` skips `post_init`, so queryset hydration does not resolve groups.
2. `StaticModelObserver.prepare_messages()` broadcasts directly to the current group set, without old/new diffing.
3. The public API remains backward compatible because the default `model_observer` behavior is untouched.
4. Documentation explicitly warns that the static mode is only safe when updates cannot move an instance between observer groups.

This direction has been validated against the fork's own test conventions. The new coverage does two things:

1. It proves parity for normal observer behavior in the static mode: create, update, delete, and filtered custom-group subscriptions still work.
2. It proves the performance property directly with dedicated test models and exact query-count assertions: a dynamic observer over a deferred queryset performs `1 + N` queries when the group function touches a deferred field, while the static observer stays at a single query for the same scenario.

The full repository test suite passed after the patch, which means the new observer mode is isolated and upstreamable. The next logical step is to turn this local fork delta into a PR that gives DCRF users a safe opt-in for static group topologies, while preserving the existing dynamic observer semantics for cases that truly need group diffing.
