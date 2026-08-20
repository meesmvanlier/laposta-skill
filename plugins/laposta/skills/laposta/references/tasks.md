# Common tasks, end to end

Every flow below was run against a live Laposta account. Where a step is marked
**confirm first**, ask the user in plain words and wait for an answer before calling it.

All examples assume `LAPOSTA_API_KEY` is set and use `jq` to keep the output readable.

## Get your bearings

Start here whenever you do not know the account yet. Two calls tell you almost
everything.

```bash
curl -s -u "$LAPOSTA_API_KEY:" https://api.laposta.nl/v2/list \
  | jq -r '.data[].list | "\(.list_id)\t\(.name)\tactive=\(.members.active)"'

curl -s -u "$LAPOSTA_API_KEY:" "https://api.laposta.nl/v2/field?list_id=$LIST" \
  | jq -r '.data[].field | "\(.custom_name)\t\(.name)\t\(.datatype)"'
```

The second call is not optional before writing subscriber data: `custom_name` is the key
you must use, and it is not the label the user sees.

## Add a subscriber, or update them if they already exist

`ip` is required. Use the address of whoever signed up; when there is no real one,
`127.0.0.1` is accepted.

```bash
curl -s -u "$LAPOSTA_API_KEY:" \
  -d "list_id=$LIST" \
  -d "ip=127.0.0.1" \
  -d "email=sanne@example.com" \
  -d "custom_fields[voornaam]=Sanne" \
  -d "options[upsert]=true" \
  https://api.laposta.nl/v2/member
```

Without `options[upsert]=true` a known address returns `400` with code `204`. That
response includes the existing `member_id`, so it is still useful. Other options worth
knowing: `options[suppress_reactivation]`, `options[suppress_email_notification]` and
`options[ignore_doubleoptin]`.

For a batch, loop with one call per subscriber and respect the rate limit (30 per minute
on a free account). `POST /list/{list_id}/members` does bulk in one go but is disabled on
most accounts.

## Change an existing subscriber

POST to the subscriber's URL. Leave out what should stay as it is. Note that `list_id`
moves into the body here.

```bash
curl -s -u "$LAPOSTA_API_KEY:" \
  -d "list_id=$LIST" \
  -d "custom_fields[voornaam]=Sanne" \
  "https://api.laposta.nl/v2/member/$MEMBER"
```

`$MEMBER` may be the member id or the email address.

## Put a label on a subscriber

Labels are the coloured tags from the app. Read the list's labels first; you can only
assign ones that already exist.

```bash
# which labels does this list have?
curl -s -u "$LAPOSTA_API_KEY:" "https://api.laposta.nl/v2/field?list_id=$LIST" \
  | jq -r '.data[].field | select(.custom_name=="labels") | .options[]'

# assign two of them (the brackets are required)
curl -s -u "$LAPOSTA_API_KEY:" \
  -d "list_id=$LIST" \
  -d "custom_fields[labels][]=Klant" \
  -d "custom_fields[labels][]=Beurs" \
  "https://api.laposta.nl/v2/member/$MEMBER"
```

If the first call returns nothing, the list has no labels yet and there is nothing to
assign. Say so rather than inventing one; see `references/pitfalls.md`.

## Unsubscribe or delete

These are different things and the user usually means the first.

```bash
# unsubscribe: stays in the list, counts as unsubscribed
curl -s -u "$LAPOSTA_API_KEY:" -d "list_id=$LIST" -d "state=unsubscribed" \
  "https://api.laposta.nl/v2/member/$MEMBER"

# delete: gone            (confirm first)
curl -s -u "$LAPOSTA_API_KEY:" -X DELETE \
  "https://api.laposta.nl/v2/member/$MEMBER?list_id=$LIST"
```

Ask which one they mean if the request is ambiguous. "Remove Sanne from the newsletter"
is almost always an unsubscribe.

## Look things up

```bash
# one subscriber, by address
curl -s -u "$LAPOSTA_API_KEY:" \
  "https://api.laposta.nl/v2/member/sanne@example.com?list_id=$LIST" | jq '.member'

# everyone who unsubscribed
curl -s -u "$LAPOSTA_API_KEY:" \
  "https://api.laposta.nl/v2/member?list_id=$LIST&state=unsubscribed" \
  | jq -r '.data[].member.email'
```

`state` accepts `active`, `unsubscribed` and `cleaned`. There is no pagination and no
search: fetching a whole list returns everything. Prefer the single lookup when you know
the address.

## What happened to this subscriber

```bash
curl -s -u "$LAPOSTA_API_KEY:" \
  "https://api.laposta.nl/v2/member/$MEMBER/history?list_id=$LIST&limit=50" \
  | jq '{page: .pagination, events: [.data[].event | {type, event_date, campaign}]}'
```

Page through it while `.pagination.has_more` is true by passing
`.pagination.next_cursor` as `cursor`.

## Add a field, and change its options without breaking data

```bash
# a plain text field
curl -s -u "$LAPOSTA_API_KEY:" \
  -d "list_id=$LIST" -d "name=Voornaam" -d "datatype=text" \
  -d "required=false" -d "in_form=true" -d "in_list=true" \
  https://api.laposta.nl/v2/field
```

`datatype` is one of `text`, `numeric`, `date`, `select_single`, `select_multiple`. A
select field without options is accepted and ends up with none, so pass the options when
you create it.

Changing the options of an existing select field is the dangerous one. Send every option
you want to keep, keyed by its current id, and read the warning in `pitfalls.md` first.

```bash
curl -s -u "$LAPOSTA_API_KEY:" \
  -d "list_id=$LIST" -d "required=false" \
  -d "options_full[1]=Aanbiedingen" \
  -d "options_full[2]=Nieuws" \
  -d "options_full[new-0]=Events" \
  "https://api.laposta.nl/v2/field/$FIELD"
```

Changing a field's `datatype` wipes that field's data for every subscriber. **Confirm
first.**

## Segments

Read them; do not write them blind.

```bash
curl -s -u "$LAPOSTA_API_KEY:" "https://api.laposta.nl/v2/segment?list_id=$LIST" \
  | jq -r '.data[].segment | "\(.segment_id)\t\(.name)\t\(.definition)"'
```

To create one, the reliable route is: the user builds it once in the app, you read the
`definition` back and reuse it. The definition format is not documented in a form that
works; see `pitfalls.md`.

## Create a campaign and send it

Five steps, and the last one is irreversible.

```bash
# 1. create. from[email] must be an approved sender address in the account
CAMPAIGN=$(curl -s -u "$LAPOSTA_API_KEY:" \
  -d "type=regular" \
  -d "name=August newsletter" \
  -d "subject=What is new in August" \
  -d "from[name]=Example BV" \
  -d "from[email]=news@example.com" \
  -d "list_ids[]=$LIST" \
  https://api.laposta.nl/v2/campaign | jq -r '.campaign.campaign_id')

# 2. fill it. read the report block in the response
curl -s -u "$LAPOSTA_API_KEY:" \
  --data-urlencode "html@newsletter.html" \
  -d "inline_css=true" \
  "https://api.laposta.nl/v2/campaign/$CAMPAIGN/content" | jq '.campaign.report'

# 3. test it, always, before anything else
curl -s -u "$LAPOSTA_API_KEY:" -d "email=you@example.com" \
  "https://api.laposta.nl/v2/campaign/$CAMPAIGN/action/testmail"

# 4a. schedule            (confirm first)
curl -s -u "$LAPOSTA_API_KEY:" -d "delivery_requested=2026-09-01 09:00:00" \
  "https://api.laposta.nl/v2/campaign/$CAMPAIGN/action/schedule"

# 4b. or send now         (confirm first, cannot be undone)
curl -s -u "$LAPOSTA_API_KEY:" -X POST \
  "https://api.laposta.nl/v2/campaign/$CAMPAIGN/action/send"
```

Before step 4, tell the user how many people this reaches. Add up `members.active` of the
lists in `list_ids`. Cancel a scheduled campaign by deleting it.

Sending twice sends twice. If a send call errors, check the campaign's state before
retrying rather than calling it again.

## Read the results

```bash
curl -s -u "$LAPOSTA_API_KEY:" "https://api.laposta.nl/v2/report/$CAMPAIGN" \
  | jq '.report'

curl -s -u "$LAPOSTA_API_KEY:" https://api.laposta.nl/v2/report \
  | jq -r '.data[].report | "\(.campaign_id)\t\(.name)"'
```

## Webhooks

Laposta calls your URL when something changes in a list. The response to a create
contains a `secret` you use to verify the calls; treat it like a key.

```bash
curl -s -u "$LAPOSTA_API_KEY:" \
  -d "list_id=$LIST" -d "event=subscribed" \
  -d "url=https://example.com/hook" -d "blocked=false" \
  https://api.laposta.nl/v2/webhook

curl -s -u "$LAPOSTA_API_KEY:" "https://api.laposta.nl/v2/webhook?list_id=$LIST" \
  | jq -r '.data[].webhook | "\(.webhook_id)\t\(.event)\t\(.url)\tblocked=\(.blocked)"'

# confirm first
curl -s -u "$LAPOSTA_API_KEY:" -X DELETE \
  "https://api.laposta.nl/v2/webhook/$WEBHOOK?list_id=$LIST"
```

`event` is one of `subscribed`, `modified`, `deactivated`. Set `blocked=true` to pause a
webhook without deleting it.

## Set up a scratch list to try things on

When the user wants to experiment, do not experiment on their real list.

```bash
curl -s -u "$LAPOSTA_API_KEY:" \
  -d "name=API test" -d "remarks=Temporary, safe to delete" \
  https://api.laposta.nl/v2/list | jq -r '.list.list_id'
```

Clean up afterwards with `DELETE /list/{list_id}`, which also removes its subscribers,
fields and segments. **Confirm first**, and make sure it is the scratch list.
