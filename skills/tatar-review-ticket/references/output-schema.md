# Reviewer Output Contract

Return only a valid JSON array with no Markdown fence or surrounding prose. Each object must contain exactly these fields:

```json
[
  {
    "id": "correctness:path/to/file.ts:42:missing-validation",
    "role": "correctness",
    "category": "correctness",
    "title": "Duplicate campaign name can advance the wizard",
    "location": "apps/web/path/to/file.tsx:42",
    "evidence": "The branch returns success and no later guard rejects the duplicate.",
    "trigger_or_requirement": "Submit an existing normalized campaign name",
    "impact": "The user advances to a state the API later rejects.",
    "suggested_direction": "Enforce uniqueness in the canonical transition.",
    "scope_relation": "current_regression",
    "confidence": "high"
  }
]
```

Constraints:

- `role` must equal the single requested role.
- `category` must be one allowed by the selected role instruction.
- `scope_relation` must be one of `current_required`, `current_regression`, `preexisting_out_of_scope`, `unclear`.
- `confidence` must be `high` or `medium`.
- `id` must be stable and deterministic: `<role>:<location>:<short-slug>`.
- Use `location="N/A"` only for `review_incomplete`.
- Return `[]` when no reportable finding remains.
