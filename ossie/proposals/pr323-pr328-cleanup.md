# PR #323 / #328 cleanup

## TL;DR

- **Close #323 as superseded by #383.** Current `main` (`cc919e9`) is a flat,
  one-model-per-document schema, so the semantic model's `ai_context` is already a native
  root property. #323's motivating `semantic_model` array no longer exists, and its
  multi-model example is invalid against `main`.
- **Leave #328 closed and record that #329 settled the token as `POWER_BI`.** The merged
  Microsoft converter writes and reads `POWER_BI`; the core schema lists `POWER_BI` as a
  well-known `Vendor` example; and #396 preserves that choice while migrating the converter
  to flat documents. #328's rule that `MICROSOFT` is the only token and readers must reject
  `POWER_BI` directly contradicts shipped behavior.
- **Two independent cleanup hunks remain valuable and were salvaged on current `main` in
  local commit `546022c`:**
  1. separate the flattened document-root example from the Enumerations YAML document; and
  2. reconcile already-established vendor tokens across the schema, spec, Python advisory
     enum, and converter index, without adding `MICROSOFT`.
- **Do not salvage either large example.** #323's example encodes the removed wrapper.
  #328's example is schema-valid only because vendor names are free-form, but it documents
  the wrong token and is coupled to a payload contract that does not match the merged
  converter.
- **Move the real multi-source Data Agent concern to a bundle-format discussion.** #383
  intentionally defines one semantic model per standalone document and explicitly does not
  define a bundle. Shared instructions across multiple source documents are now a bundle
  manifest concern, not a missing root `ai_context` concern.

No PR was pushed, commented on, closed, updated, or merged.

## Baseline inspected

| Item | State at inspection | Relevant head |
|---|---|---|
| `main` | current | `cc919e9` (`Define one semantic model per document without a wrapper (#383)`) |
| #323 | open, conflicting, changes requested | `62e5ee5` |
| #328 | closed unmerged at 2026-09-16 19:51:14Z | `88c0c19` |
| #329 | merged at 2026-09-16 04:53:38Z | main commit `7b8cdaa` |
| #383 | merged at 2026-09-16 17:08:07Z | main commit `cc919e9` |
| #396 | open, mergeable, checks passing | `ad8f337` |

## Conclusion 1: #383 made root `ai_context` native

#323 was correct for its original schema: the document root was
`{version, semantic_model: [...]}`, while `ai_context` lived on each member of the
`semantic_model` array. Its proposed root key supplied context shared by all members.

#383 removed that shape. On current `main`, a standalone document is the semantic model:

```text
properties:
  version, name, description, ai_context, datasets, relationships, metrics,
  custom_extensions
required:
  version, name, datasets
additionalProperties: false
```

The root schema's `ai_context` is:

```json
{"$ref": "#/$defs/SemanticModel/properties/ai_context"}
```

That is not merely equivalent functionality: the former nested model property is literally
reused at the document root. #323's schema, Markdown, Python model, serialization tests, and
docs changes therefore target a document wrapper that no longer exists.

The proposed `examples/multi_model_ai_context.yaml` fails the current schema with exactly
three root errors:

```text
'name' is a required property
'datasets' is a required property
Additional properties are not allowed ('semantic_model' was unexpected)
```

The Python SDK and converters on `main` were not migrated together with #383. That is a
downstream integration break, not a reason to retain #323's legacy wrapper: #396 performs
the flat-document migration, rejects legacy `semantic_model` wrappers explicitly, and makes
the Microsoft exporter return `{"version": ..., **semantic_model}`.

### Still-valuable #323 hunk

Commit `dce3ecd`, split from #323 as #367 and later closed unmerged, added a missing `---`
before the document-root section in `core-spec/spec.yaml`. #383 changed the root fields but
left the structural bug in place: without the delimiter, `name`, `datasets`, and the other
root fields parse in the same YAML document as `dialects`, `datatypes`, and `vendor_name`,
despite the file saying its sections describe separate schema components.

The hunk was reapplied to the flattened root. Parsing now yields:

```text
document 0: version
document 1: dialects, datatypes, vendor_name
document 2: name, description, ai_context, datasets, relationships, metrics,
            custom_extensions
```

No #323 root-`ai_context` code was retained.

## Conclusion 2: #329 settled the vendor token as `POWER_BI`

The merged evidence is consistent:

- `converters/microsoft/src/ossie_microsoft/_common.py` defines
  `VENDOR = "POWER_BI"`.
- The converter's reader selects only that token; its writer creates or merges a
  `POWER_BI` entry.
- The converter README describes unsupported model features as a `POWER_BI`
  round-trip stash.
- `core-spec/ossie-schema.json` lists `POWER_BI` in `Vendor.examples`.
- #396 preserves `VENDOR = "POWER_BI"` while adapting the converter to #383.

#328 instead says:

```text
There is exactly one token. MICROSOFT is canonical for both writing and reading.
Writers MUST NOT emit POWER_BI ... Readers MUST NOT accept them as aliases.
```

Those requirements cannot coexist with #329. Adding `MICROSOFT` as a second well-known token
would also leave unclear whether it is an alias, a future generic Tabular contract, or a
different payload. #328's organization-wide rationale is broader than the product-specific
stash that actually shipped.

The worked `examples/microsoft_extension.yaml` does validate against current `main`, but that
does **not** establish the token: `Vendor` is intentionally any string, and its examples are
advisory. The example was not wired into validation CI. The accompanying 436-line contract
also diverges from implementation:

- it requires `MICROSOFT`, while the converter uses `POWER_BI`;
- it defines a `format` negotiation rule that the converter does not implement;
- it requires foreign vendor entries to be preserved, while Power BI export explicitly
  warns that foreign entries have no equivalent and drops them; and
- its sequencing section says DAX has not landed, but #329 landed DAX.

Retokenizing the document is therefore not a safe mechanical salvage. A future payload
reference should be generated from and tested against the actual `POWER_BI` converter
contract, preferably after #396 lands.

### Still-valuable #328 hunks

#328 also exposed real list drift unrelated to `MICROSOFT`:

- the core Markdown table already listed Honeydew and Sigma, but the schema examples omitted
  them;
- the converter index listed Omni and NVIDIA GSF, but the core table and Python advisory enum
  omitted them;
- `POWER_BI` was in the schema and converter but absent from the core table and Python enum;
  and
- the Databricks example used `Databricks` instead of the established `DATABRICKS` token.

Local commit `546022c` keeps those reconciliations while omitting `MICROSOFT`, the proposed
Microsoft payload contract, and the proposed example. It also clarifies that the registry is
advisory and that arbitrary vendor strings remain valid.

## Conflicts and downstream references

### Mechanical conflicts

- Merging #323 into current `main` produces content conflicts in
  `core-spec/ossie-schema.json` and `core-spec/spec.yaml`. Other files can auto-merge, but
  their wrapper-based content is semantically obsolete.
- #328's current head merges post-#383 `main`, so `git merge-tree` reports no textual
  conflict. Its conflict is semantic: it forbids the token now emitted by merged code.

### References that need cleanup or redirection

- #323 references open issue #322, whose problem statement also assumes a multi-model
  `semantic_model` array. Close it as superseded or reframe only the surviving bundle
  question.
- #323 is cross-referenced by #367, the closed unmerged sectioning PR. That one useful hunk
  is present in local commit `546022c`.
- #328 is cross-referenced by merged #329, and #329's body still says the converter works
  "in combination with #328." That sentence is now stale: the converter and schema shipped
  `POWER_BI` without #328.
- #383 explicitly names #396 as the required converter/Python migration and says the two
  should be coordinated. They were not: Microsoft CI passed on `7b8cdaa` and failed on
  `cc919e9`. #396 is now the correct place for flat-document converter fixes; its Microsoft
  changes preserve `POWER_BI`.
- #383 explicitly says bulk exchange uses separate documents and introduces no bundle
  format. That sentence is the natural starting point for the Data Agent follow-up.

## Future bundle-format discussion for multi-source Data Agents

The use case behind #323 remains legitimate but changed layers. A Data Agent may federate
several sources that have no shared schema, namespace, relationships, or query language. #383
now requires each source model to be a separate standalone document, so instructions shared
across all sources cannot be expressed once inside any one core document.

A future discussion should ask whether Ossie needs an optional bundle manifest containing:

- bundle identity and version;
- ordered or named references to standalone model documents;
- bundle-level `ai_context` for cross-source instructions;
- source identity and query-language metadata per member; and
- explicit semantics that no cross-model relationship or join is implied.

It should not reintroduce `semantic_model: [...]` into the core document or overload a single
model's root `ai_context` with bundle scope.

## Exact closure comment drafts

### #323

> Thanks everyone for the review. #383 has now superseded this change by making each
> standalone Ossie document exactly one semantic model, with the model properties directly
> at the root. As a result, `ai_context` is already a native root property on `main`; the
> `semantic_model` array that motivated this PR no longer exists, and the multi-model example
> here is no longer valid under the core schema.
>
> The multi-source Data Agent use case is still real, but it is now a separate bundle-format
> question. #383 deliberately leaves bundles out of scope and represents bulk exchange as
> separate documents, so shared guidance across several source models should be discussed as
> bundle-level metadata rather than by restoring another root key to the old wrapper.
>
> The independent `spec.yaml` section-boundary cleanup remains useful and can move as a tiny
> standalone follow-up. I am closing this PR as superseded by #383 rather than rebasing the
> obsolete wrapper-based changes. Thank you for pushing on the motivation and helping narrow
> the scope.

### #328

> Thanks for the review and coordination here. Since this was opened, #329 merged the
> Microsoft converter with `POWER_BI` as the `custom_extensions.vendor_name`, and the core
> schema now lists `POWER_BI` as a well-known Vendor example. The flat-document migration in
> #396 preserves the same token.
>
> That shipped behavior conflicts with this PR's central rule that `MICROSOFT` is the only
> canonical token and that readers must not accept `POWER_BI`. Introducing a second token now
> would leave unclear whether it is an alias or a distinct payload contract. The useful
> registry reconciliation and Databricks casing fix can be carried independently without
> registering `MICROSOFT`; any detailed Microsoft extension reference should instead be
> derived from and tested against the actual `POWER_BI` converter payload.
>
> I am leaving this PR closed because #329 settled the token differently. Thank you for the
> discussion and for surfacing the registry drift.

## Recommendation

1. Post the #323 closure comment and close #323 as superseded.
2. Optionally post the #328 note to document why the already-closed PR is not being revived.
3. Close or reframe #322; open a focused bundle-format discussion only if multi-source Data
   Agent exchange is ready for design.
4. Land #396 to repair the converter/Python break introduced when #383 landed first.
5. Upstream the small cleanup represented by local commit `546022c`, either as one focused
   cleanup PR or as two commits if maintainers prefer the section delimiter separated from
   vendor-list reconciliation.
6. If a Microsoft extension reference is still wanted, document the shipped `POWER_BI`
   payload in a separate, converter-tested PR after #396 rather than retokenizing #328.

## Local validation

- `git diff --check`: passed.
- JSON parse of `core-spec/ossie-schema.json`: passed.
- `core-spec/spec.yaml` structural assertions with `yaml.safe_load_all`: passed; the document
  root is isolated from Enumerations.
- `validation/validate.py examples/tpcds_semantic_model.yaml`: passed.
- `python/tests/test_models.py`: 9 passed.
- PR example checks against current schema: #323 invalid with the three expected wrapper
  errors; #328 structurally valid because vendor tokens are free-form.

`uv` could not download `hatchling`/`sqlglot` because this environment received a PyPI TLS
`HandshakeFailure`; validation was rerun successfully with the already-installed local
Python packages (`pytest 9.0.3`, `jsonschema 4.26.0`, `sqlglot 30.15.0`).
