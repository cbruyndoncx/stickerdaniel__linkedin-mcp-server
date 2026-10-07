# Tool Reference

What each MCP tool takes, returns, and changes on your account. Written against
v4.26.2. The tool docstrings in `linkedin_mcp_server/tools/` are the source of
truth, and your MCP client shows them; when this page and a docstring disagree,
the docstring wins.

## How results look

Every tool reads LinkedIn's rendered page in a real browser. No private API is
used. Most tools return:

- `url`: the page that was read.
- `sections`: section name -> raw page text. The agent parses this text; the
  server does not turn it into structured fields.
- `references` (optional): typed links found on the page, such as `/in/` profile
  paths, company slugs, job IDs and post permalinks.
- `section_errors` (optional): why a section is missing or incomplete, for
  example a rate-limit page or an extraction that found nothing.
- `unknown_sections` (optional): section names you asked for that do not exist.

A missing section means nothing was extracted, not that the data does not exist.

Usernames, company names and job IDs also accept the full LinkedIn URL, which is
reduced to the slug or ID.

## Write actions and side effects

| Tool | What it changes |
|------|-----------------|
| `send_message` | Sends a message when `confirm_send` is true. |
| `connect_with_person` | Sends a connection request, or accepts an incoming one. |
| `get_conversation` | Looking up by username clicks inbox rows, which can mark them as read. |
| `search_conversations` | Clicks result rows, which can mark them as read. |
| `close_session` | Closes the browser. Your saved login is kept. |

`send_message` and `connect_with_person` are marked destructive, so MCP clients
that honour that hint ask you before running them. All other tools are read-only.

## People

### `get_person_profile`

Read someone's profile.

| Parameter | Default | Notes |
|-----------|---------|-------|
| `linkedin_username` | required | e.g. `williamhgates` |
| `sections` | main page only | Comma-separated extras: `experience`, `education`, `interests`, `honors`, `languages`, `certifications`, `skills`, `projects`, `contact_info`, `posts` |
| `max_scrolls` | 5 per detail section, 10 for posts | 1-50. "Show more" clicks or scrolls per section |

Each extra section loads its own page, so ask only for the sections you need.
Request heavy sections, like 30+ certifications, in a separate call with a higher
`max_scrolls`.

### `get_my_profile`

Same as `get_person_profile` for the signed-in account, without a username. The
returned `url` reveals your real username.

### `search_people`

| Parameter | Default | Notes |
|-----------|---------|-------|
| `keywords` | required | e.g. `recruiter at Google` |
| `location` | none | e.g. `New York` |
| `network` | none | Connection degree: `F` (1st), `S` (2nd), `O` (3rd+). A list, a single token or `"F,S"` |
| `current_company` | none | Numeric company URN id only. Plain names are silently ignored by LinkedIn. Get the id from `get_company_profile` under `references["about"]` |

### `get_sidebar_profiles`

Takes `linkedin_username`. Returns `/in/` paths from the "More profiles for
you", "Explore premium profiles" and "People you may know" sidebars on that
profile, following "Show all". Sections that lead to LinkedIn Premium are
skipped.

### `connect_with_person`

| Parameter | Default | Notes |
|-----------|---------|-------|
| `linkedin_username` | required | |
| `note` | none | Free accounts have a limited quota of personalized notes |

Returns a `status`:

| Status | Meaning |
|--------|---------|
| `connected` | This call sent the invitation. Not yet a 1st-degree connection |
| `accepted` | An incoming invitation was accepted |
| `pending` | An invitation was already outstanding |
| `already_connected`, `follow_only`, `connect_unavailable`, `unavailable` | Nothing sent |
| `note_not_supported`, `custom_note_limit_reached` | The note could not be added |
| `send_failed` | The send did not complete. One may still have gone out |
| `outcome_unknown` | The browser stopped mid-call. Check the profile before retrying, or you may invite twice |

## Companies

### `get_company_profile`

| Parameter | Default | Notes |
|-----------|---------|-------|
| `company_name` | required | The URL slug, e.g. `docker` |
| `sections` | about page only | Extras: `posts`, `jobs` |

`references["about"]` usually carries the numeric `company_urn` that
`search_people` needs for `current_company`.

### `get_company_posts`

Takes `company_name` and `max_scrolls` (1-50, default 10). Increase
`max_scrolls` to read further back.

### `search_companies`

Takes `keywords`. Use it to find a company's slug before calling the other
company tools.

### `get_company_employees`

| Parameter | Default | Notes |
|-----------|---------|-------|
| `company_name` | required | The exact URL slug, not the display name. Anthropic is `anthropicresearch`, not `anthropic` |
| `keywords` | none | Filter by name, title or skill |

Also returns where employees live, where they studied, and a breakdown by
function. To filter by connection degree or location, use `search_people` with
`current_company` instead.

## Jobs

### `search_jobs`

| Parameter | Default | Notes |
|-----------|---------|-------|
| `keywords` | required | |
| `location` | none | e.g. `Remote` |
| `max_pages` | 3 | 1-10 |
| `date_posted` | any | `past_hour`, `past_24_hours`, `past_week`, `past_month` |
| `job_type` | any | Comma-separated: `full_time`, `part_time`, `contract`, `temporary`, `volunteer`, `internship`, `other` |
| `experience_level` | any | Comma-separated: `internship`, `entry`, `associate`, `mid_senior`, `director`, `executive` |
| `work_type` | any | Comma-separated: `on_site`, `remote`, `hybrid` |
| `easy_apply` | false | Only Easy Apply jobs |
| `sort_by` | LinkedIn's default | `date` or `relevance` |

Returns `job_ids` for `get_job_details`, plus `total` (LinkedIn's advertised
count; `exact` is false for values like "1,000+") and `promoted_job_ids`. No
matches gives an empty `job_ids` and a `no_matching_jobs` error, not LinkedIn's
unrelated recommendations.

### `get_job_details`

Takes `job_id`. Returns the posting text, similar jobs as references, and
`apply: {type: "applied" | "closed"}` when the posting shows either. No `apply`
key does not mean the job is open. A `description_missing` error means the text
may be incomplete; calling again may return more.

### `get_job_apply_url`

Takes `job_id`. Returns `apply.type` (`easy_apply`, `external`, `applied`,
`closed` or `unknown`) and, for external postings, the employer's application
link as LinkedIn gives it. Nothing is clicked and the link is not opened, so
short links stay unexpanded.

### `get_saved_jobs`

| Parameter | Default | Notes |
|-----------|---------|-------|
| `stage` | `saved` | Job tracker tab: `saved`, `in_progress`, `applied`, `archived` |
| `max_pages` | 3 | 1-10 |

## Messaging

### `get_inbox`

Takes `limit` (1-50, default 20). Returns recent conversations with thread
links where they could be verified. `section_errors.inbox` says where that
stopped.

### `get_conversation`

| Parameter | Default | Notes |
|-----------|---------|-------|
| `thread_id` | none | Preferred. Reads the thread without touching other rows |
| `linkedin_username` | none | Searches the inbox and clicks matching rows, which can mark them as read |
| `index` | 0 | Which thread, when one person has several (e.g. a DM and an InMail). Ignored with `thread_id` |

Pass one of `thread_id` or `linkedin_username`.

### `search_conversations`

Takes `keywords` and `limit` (1-50, default 20). Each result row is clicked to
find its thread, which can mark it as read, so keep `limit` low for broad
queries.

### `send_message`

| Parameter | Default | Notes |
|-----------|---------|-------|
| `linkedin_username` | required | |
| `message` | required | Multi-line is fine. Tabs and other control characters are refused |
| `confirm_send` | required | `false` is a dry run; `true` sends |
| `profile_urn` | none | Extra check that the profile is the person you mean |

Things to know:

- It always opens a new compose window from the profile. It is **not** a reply:
  it may start a separate DM even when a thread with that person exists
  ([#483](https://github.com/stickerdaniel/linkedin-mcp-server/issues/483)).
- The recipient must have a Message button on their profile. If not, connect
  first and send after they accept.
- `enter_to_send_enabled` means your LinkedIn setting "Press Enter to Send" is
  on. Switch it to "Click Send to send", then retry.
- `sent: true` means the message appeared in the thread. It says nothing about
  delivery or reading.
- **Never retry when `retry_safe` is false**, or the message may be sent
  twice. `outcome_unknown` means only LinkedIn knows whether it went out.

## Feed and posts

### `get_feed`

Takes `num_posts` (1-50, default 10). Posts load in batches of about 5, so you
may get a few more. Long posts are not expanded; open the permalink in
`references["feed"]` for the full text.

### `search_posts`

| Parameter | Default | Notes |
|-----------|---------|-------|
| `keywords` | required | |
| `date_posted` | any | `past-24h`, `past-week`, `past-month` (the `search_jobs` spellings work too) |
| `max_pages` | 3 | 1-10. Scroll depth, about 5 scrolls each |

Searches all public posts, not just your feed. Permalinks in `references` are not
in result order.

## Session

### `close_session`

No parameters. Closes the shared browser and keeps your login. The browser also
closes by itself after `BROWSER_IDLE_TIMEOUT` (default 600 seconds) without a
tool call.

## Timeouts

A whole tool call times out after `TOOL_TIMEOUT` seconds (default 180). Raise it
for calls that read many sections or pages, or on a slow connection. Single page
operations use `TIMEOUT` (default 5000 ms).
