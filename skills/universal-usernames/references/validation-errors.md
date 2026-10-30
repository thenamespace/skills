# Validation errors

Every issue the validator CLI, the Namespace App, and the gateway report. Each has a stable `code`, a `kind`, a `path` (RFC 6901 JSON Pointer into the body, `""` for the root), and a message. Fix every reported issue, then validate again until the CLI exits `0`.

- `envelope`: fails the whole profile. Every lookup for that username errors.
- `record`: fails one record. Other records still resolve.
- `transport`: the request itself failed, before a profile could be read.

Source of truth: https://docs.namespace.ninja/developer-guide/universal-usernames/troubleshooting

## Envelope

| Code | Cause | Fix |
| --- | --- | --- |
| `INVALID_JSON` | Body isn't valid JSON (trailing comma, HTML error page served as `200`) | Serialize with the platform JSON encoder |
| `DUPLICATE_KEY` | A key repeats in one object, at any depth | Build objects from a map so each key appears once |
| `NOT_AN_OBJECT` | Body is an array, string, or other non-object | Return one profile object |
| `UNSUPPORTED_VERSION` | `version` missing or not the number `1` | Send `"version": 1` |
| `UNKNOWN_FIELD` | A top-level field outside `version`, `username`, `texts`, `addresses`, `contenthash` | Move text records into `texts`; drop the rest |
| `USERNAME_MISSING` | `username` missing or empty | Include it on every `200` |
| `USERNAME_MISMATCH` | `username` differs from the requested username | Echo the decoded path segment, not the stored display name |
| `TEXTS_NOT_OBJECT` | `texts` missing or not an object | Send `"texts": {}` when empty |
| `ADDRESSES_NOT_OBJECT` | `addresses` missing or not an object (for example an array of pairs) | Send an object keyed by coin type, or `{}` |
| `TOO_MANY_TEXTS` | More than 128 text keys | Publish only the records wallets need |
| `TOO_MANY_ADDRESSES` | More than 128 addresses | Use the default EVM address `2147483648` instead of one entry per EVM chain |

## Record

| Code | Cause | Fix |
| --- | --- | --- |
| `TEXT_KEY_INVALID` | Text key empty or longer than 256 characters | Shorten or drop it |
| `TEXT_VALUE_INVALID` | Text value isn't a string (`null`, number, object) | Stringify it, or omit the record |
| `TEXT_VALUE_TOO_LONG` | Text value longer than 4,096 characters | Truncate, or link to the content with `url` |
| `COIN_TYPE_INVALID` | Address key isn't a canonical decimal in `0..4294967295` (`"ETH"`, `"0x3c"`, `"060"`) | Use the decimal coin type, such as `"60"` |
| `COIN_TYPE_UNSUPPORTED` | No address coder for that coin type, usually a chain ID used by mistake (`"8453"`) | Use the coin type (`"2147492101"` for Base) or remove the entry |
| `ADDRESS_INVALID` | Value doesn't decode for its coin: bad EVM checksum, a Bitcoin address under `"60"`, empty, or over 1,024 characters | Send checksummed or lowercase EVM addresses and match each address to its coin |
| `CONTENTHASH_INVALID` | Known protocol, malformed value (`ipfs://` followed by something that isn't a CID) | Fix the CID or hash |
| `CONTENTHASH_UNSUPPORTED` | Missing or unknown protocol (bare CID, `https://`) | Prefix `ipfs://`, `ipns://`, `bzz://`, or `ar://`, or omit `contenthash` |

## Transport

| Code | Cause | Fix |
| --- | --- | --- |
| `INVALID_URL` | Endpoint URL isn't `https://`, or has credentials, a query, a fragment, or over 2,048 characters | Connect a plain `https://` base URL |
| `DESTINATION_BLOCKED` | Host resolves to a private, loopback, link-local, CGNAT, multicast, or metadata address | Serve from a public address |
| `HTTP_TIMEOUT` | No complete response within 5 s | Index the username lookup; remove slow calls from the request path; avoid cold starts |
| `NETWORK_ERROR` | DNS, connection, or TLS failure | Check the hostname, that the server runs, and the certificate |
| `HTTP_REDIRECT` | Endpoint answered `3xx` | Connect the final URL; redirects are never followed |
| `UNAUTHORIZED` | Endpoint answered `401` or `403` | Match the saved bearer token; accept `Authorization: Bearer <token>`; accept old and new tokens during rotation |
| `RATE_LIMITED` | Endpoint answered `429` | Allow at least 10 requests/s with bursts of 20 from Namespace |
| `HTTP_ERROR_STATUS` | Any other non-`2xx`, usually `5xx` | Check server logs; answer `404` only for absent users |
| `USERNAME_NOT_FOUND` | `404` for the sample username during setup only | Validate with a username that exists |
| `CONTENT_TYPE_INVALID` | `Content-Type` isn't `application/json` | Set it explicitly; many frameworks default strings to `text/plain` or `text/html` |
| `RESPONSE_TOO_LARGE` | Body over 256 KiB after decompression | Remove large values such as data-URI images; link to them instead |
