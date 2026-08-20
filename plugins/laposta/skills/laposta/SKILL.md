---
name: laposta
description: 'Work with Laposta, the Dutch email marketing platform (api.laposta.nl), through its REST API: lists, subscribers ("relaties"), custom fields, segments, campaigns, results and webhooks. Use when the user names Laposta, LAPOSTA_API_KEY or api.laposta.nl, when Laposta is known to be their mailing list tool, or when they ask how a Laposta endpoint works (/list, /member, /field, /segment, /campaign, /report, /webhook). It then covers adding, updating, looking up and unsubscribing subscribers, building and inspecting segments, creating, filling, testing, scheduling and sending campaigns, reading results, and managing webhooks. Do not use for other email platforms such as Mailchimp, Brevo, MailerLite or plain SMTP, for writing newsletter copy, or for generic contact data and CSV work that does not touch the Laposta API.'
license: MIT
metadata:
  api: https://api.laposta.nl/v2
  reference: generated from the Laposta API source, 41 operations across 7 resources
---

# Laposta

Laposta is an email marketing platform. Everything below talks to its REST API at
`https://api.laposta.nl/v2`.

## The API key

The key identifies the account, so it is the account. Handle it accordingly.

- Read it from the `LAPOSTA_API_KEY` environment variable. If it is not set, ask the
  user to export it in their own shell and run the request again. Do not accept the key
  pasted into the conversation, and never set it yourself with `export
  LAPOSTA_API_KEY=...` in a command: that writes the value into the command log.
- Never write the key into a file, a script, a commit, a log line or a printed
  command. When you show a command, write `-u "$LAPOSTA_API_KEY:"`, never the value.
- Never print the key to check it. No `echo $LAPOSTA_API_KEY`, no `env | grep LAPOSTA`.
  To test whether it works, call `GET /list` and look at the status code.
- A key can be revoked in the app under Koppelingen. If a key leaks, say so and tell
  the user to revoke it.

Get one: in Laposta go to **Koppelingen** and create an API key. It is shown once. A
free account can hold three keys.

## Calling the API

Authentication is HTTP basic with the key as the username and an empty password. The
trailing colon is required.

```bash
curl -s -u "$LAPOSTA_API_KEY:" https://api.laposta.nl/v2/list
```

Six conventions decide whether your call works:

1. **There is no PUT or PATCH.** You update by POSTing to the resource URL, for
   example `POST /member/{member_id}`. Updates are partial: fields you leave out keep
   their value.
2. **Bodies are form encoded**, `application/x-www-form-urlencoded`, with PHP style
   brackets for nested values: `from[name]`, `list_ids[]`, `custom_fields[voornaam]`,
   `options[upsert]`. The one exception is `POST /list/{list_id}/members`, which takes
   JSON.
3. **`list_id` moves.** On GET and DELETE it is a query parameter; on POST it is a
   body field. This is the single most common mistake.
4. **Responses are wrapped.** One object comes back as `{"list": {...}}`; a collection
   as `{"data": [{"list": {...}}, ...]}` with no pagination. Only the two history
   endpoints paginate, with a cursor.
5. **DELETE does not return 204.** It returns the object with `"state": "deleted"`.
6. **An unknown id gives 400 with code 203, never 404.** Do not probe with a 404 check.

## Safety

These rules are not optional. They exist because this API touches a real mailing list
and one of its operations is irreversible.

- **Ask the user to confirm, in plain words, before any destructive or sending
  operation.** That means every DELETE, plus `POST /list/{list_id}/members` (bulk
  sync), `POST /campaign/{campaign_id}/action/schedule` and
  `POST /campaign/{campaign_id}/action/send`. Say what will happen and to how many
  people.
- **Writing to many subscribers at once needs the same confirmation**, even though it
  is a loop of ordinary calls. From roughly ten writes in one go, stop and state the
  number, the list name and what changes. Doing it one call at a time does not make it
  a small change.
- One confirmation covers one action, or one batch you have spelled out. Do not ask
  twelve times for a batch you already described, and do not stretch a single yes to
  cover something the user has not seen.
- **`action/send` cannot be undone**, and calling it twice sends the campaign twice.
  Always send a test mail first with `action/testmail`.
- **`options[upsert]=true` overwrites** the fields you send on a subscriber who already
  exists. It is convenient and it is destructive; say that you are using it.
- **Deleting a list deletes its subscribers, fields and segments.** Deleting a
  subscriber is not the same as unsubscribing: to unsubscribe, POST the member with
  `state=unsubscribed`.
- **Start on a test list.** When the user is trying something out, offer to create a
  list called something like "API test" and work there.
- Read operations need no confirmation. Never ask permission to look something up.

**When part of a batch fails**, stop rather than push on. Report which records
succeeded, which failed and why, using the `message` and `parameter` from the error.
Laposta has no transactions, so there is nothing to roll back: the user needs to know
exactly where it stopped in order to fix it.

**Subscriber data is personal data.** Fetch what the question needs, not the whole list.
Do not dump full subscriber exports into the conversation or write them to a file unless
the user asked for exactly that.

## Where to look

Read the reference file you need; do not guess an endpoint from memory.

| File | What is in it |
| --- | --- |
| `references/endpoints.md` | Every operation: path, method, required and optional parameters, response |
| `references/schemas.md` | Every response object, field by field |
| `references/tasks.md` | The common jobs as end to end flows, in the right order |
| `references/pitfalls.md` | The traps: custom field names, segment definitions, state values, dates |
| `references/errors-and-limits.md` | Status codes, numeric error codes, rate limits |

**Before you write any subscriber data, do these two calls first.** They are not
optional, and reading the documentation is not a substitute for them.

```bash
curl -s -u "$LAPOSTA_API_KEY:" https://api.laposta.nl/v2/list        # which list_id
curl -s -u "$LAPOSTA_API_KEY:" \
  "https://api.laposta.nl/v2/field?list_id=$LIST"                    # which field keys
```

Custom fields are keyed by the `custom_name` from that second call, not by the label the
user sees. Guessing it does not raise an error; the value is simply dropped. Read
`references/pitfalls.md` before touching select fields or segments.

## What Laposta cannot do through the API

Say so plainly instead of improvising a workaround.

- **Labels can be read and set, but not created.** They arrive as a subscriber field
  named `labels`, and only labels the list already has are accepted. See
  `references/pitfalls.md`; this one is not in Laposta's published API documentation.
- **Campaigns built in the drag and drop editor** cannot have their content read or
  written; `/content` works only for campaigns created through the API or by import.
- **Bulk sync** is off unless the account is paid and has it enabled; you get a 401
  saying bulk requests are not permitted.
- There is no sandbox. Every call hits the live account the key belongs to.

## The operations

<!-- GENERATED:INDEX:START -->
All 41 operations. ⚠ marks the ones that need explicit user confirmation.

| Method | Path | Operation | What it does |
| --- | --- | --- | --- |
| POST | `/list` | `lijstToevoegen` | Adding a list |
| GET | `/list/{list_id}` | `lijstOpvragen` | Requesting a list |
| POST | `/list/{list_id}` | `lijstWijzigen` | Modifying lists |
| DELETE | `/list/{list_id}` | `lijstVerwijderen` ⚠ | Deleting a list |
| GET | `/list` | `alleLijstenOpvragen` | Requesting all lists |
| DELETE | `/list/{list_id}/members` | `lijstLeegmaken` ⚠ | Purging a list |
| POST | `/list/{list_id}/members` | `lijstSynchroniserenBulk` ⚠ | Synchronizing a list (bulk) |
| POST | `/field` | `veldToevoegen` | Adding a field |
| GET | `/field/{field_id}` | `veldOpvragen` | Requesting a field |
| POST | `/field/{field_id}` | `veldWijzigen` | Modifying fields |
| DELETE | `/field/{field_id}` | `veldVerwijderen` ⚠ | Deleting a field |
| GET | `/field` | `alleVeldenOpvragen` | Get all fields |
| POST | `/segment` | `segmentToevoegen` | Add segment |
| GET | `/segment/{segment_id}` | `segmentOpvragen` | Retrieve segment |
| POST | `/segment/{segment_id}` | `segmentWijzigen` | Modify segment |
| DELETE | `/segment/{segment_id}` | `segmentVerwijderen` ⚠ | Delete segment |
| GET | `/segment` | `alleSegmentenOpvragen` | Get all segments |
| POST | `/member` | `relatieToevoegen` | Adding a subscriber |
| GET | `/member/{member_id}` | `relatieOpvragen` | Requesting a subscriber |
| GET | `/member/{member_id}/history` | `relatiegeschiedenisOpvragen` | Subscriber history |
| POST | `/member/{member_id}` | `relatieWijzigen` | Modifying a subscriber |
| DELETE | `/member/{member_id}` | `relatieVerwijderen` ⚠ | Deleting a subscriber |
| GET | `/member` | `alleRelatiesOpvragen` | Get all subscribers |
| POST | `/webhook` | `webhookToevoegen` | Adding a webhook |
| GET | `/webhook/{webhook_id}` | `webhookOpvragen` | Requesting a webhook |
| POST | `/webhook/{webhook_id}` | `webhookWijzigen` | Modifying a webhook |
| DELETE | `/webhook/{webhook_id}` | `webhookVerwijderen` ⚠ | Deleting a webhook |
| GET | `/webhook` | `alleWebhooksOpvragen` | Get all webhooks |
| POST | `/campaign` | `campagneAanmaken` | Creating a campaign |
| GET | `/campaign/{campaign_id}` | `campagneOpvragen` | Requesting a campaign |
| GET | `/campaign/{campaign_id}/history` | `campagnegeschiedenisOpvragen` | Requesting campaign history |
| POST | `/campaign/{campaign_id}` | `campagneWijzigen` | Modifying a campaign |
| DELETE | `/campaign/{campaign_id}` | `campagneVerwijderen` ⚠ | Deleting a campaign |
| GET | `/campaign` | `alleCampagnesOpvragen` | Get all campaigns |
| GET | `/campaign/{campaign_id}/content` | `campagneContentOpvragen` | Requesting campaign content |
| POST | `/campaign/{campaign_id}/content` | `campagneContentVullen` | Filling campaign content |
| POST | `/campaign/{campaign_id}/action/send` | `campagneVerzenden` ⚠ | Sending a campaign |
| POST | `/campaign/{campaign_id}/action/schedule` | `campagneInplannen` ⚠ | Planning a campaign |
| POST | `/campaign/{campaign_id}/action/testmail` | `campagneTesten` | Testing a campaign |
| GET | `/report/{campaign_id}` | `campagneresultatenOpvragen` | Campaign results |
| GET | `/report` | `alleResultatenOpvragen` | Get all results |
<!-- GENERATED:INDEX:END -->
