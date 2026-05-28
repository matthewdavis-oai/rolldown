# AST Mutation Between Passes

## Summary

Rolldown threads per-AST-node metadata between compiler passes via side tables. The cross-pass identity key is now oxc's post-semantic `NodeId`, while `Span` remains source-location metadata for diagnostics, comments, source maps, and generated replacement spans.

The public oxc type is `oxc::semantic::NodeId`. It is the implementation behind the `node_id()` / `set_node_id()` accessors on AST nodes after semantic analysis; there is not a separate public `AstNodeId` type in the version Rolldown currently uses.

## Pass Overview

Rolldown's bundling pipeline has three stages that interact with the AST:

- **Scan** - `ScanStage::scan` parses each module, runs Rolldown's pre-scan AST tweaks, then rebuilds semantic/scoping information. This final rebuild is what assigns every node — including the nodes the tweaks created — its `NodeId`, so the subsequent read-only walk via `AstScanner` sees stable ids while populating `EcmaView` side tables.
- **Link** - `LinkStage::link` performs cross-module work such as symbol binding, export resolution, tree shaking, and cross-module optimization. It still does not mutate the AST, but it can derive additional side tables from scan-time records.
- **Generate / Finalize** - `ScopeHoistingFinalizer`, driven from `GenerateStage::generate`, is the main stage that mutates the AST in place. It visits interesting nodes, calls `node_id()`, and queries the side tables to decide what to rewrite.

Between passes, Rolldown does not hold direct references to AST nodes. Lifetimes and parallel cross-module work make that impractical. The durable identity for a node within one module AST is therefore its `NodeId`.

## NodeId Contract

The shared invariant across passes:

- **Insertion**: scan/link write side-table entries using the `NodeId` of the AST node being recorded.
- **Lookup**: finalizer or another later AST walker reads `node_id()` from the current node and queries the table.
- **Required guarantees**: the node comes from the same post-semantic AST, and the side table is scoped to the module unless the key also includes `ModuleIdx`.

Important constraints:

- `NodeId` is only unique within a single AST. Any table that combines records from multiple modules must key by `(ModuleIdx, NodeId)`.
- `NodeId` is meaningful only after semantic analysis has assigned ids. Rolldown's normal scan path is post-semantic, so scan-created records are valid.
- Synthetic/default nodes use `NodeId::DUMMY` unless ids are assigned later. Do not insert cross-pass side-table records for synthetic `DUMMY` nodes.
- Cloned post-semantic nodes can preserve the original node id unless the clone is reset or semantic information is rebuilt. Treat cloned nodes as identity-sensitive.

Rolldown relies on that last point deliberately. Both the incremental-build cache (`NormalizedScanStageOutput::make_copy`) and HMR finalize a *clone* of the scanned AST, produced by `EcmaAst::clone_with_another_arena` into a fresh allocator. That clone uses oxc's `clone_in_with_semantic_ids` rather than plain `clone_in` (which would reset every id to `NodeId::DUMMY`), so the copied nodes keep their original ids. This is what lets the "same post-semantic AST" guarantee hold even when the finalizer mutates a different allocation than the one scan walked.

## Current NodeId-Keyed Tables

The main cross-pass side tables keyed by `NodeId` are:

- `EcmaView::imports` - import declarations, export-from declarations, dynamic `import()` expressions, and recognized `require()` call expressions.
- `EcmaView::dummy_record_set` - `require` identifier references that need the runtime helper rewrite.
- `EcmaView::new_url_references` - `new URL('...', import.meta.url)` nodes mapped to asset import records.
- `EcmaView::this_expr_replace_map` - top-level `this` expressions that should become `exports` or `undefined`.
- `MemberExprRef::node_id` and `LinkingMetadata::resolved_member_expr_refs` - namespace/member-expression resolution from scan through link to finalization.
- `DynamicImportExprInfo::node_id` records the dynamic `import()` node within its own module; `EntryPoint::related_stmt_infos` then carries `(ModuleIdx, …, NodeId, …)` tuples so a dynamic-import entry can be traced back across the module graph.
- Cross-module optimization state, which comes in two shapes: a per-module set of side-effect-free call expressions (bare `NodeId`, only consumed within the same module's traversal) and a graph-wide set of unreachable dynamic imports keyed by `(ModuleIdx, NodeId)` because it aggregates records from every module.

This means finalizer-generated nodes that keep the default `NodeId::DUMMY` do not accidentally match scan-time records. `Span` no longer needs to double as the key for these rewrite decisions.

## Where Span Still Belongs

`Span` remains the right representation for source positions. It is still used for:

- diagnostics and warnings that point at user source;
- comments, source-map ranges, directive/hashbang ranges, and TLA keyword locations;
- generated replacement spans where codegen should preserve a useful source location;
- import-record source locations, including the raw module-request span for resolver diagnostics and resolved `importer_span` for diagnostics that need to point at the full import site.

For import records, the module-request span belongs to `ImportRecordStateInit`: dependency
resolution diagnostics still need to underline the original specifier, but the span is not
carried into `ImportRecordStateResolved`. Resolved records keep `importer_span` because later
passes, such as TLA import-chain diagnostics, need a location for the resolved import edge.

For member expressions, `NodeId` is the cross-pass lookup key, but spans remain necessary as
source locations: `MemberExprRef::span` points diagnostics at the original expression, and the
finalizer applies the current member expression span to generated replacements so source-map and
diagnostic locations stay tied to the rewritten source range.

Do not add a cross-pass node side table keyed only by `Span`. If a later pass needs to identify the same AST node, prefer `NodeId`; if records from more than one module can share a table, include `ModuleIdx`.

## Address Use

Oxc `Address` is still acceptable for scratch state inside one live AST traversal, where producer and consumer operate before the traversal returns and no data survives as cross-pass metadata. The current example is:

- `PreProcessor`'s `statement_stack` / `statement_replace_map` in `crates/rolldown/src/utils/tweak_ast_for_scanning.rs`.

`PreProcessor` specifically *cannot* use `NodeId`: it runs before the final semantic rebuild (`recreate_scoping` in `crates/rolldown/src/utils/pre_process_ecma_ast.rs`), so node ids are not yet assigned to the nodes it creates or moves. `Address` is the only stable per-node identity available at that point, and it is safe because the table never outlives the traversal.

Do not store `Address` in module metadata, entry metadata, or link-stage tables that outlive the traversal that produced it. In the post-semantic scanner, prefer `NodeId` even for same-traversal node identity checks when the compared nodes already have semantic IDs.

## Pre-Scan Span Normalization

`PreProcessor` still rewrites duplicate spans for a targeted set of node kinds before semantic information is rebuilt. This remains useful for source-position behavior and for older code paths that still assume certain spans are unique, but it is no longer the identity mechanism for the cross-pass rewrite tables listed above.

The practical rule is simple: treat `Span` as location, `NodeId` as same-AST node identity, and `(ModuleIdx, NodeId)` as cross-module node identity.

## Related

- [bundler-data-lifecycle](./bundler-data-lifecycle.md)
- [module-id](./module-id.md)
