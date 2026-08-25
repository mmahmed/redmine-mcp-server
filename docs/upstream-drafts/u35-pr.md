# Pull request body — draft, not filed

Branch: `feat/issue-list-filter-forms`. Substitute the real issue number for `#ISSUE` after filing the issue, in this body, the CHANGELOG bullets and the test docstring.

---

## Summary

Fixes #ISSUE.

`list_redmine_issues` offers "find unassigned issues" in its description and then refuses the only value that expresses it. Redmine carries a filter's operator inside the value and joins alternatives with `|`, which the typed parameters cannot represent — so `!*`, `56|57` and `!4` go through `filters`, which is merged after the named parameters and overrides them. That route already worked and was undiscoverable, because the `filters` entry said it was for "any filter not listed above".

This is a documentation change. The typed parameters are deliberately left alone: #116 tightened them so the boundary refuses garbage rather than handing it to Redmine and getting back an empty list, and widening them again would sell that back for something `filters` already does.

## Changes

- `filters` describes the operator-inside-the-value rule with the four forms a caller actually needs, says that it overrides the named parameter of the same name, and states that an operator a filter's type does not accept is read as a literal rather than erroring.
- `filters` states that a filter Redmine cannot read answers 200 with the collection unnarrowed, and that a `cf_<id>` needs "Used as a filter" on. `list_redmine_projects` already carried both sentences.
- `limit` states the paging cost: above 100 the request is paged in chunks of 100, one request per chunk *asked for*, so a ten-issue project read at `limit=1000` costs ten requests to return ten rows.
- `assigned_to_id` points at `filters` for the operator forms, where it already explains that arbitrary strings are refused.
- `docs/tool-reference.md` documents the `filters` parameter, which it had never listed for this tool, and the `limit` cost. The operator sets each filter's type accepts go here rather than in the docstring.

## Testing

`tests/test_issue_filter_forms.py`, 12 cases. Three assert the shipped schema now carries each fact and fail against the parent; the rest are regression pins on the route itself — each short-filter form reaching Redmine verbatim, `filters` overriding the named parameter, and #116's boundary refusal still firing.

The schema assertions read the registered tool through a FastMCP client rather than the source, since only the leading prose and the `Args:` entries are shipped in `tools/list`, and they collapse whitespace so they do not depend on where the docstring happened to wrap.

Measured cost, from the registered schema rather than read off the source: `list_redmine_issues` goes from 2,538 to 3,717 characters, +1,179 across the whole server. Nearly all of it is the `filters` entry, at 119 → 939. That is the entry a caller cannot act without — the operator rule is the whole fix — and the material that is reference rather than instruction went to `docs/tool-reference.md` instead.

Full suite green on a clean checkout, `black --check src/` and `flake8 src/ --max-line-length=88` clean.

## Checklist

- [x] `python tests/run_tests.py --all` passes (activate `.venv` first)
- [x] `uv run black --check src/` and `uv run flake8 src/ --max-line-length=88` are clean
- [x] `CHANGELOG.md` [Unreleased] has a bullet for user-facing changes
- [x] No manual version bumps
- [x] Docs updated where behavior changed
- [x] Change works with both local and Docker deployments
