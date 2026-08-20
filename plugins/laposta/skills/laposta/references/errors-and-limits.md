# Errors and limits

The numeric error codes and the rate limits are not part of the machine readable spec,
so this file is maintained by hand from the published Laposta API documentation.

## Rate limits

Counted per account, per minute.

| Account | Requests per minute |
| --- | --- |
| Free | 30 |
| Paid | 120 |

Going over gives `429` with a `Retry-After` header holding the number of seconds to
wait:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

There are no headers that report your usage while things are going well, so keep count
yourself and honour `Retry-After`. Cache what rarely changes: a list's fields do not
need fetching on every operation.

## Status codes

| Code | Meaning |
| --- | --- |
| `200` | Success. |
| `201` | Success, and something was created. |
| `400` | Something is wrong with the request or with the data you sent. |
| `401` | No valid key, or this endpoint is not enabled for this account. |
| `402` | The data was fine, but the request was not carried out. |
| `404` | This path does not exist. |
| `429` | Too many requests in one minute. |
| `500` | Something went wrong on Laposta's side. |

A `404` means the *path* is wrong. An object that does not exist gives `400` with error
code `203`. Never test for `404` to find out whether a list or subscriber exists.

## Error shape

```json
{
  "error": {
    "type": "invalid_input",
    "message": "Email: invalid address",
    "code": 208,
    "parameter": "email"
  }
}
```

`type` is one of three: `invalid_request` when the call itself is wrong,
`invalid_input` when the data you sent is wrong, `internal` when it is Laposta's fault.
`code` and `parameter` are usually present on `invalid_input` and are the fastest route
to a useful message for the end user.

## Numeric error codes

| Code | Meaning |
| --- | --- |
| `201` | The parameter is empty. |
| `202` | The value does not have the right form. |
| `203` | The value does not exist, for example an id that does not belong to this account. |
| `204` | The value already exists. |
| `205` | The value is not a number. |
| `206` | The value is not `true` or `false`. |
| `207` | The value is not a date. |
| `208` | The value is not an email address. |
| `209` | The value is not a URL. |
| `210` | The value is not valid JSON. |
| `999` | Something else is wrong with the value. |

An id belonging to a different account returns exactly what a nonexistent id returns:
`Unknown list`, code `203`. That is deliberate, so the API does not reveal what exists
on other accounts. Do not tell the user "that list belongs to another account"; you
cannot know that.

## Handling errors well

- Report the `message` to the user, not just the status code. It usually names the
  field.
- On `400` with code `204` while adding a subscriber, the address is already in the
  list. That is not a failure to retry; use `options[upsert]=true` instead.
- On `401`, check the key before anything else. The same `401` also appears when an
  endpoint is not enabled for the account, for instance bulk sync.
- On `429`, wait the number of seconds in `Retry-After` and try once more. Do not
  retry in a tight loop.
- On `500`, retry once after a short pause; if it persists, stop and say so.
