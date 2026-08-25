# Issue body — draft, not filed

Template: `.github/ISSUE_TEMPLATE/bug_report.yml`. Headings below match its labels exactly, because `gh issue create --body-file` bypasses the template without warning.

Dropdown values: Deployment Method → Docker. MCP Client → Claude Code (CLI). Version → 2.12.0. Redmine Version → 6.1.1.

---

## Bug Description

`list_redmine_issues` offers "find unassigned issues" in its own description and then refuses the only value that expresses it.

Redmine carries a filter's operator *inside* the value. `Query#add_short_filter` detects an operator prefix for the filter's type and splits the remainder on `|`, defaulting to `=` over the split list, so `"56|57"` is either tracker, `"!*"` is "none" and `"!4"` is "not 4". `tracker_id`, `priority_id` and `fixed_version_id` are typed `Optional[int]`, and `assigned_to_id` is `Optional[Union[int, Literal["me"]]]`, so none of those forms can be written on the named parameter — and `!*`, unassigned, cannot be written on it at all.

The escape hatch exists and the documentation closes it. `filters` is merged *after* the named parameters, so a key there overrides the parameter of the same name and every one of those forms works. But the `filters` entry reads "Use this for any filter not listed above", which tells a caller the route does not apply to exactly these keys.

Two adjacent gaps in the same docstring, found while establishing the above:

`filters` says nothing about a filter Redmine cannot read. Redmine applies a filter only if it is registered, and otherwise does nothing, silently — so the response is 200 with the collection unnarrowed, which reads as "everything matched" rather than as an error. A custom field enters the registered set only when it carries "Used as a filter". `list_redmine_projects` documents both facts; `list_redmine_issues`, the busier tool, documents neither.

`limit` says "default: 25, max: 1000" and nothing about cost. Since #241 the limit reaches python-redmine whole and is paged there in chunks of 100, and the follow-up requests are built from the number asked for without checking whether the collection is exhausted — so `ceil(limit / 100) - 1` of them are issued regardless. `list_redmine_projects` gained the equivalent sentence in review of #239; the two tools should not disagree about a cost they now share.

## Steps to Reproduce

Against any instance, with `status_id="*"` on every call so the open-only default cannot skew the comparison, and totals read from `include_pagination_info`:

1. `list_redmine_issues(status_id="*")` — the baseline count.
2. `list_redmine_issues(status_id="*", filters={"assigned_to_id": "!*"})` and again with `"*"`.
3. `list_redmine_issues(status_id="*", tracker_id=<a>)`, again with `<b>`, then `filters={"tracker_id": "<a>|<b>"}`.
4. `list_redmine_issues(assigned_to_id="!*")` — the same value on the named parameter.

## Expected Behavior

Steps 2 and 3 narrow the collection, and the docstring says how to write them.

## Actual Behavior

Steps 2 and 3 do narrow correctly — the two `assigned_to_id` halves sum to the baseline and the two trackers sum to the `|` form, which is the discriminating control, since a *discarded* filter also answers 200 with the unnarrowed collection. So the capability is there.

Step 4 returns `INVALID_ARGUMENTS`, "Input should be a valid integer ... or Input should be 'me'".

So the tool can do it, the parameter cannot, and the parameter's documentation points away from the thing that can.

## Version

2.12.0

## Deployment Method

Docker

## MCP Client

Claude Code (CLI)

## Redmine Version

6.1.1

## Relevant Logs

Not applicable — every call above returns HTTP 200 except step 4, which is refused at the FastMCP boundary before a request is made.

## Additional Context

**The fix should not be to widen the parameter types.** `assigned_to_id` is `Optional[Union[int, Literal["me"]]]` deliberately: #116 tightened it from `Union[int, str]` so the boundary refuses garbage rather than passing it to Redmine and returning an empty list a model reads as "nothing matched", and `test_user_id_typing_schema.py` pins that a bare-string branch is a regression. Accepting free-form strings would sell that back. Documenting the `filters` route costs nothing and keeps it.

Happy to open a pull request — the change is in the docstring's `filters`, `limit` and `assigned_to_id` entries and in `docs/tool-reference.md`, which does not currently list `filters` for this tool at all.
