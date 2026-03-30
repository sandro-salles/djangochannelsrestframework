# Napkin

- 2026-03-29: Root issue confirmed in upstream library. `ModelObserver` connects `post_init` and eagerly resolves `group_names_for_signal(instance=...)` while Django is hydrating queryset rows. If the group function touches deferred fields or relations, queryset evaluation degrades into N+1 queries.
- 2026-03-29: Local app-side workaround validated the right upstream direction: add an explicit static-group observer mode instead of changing `model_observer` semantics. This preserves backward compatibility and gives users an opt-in escape hatch for immutable group topologies.
- 2026-03-29: For this repo, testing style is direct `pytest` with consumer classes declared inline inside tests. `tests/conftest.py` self-configures Django settings with sqlite, so regression coverage should follow that pattern instead of introducing extra harness/config files.
- 2026-03-29: The upstream-friendly shape works cleanly: `StaticModelObserver(ModelObserver)` can reuse save/delete/m2m machinery and only replace `_connect()` plus `prepare_messages()`. No need to fork serializer or channel send logic.
- 2026-03-29: Regression proof is strongest with dedicated test models. Reusing models from older observer tests risks cross-test contamination because signal receivers stay connected for the process lifetime.
- 2026-03-29: Validation result for the fork after the patch: full suite passed (`56 passed`). The deferred-hydration regression is covered with exact query-count assertions (`4` for dynamic observer over three rows, `1` for static observer).
