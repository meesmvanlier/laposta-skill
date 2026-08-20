# Pitfalls

Every item below was verified against a live Laposta account. These are the things that
make the difference between code that works and code that silently does the wrong
thing.

## Custom fields are keyed by `custom_name`, not by the label

A field called "Voornaam" is written as `custom_fields[voornaam]`. The key is the
field's `custom_name`, which Laposta derives from the name. Never guess it: fetch the
fields first and read `custom_name` off each one.

```bash
curl -s -u "$LAPOSTA_API_KEY:" "https://api.laposta.nl/v2/field?list_id=$LIST" \
  | jq -r '.data[].field | "\(.custom_name)\t\(.name)\t\(.datatype)"'
```

The email field is the exception: it has `custom_name: null` and `is_email: true`. You
send the address as the top level `email` parameter, not inside `custom_fields`.

Cache this mapping. It changes only when someone edits the list's fields.

## A duplicate address is an error, and the error hands you the id

Adding an address that is already in the list returns `400` with code `204` and
`"message": "Email address exists"`. That response also contains the `member_id` of the
existing subscriber, which is often all you needed:

```json
{"error": {"type": "invalid_input", "message": "Email address exists",
           "code": 204, "parameter": "email", "member_id": "msbluircui"}}
```

If you want add-or-update behaviour, do not catch the error: pass
`options[upsert]=true` on `POST /member` and Laposta updates the existing subscriber
instead of failing.

## Rewriting the options of a select field can rewrite your subscribers' data

This one is quiet and destructive. Options are stored per option **id**, and a
subscriber stores the id, not the label. Send `options_full` as a key/value map of
`id → label` using bracket notation:

```bash
# keeps ids 1 and 2, adds a third option
curl -s -u "$LAPOSTA_API_KEY:" \
  -d "list_id=$LIST" -d "required=false" \
  -d "options_full[1]=Aanbiedingen" \
  -d "options_full[2]=Nieuws" \
  -d "options_full[new-0]=Events" \
  "https://api.laposta.nl/v2/field/$FIELD"
```

Rules for the keys: an existing numeric id keeps that option and renames it if the
label differs; a non numeric key such as `new-0` adds a new option; **an id you leave
out is deleted**. Always send every option you want to keep.

Do not use `options_full[]=A&options_full[]=B`. It is accepted, but it renumbers the
ids, and every subscriber then points at a different option than before. In a live test
a subscriber set to "Nieuws" came back as "Aanbiedingen" without any error.

A JSON encoded map (`options_full={"1":"Nieuws"}`) is rejected with
`options has to be an array` when the body is form encoded. Use the brackets.

## Labels are a custom field, and they hide until the list has one

Labels (the coloured tags on a subscriber in the app) work through the API, even though
Laposta's published API documentation does not mention them anywhere: the datatype list
there still stops at `select_multiple`. Everything below was measured against a live
account.

**They are invisible until the list has at least one label.** On a list with no labels,
`GET /field` and `GET /member` say nothing about them at all. Create a label in the app
first, then the field appears:

```json
{"field": {"custom_name": "labels", "datatype": "labels", "name": "Labels",
           "options": ["Klant"], "options_full": [{"id": 10542, "value": "Klant"}]}}
```

Do not conclude from an empty list that the account cannot do labels.

**Reading** them: they come back inside `custom_fields` as an array of label names.

```json
"custom_fields": {"voornaam": "Mees", "labels": ["Klant"]}
```

**Writing** them needs the array form, on `POST /member` and on
`POST /member/{member_id}` alike:

```bash
-d "custom_fields[labels][]=Klant" -d "custom_fields[labels][]=Beurs"
```

Three traps in that one line:

- **Without the brackets it fails silently.** `custom_fields[labels]=Klant` returns `200`
  with the labels unchanged. No error, no effect.
- **Only existing labels are accepted.** An unknown name gives `400` with code `203` and
  "Field labels: unknown option". You cannot create a label by assigning it; that happens
  in the app or through an import.
- **Use the name, not the id.** Sending the numeric id from `options_full` gives the same
  `203`, even though that id is what the field itself reports.

Send `custom_fields[labels][]=` with an empty value to clear them.

**In bulk the same mistake is worse.** `POST /list/{list_id}/members` takes JSON, so there
are no brackets — labels go in as an array. A plain string does not fail silently there,
it wipes the labels that were already on the subscriber:

```json
"custom_fields": {"labels": "Klant"}     // 200, edited_count 1, labels empty afterwards
"custom_fields": {"labels": ["Klant"]}   // correct
```

An unknown label does behave sanely in bulk: that subscriber lands in `errors` with the
same code `203` and is not added, while the rest of the batch goes through. Note that bulk
is off by default and answers `401 Not permitted to issue bulk requests (1)` until Laposta
enables it for the account, and only on paid accounts.

## Unsubscribing is not deleting

- `POST /member/{member_id}` with `state=unsubscribed` unsubscribes. The subscriber
  stays in the list and counts as `unsubscribed`.
- `DELETE /member/{member_id}` removes them.

Both return `200` with the member object. The spec's enum for `state` claims
`active|deleted`; in practice the values that work are `active`, `unsubscribed` and,
when reading, `cleaned`. Filter with `GET /member?list_id=...&state=unsubscribed`.

## `member_id` may be an email address

`GET /member/skilltest2@example.com?list_id=...` works and returns the same object as
the id would. Handy when you have the address and not the id. The same holds inside the
bulk sync body.

## Two different errors for a bad id

- A string that cannot be an id at all gives code `202`, "Invalid list_id".
- A well formed id that does not exist gives code `203`, "Unknown list".
- An id belonging to a different account gives that same `203`.

There is no `404` for missing objects; `404` means the path is wrong.

## Segment definitions cannot be written blind

`definition` must be valid JSON, but the exact shape is not documented anywhere that
holds up. The example in Laposta's own API documentation, `{"postcode"} = '1234AB'`,
is rejected as not valid JSON, and wrapping that same expression as a JSON string
returns **HTTP 500 with an empty body**. Several plausible JSON shapes all come back
with code `210`, "Invalid definition".

So do what Laposta's documentation itself advises: build the segment once in the app,
then read it back and reuse its `definition` verbatim.

```bash
curl -s -u "$LAPOSTA_API_KEY:" "https://api.laposta.nl/v2/segment?list_id=$LIST" \
  | jq -r '.data[].segment | "\(.segment_id)\t\(.name)\t\(.definition)"'
```

Tell the user this rather than guessing at a definition. A definition that references a
field name that does not exist is accepted and yields a segment that matches nobody.

## Campaign content tells you what is wrong with it

`POST /campaign/{campaign_id}/content` returns a `report` block. Read it, because the
call succeeds even when the html has problems:

```json
"report": {"no_unsubscribe": true}
```

That flag means Laposta found no unsubscribe link and added one itself. Merge tags you
invent are not tags: `{{unsubscribe_url}}` is treated as literal text. The generated
`plaintext` in the same response is the fastest way to see what Laposta made of your
html.

Content only works for campaigns created through the API or by import. A campaign built
in the drag and drop editor returns an error about the wrong campaign type.

## Sending

- `action/testmail` needs content to exist and the campaign not to be sent yet.
- `action/send` is final, and a second call sends the whole campaign again.
- `action/schedule` takes `delivery_requested` as a date; you cancel a scheduled
  campaign by deleting it.
- The `from[email]` address must already be approved in the account. An unapproved
  address fails at campaign creation, not at send time.

## Dates come in two flavours

Every timestamp appears twice: `created` in the account's timezone
(`2026-08-19 20:40:41`) and `created_iso` with an offset
(`2026-08-19T20:40:41+02:00`). The account timezone is in the `timezone` field of the
same object. The `_iso` variant can be `null` or missing entirely when the plain value
is empty, so read defensively. In a query string a `+` must be written `%2B`.

## Only history paginates

`GET /member/{id}/history` and `GET /campaign/{id}/history` use a cursor: read
`pagination.next_cursor` and pass it back as `cursor` while `pagination.has_more` is
true. `limit` defaults to 100 and caps at 1000, and `order_by` accepts only
`event_date`.

Every other collection endpoint returns everything at once, with no pagination and no
count. On a large list that is a big response; ask for what you need rather than
fetching all subscribers to find one.

## Bulk sync is usually off

`POST /list/{list_id}/members` returns `401` with "Not permitted to issue bulk
requests" unless the account is paid and has it enabled. Fall back to individual
`POST /member` calls with `options[upsert]=true`, and watch the rate limit.
