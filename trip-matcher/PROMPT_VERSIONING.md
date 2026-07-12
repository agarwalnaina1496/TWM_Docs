# Prompt Versioning

Scout and Meridian are runtime prompts that evolve independently. Each deployed prompt has a human-readable semantic version so behavior can be released, debugged, and discussed without relying on a Git commit as the product-facing identifier.

## Files

The backend repository stores prompt release metadata beside the prompts:

```text
twm/prompts/
  scout.md
  meridian.md
  versions.json
  CHANGELOG.md
```

`versions.json` contains exactly one independent version per supported prompt:

```json
{
  "scout": "1.0.0",
  "meridian": "1.0.0"
}
```

`CHANGELOG.md` contains a heading for each current release:

```text
## Scout 1.0.0 — YYYY-MM-DD
## Meridian 1.0.0 — YYYY-MM-DD
```

The initial `1.0.0` versions establish tracked prompt releases. Meridian's legacy/model-returned `matcher_v2` value describes an output contract and is not prompt provenance.

## Semantic Version Policy

Each prompt follows semantic versioning independently:

```text
PATCH  wording or constraint clarification intended to preserve the contract
MINOR  backward-compatible behavioral capability or instruction change
MAJOR  breaking routing, ownership, state, or response-contract behavior
```

When a change affects only Scout, only Scout's version changes. The same rule applies to Meridian.

Every behavioral prompt change must include, in the same backend change:

1. the prompt edit;
2. the corresponding version bump in `versions.json`;
3. a matching entry in `CHANGELOG.md`;
4. relevant prompt regression tests; and
5. updated canonical documentation.

## Validation and Enforcement

Backend loading rejects:

- a missing or invalid `versions.json`;
- missing or unknown prompt keys;
- versions that are not valid semantic versions; and
- a current prompt version without a matching changelog heading.

The repository prompt-version check compares prompt files with a supplied base Git ref. If a prompt changed, its version must also change and its current changelog heading must exist.

Example:

```powershell
python scripts/check_prompt_version_changes.py origin/main
```

Git remains useful for implementation history, but it is not the primary released prompt version.

## Runtime Boundary

This versioning layer defines and validates prompt releases only. Returning deterministic version metadata from FastAPI responses is a separate API capability. The backend—not the LLM or n8n output—owns runtime provenance.

The UI must not infer a prompt version from response content.

## Documentation Definition of Done

A prompt release is incomplete until its changelog, behavior documentation, API/state documentation when applicable, and verification evidence agree with the implemented prompt.
