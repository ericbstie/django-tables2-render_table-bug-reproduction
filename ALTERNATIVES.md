# Alternative fixes for the re-entrant `{% render_table %}` bug

Upstream references: issue [jieter/django-tables2#1036](https://github.com/jieter/django-tables2/issues/1036),
PR [jieter/django-tables2#1037](https://github.com/jieter/django-tables2/pull/1037).

## The bug

`RenderTableNode.render()` attaches the template context to the table instance
(`table.context = context`) so `TemplateColumn` can render cell templates within
the surrounding context, and unconditionally runs `del table.context` in a
`finally` block. When the same instance is rendered re-entrantly — a custom
table template containing `{% render_table %}`, a `before_render()` hook, or a
`TemplateColumn` rendering the same table — the inner call deletes
`table.context` first and the outer `finally` raises:

```
AttributeError: 'MyTable' object has no attribute 'context'
```

## Red-green loop

Commit `d104452` adds three reproduction tests on top of unfixed `master`
(`e52d5bc`), all failing with the `AttributeError` above:

- `test_nested_render_of_same_table` — nested render via `before_render()`
  (the scenario from the issue)
- `test_nested_render_with_template_argument` — custom table template that
  itself contains `{% render_table table %}`
- `test_restores_existing_context_attribute` — a manually assigned
  `table.context` must survive a render

The third test also rejects the naive fix of wrapping the `del` in
`try/except AttributeError`, which silences the crash but still destroys
state and leaves the outer render without its context.

Each alternative below is one commit on this branch; each was verified green
against the full test suite (391–392 tests).

## Alternative 0 (upstream PR #1037, for reference): save and restore

Save `getattr(table, "context", sentinel)` before overwriting, restore (or
delete) it in `finally`. Minimal diff, fully backward compatible, fixes
re-entrancy. Still mutates the instance, so concurrent renders of one shared
instance from multiple threads/async tasks can clobber each other (a
pre-existing flaw it inherits from `master`).

## Alternative 1 (`c172d3b`): stack-backed `context` property on `Table`

`Table.context` becomes a property over a per-instance stack: assignment
pushes, `del` pops, reading returns the top. The template tag code is
**byte-identical to master** — its existing `table.context = context …
del table.context` protocol simply becomes re-entrant because every set/del
pair is balanced.

- Pros: smallest conceptual change; the tag, `TemplateColumn`, and code
  reading `table.context` in `before_render()` all work unchanged; manual
  assignment still works.
- Cons: assignment-as-push is surprising semantics for a public-ish attribute
  (two consecutive assignments now need two `del`s to fully clear); state
  still lives on the instance, so the cross-thread flaw remains.

## Alternative 2 (`0ed117b`): per-table render-context registry

A module-level `WeakKeyDictionary` in `utils.py` maps each table to a stack of
the `{% render_table %}` contexts it is currently being rendered in. The tag
pushes/pops; `TemplateColumn` reads the innermost entry, falling back to a
manually assigned `table.context` (which stays a plain attribute). The `Table`
class and instance are never touched.

- Pros: confines the whole mechanism to the two sites that participate in the
  hack; no property magic; the table object is never mutated.
- Cons: user code reading `table.context` inside `before_render()` no longer
  sees the render context (behavior change); the shared dictionary has the
  same cross-thread interleaving flaw as the instance attribute.

## Alternative 3 (`da6d9ba`, branch HEAD — recommended): `ContextVar` stack

A `ContextVar` in `utils.py` holds an immutable tuple of
`(table, context)` pairs for the renders currently in flight, innermost last.
The tag activates its context on entry and resets the token in `finally`.
`Table.context` becomes a property: while rendering it returns the innermost
active render context for that instance; outside rendering it falls back to
instance storage, so manual assignment/deletion behaves exactly like the old
plain attribute.

- Pros: fixes re-entrancy *and* makes renders isolated per thread and per
  async task (token reset is exception-safe and nest-safe by construction);
  `TemplateColumn` is untouched; `table.context` keeps working for readers
  (e.g. in `before_render()`) and writers; removes the long-standing `# HACK`
  comment for real.
- Cons: largest diff of the three; property + `ContextVar` indirection is more
  machinery than the save/restore one-liner.

Verified by an additional test rendering the same instance concurrently from
four threads (`test_concurrent_render_of_same_table_in_threads`), which no
instance-attribute approach can pass deterministically.

## Rejected along the way

- **`try/except` around the `del`** — hides the crash but the outer render
  loses its context (caught by the reproduction tests).
- **Per-render shallow copy of the table** — `TableData`, `BoundRows` and
  `BoundColumns` all back-reference the table, so a faithful copy needs
  invasive rebinding and changes `before_render()` semantics.

## Recommendation

For an upstream contribution aiming at minimal review surface, PR #1037
(save/restore) is the safest. If upstream is open to retiring the hack
properly, Alternative 3 is the most robust: it solves the reported bug and the
latent thread/async-safety issue in one move while keeping the public
`table.context` behavior intact.
