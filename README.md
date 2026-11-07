# integration-spec: an open format for serving redirects from a rules file

**The open format behind RedirHub Integration Mode:** a plain-text rules file that a website's own
platform (a web server, a CDN, an app framework, a CMS) downloads to serve redirects itself, the
check path that proves it does, and the visit reports it sends back.

[RedirHub](https://www.redirhub.com) writes the rules file for every hostname connected "through
your own platform" and reads the reports into its analytics. Anyone can implement a client; this
repository is the contract between them.

| | |
| --- | --- |
| [Rules file format v1](#rules-file-format-v1) | What a client downloads and how it reads it |
| [Answering requests](#answering-requests) | The check path, exact rules, the catch-all |
| [Reports](#reports) | What a client posts to `@hits` |
| [Identifying the client](#identifying-the-client) | The `User-Agent` of its own requests |
| [`spec/v1/`](spec/v1) | The golden sample: normative, every implementation tests against it |

## Implementations

| Platform | Repository |
| --- | --- |
| Caddy | [redirhub/caddy-redirects](https://github.com/redirhub/caddy-redirects) |
| Next.js | [redirhub/nextjs-redirects](https://github.com/redirhub/nextjs-redirects) |
| Cloudflare Workers | [redirhub/cloudflare-redirects](https://github.com/redirhub/cloudflare-redirects) |

RedirHub's rules writer and its edge (which receives the reports) test against the same sample.
Writing a client for another platform (a CMS, a web server, a CDN, a framework)? Implement the
three parts below, test against [`spec/v1/`](spec/v1), keep a copy of it with a CI check, and add
your row to [the list of copies](spec/v1/README.md#copies).

## Rules file format v1

```
@format 1
@refresh 60
@notify https://service.redirhub.com/.well-known/mercure?topic=integration%2F<file token> token=<jwt>
@hits https://rediredge.com/_edge/hits
@check /_redirhub/check
@powered-by RedirHub (integrated)
@host promo.example.com check=EEEXQ4RS report404
@host shop.example.com check=K3Vb9QmX
promo.example.com/old-pricing	https://promo.example.com/pricing
promo.example.com/promo	https://sale.example.com/	302
shop.example.com/*	https://www.example.net/	301	pq
```

UTF-8 text, one item per line; empty lines are ignored, a trailing `\r` is ignored.

### Directives

- `@format 1`: the first line. A file whose major version is not 1 counts as a failed download:
  the last good copy stays in use. A file without `@format` is read as v1.
- `@refresh <seconds>`: the poll interval, clamped to 30–3600 seconds, default 60.
- `@hits <url>`: where reports go. Without it, nothing is reported.
- `@check <path>`: the check path (default `/_redirhub/check`).
- `@powered-by <value>` (optional): the rest of the line, trimmed, is sent verbatim as
  `X-Powered-By` on every redirect from the rules. Printable ASCII, up to 128 bytes.
- `@notify <subscribe url> token=<token>` (optional): a [Mercure](https://mercure.rocks) hub's
  subscribe URL with the file's topic, and a bearer token allowed to subscribe to it. A message on
  the topic means "the file changed, download it now"; nothing in it is trusted. Clients that
  cannot keep a connection open (serverless functions, Workers, PHP) ignore it and poll.
- `@host <hostname> check=<token> [report404]`: one per served hostname. `report404` asks for
  the site's own 404 answers to be reported too. Unknown `key=value` fields are ignored; the first
  `@host` line for a hostname wins.
- Unknown `@` directives are ignored. Every other non-empty line is a rule.

### Rules

Tab-separated columns: `source<TAB>target[<TAB>status[<TAB>flags[<TAB>options]]]`.

- **source**: hostname and path without a scheme, `{host}/path` (percent-encoded where needed);
  `{host}/*` is the host's catch-all. It matches `http` and `https` requests alike. Rules for
  hostnames without an `@host` line are ignored, and so is a source with a scheme, without a path,
  or with a space, `"` or `\`. A query in the source is ignored.
- **target**: an absolute `http` or `https` URL, used as is.
- **status**: 301, 302, 307 or 308; empty means 301. Any other value skips the rule.
- **flags** (may be empty): `p` appends the request path (catch-all only, no double slash), `q`
  keeps the query string (with `?` or `&`). Flags are must-understand: a rule with an unknown flag
  letter is skipped.
- **options**: space-separated `key=value`, may be ignored; further columns are ignored.
- A line that cannot be parsed is skipped; the first rule for a source wins.

### Downloading

- `GET` the rules address every `@refresh` seconds (not per request), with a 5 second timeout, no
  credentials, and `If-None-Match` with the ETag of the copy in use: `304` keeps the copy, `200`
  brings a new body and ETag.
- Any failure (network, timeout, `404`, `5xx`, an unknown format) keeps the last good copy, and
  the client keeps redirecting. Until the first good copy, nothing is redirected and the check
  path is not answered. Clients may store the last good copy (on disk, in a cache) so a restart
  redirects at once.
- Clients may refuse files over 32 MB; RedirHub keeps each hostname under 50,000 rules.

## Answering requests

Only **GET and HEAD** requests on a **hostname listed** with `@host` are answered; everything else
goes on to the site unchanged. A client **fails open**: on any error the request goes on.

1. **Check path**: `200`, `Content-Type: text/plain`, `Cache-Control: no-store`, body
   `token=<check token> etag=<ETag of the rules in use, without quotes or W/>`. Its first answer
   proves the hostname is the site's and connects it; RedirHub calls it regularly to see whether
   the client has the latest rules.
2. **Exact rules** redirect **before** the site's own routes, even when the page exists.
3. Otherwise the site answers. Only when it answers **404** and the hostname has a **catch-all**,
   the 404 is discarded and the catch-all redirects instead. A working page is never taken over.
   A platform that never sees the site's final status (e.g. a Next.js proxy) serves no catch-all.

**Matching.** The hostname in any letter case, port and trailing dot ignored. The path
case-sensitive, compared percent-decoded (bytewise: `%C3%BC`, `%c3%bc` and a raw `ü` are one
path), leading and trailing slashes ignored as on RedirHub's edge (`/old/` and `//old` match
`{host}/old`); the root is `/`. Scheme and query string are ignored.

**Location.** The target as written. Without `q` the request query is dropped; with `q` it is
appended with `?`, or `&` when the target has a query. With `p` (catch-all only) the escaped
request path is appended to the target's path without a double slash. A fragment of the target
stays last. Every redirect from the rules carries `X-Powered-By` when the file has `@powered-by`;
the check path and the site's own answers do not.

## Reports

When the file has `@hits` and the client has the workspace's **integration key**, it reports every
redirect it served (exact and catch-all) and, for hostnames with `report404`, the site's own 404s
that no rule redirected. No other status is reported.

```
POST <@hits>
Authorization: Bearer <integration key>
Content-Type: text/plain
Content-Encoding: gzip        (optional)

<one line per visit>
```

The endpoint answers `204`. A request carries at most 100 lines and 64 KB. Send in the
background, never delaying a response, with a 2 second timeout; a failed or rejected request is
dropped, not retried.

Each line is the Apache combined log format with the absolute URL:

```
203.0.113.9 - - [31/Oct/2026:19:10:00 +0000] "GET https://promo.example.com/old-pricing?utm=x HTTP/1.1" 301 0 "https://www.google.com/" "Mozilla/5.0 …" file=promo.example.com/old-pricing destination=https://promo.example.com/pricing
```

- The visitor's IP address, from a value the platform sets (the connection, `CF-Connecting-IP`,
  a header the site's own proxy overwrites), never a header a client can forge; `-` when unknown.
- The time in UTC, `[dd/Mon/yyyy:HH:mm:ss +0000]`.
- The request: method, the absolute URL (`http://` or `https://` as the visitor asked, the
  hostname as listed: lowercase, no port; the path and query as sent, escaped), the protocol.
  Spaces, `"`, `\` and control characters in the URL are percent-encoded.
- The status, the response body size (`0` for a redirect), the Referer and the User-Agent, each
  cut to 1 KB, `-` when missing, `"` and `\` backslash-escaped, control characters as `\xHH`.
- For a redirect: ` file=<the rule's source column, verbatim>` and ` destination=<the Location
  sent>` (spaces, `"`, `\` and control characters percent-encoded). A 404 line ends after the
  User-Agent.

RedirHub takes the link from `file=` (without a trailing `/*`, percent-decoded) and counts the
line for the key's workspace. No cookies, other headers or bodies are ever sent.

## Identifying the client

Every request a client makes itself, the rules download and each report, should carry a short
`User-Agent` that names the client and its version, so RedirHub can count which integrations are
in use and spot an outdated one:

```
User-Agent: <client>/<version>
```

- One product token: the client's package or repository name, a slash, its version. No comments,
  URLs, platform details or site names; keep it under 40 characters.
- The same value on the rules download and on the reports.
- A client written for one site, without a release of its own (for example from RedirHub's setup
  prompt), sends `<platform>-custom/1`, raising the number when its behaviour changes.

| Client | `User-Agent` |
| --- | --- |
| Caddy module | `caddy-redirects/0.3.2` |
| Next.js package | `nextjs-redirects/0.2.1` |
| Cloudflare Workers package | `cloudflare-redirects/0.2.1` |
| A WordPress plugin written for one site | `wordpress-custom/1` |

This is the header of the client's own requests. The `User-Agent` field inside a report line is
the visitor's, copied from the request the client redirected, and never the client's.

## Versioning

[`spec/v1/`](spec/v1) is normative for format v1: where it and this README disagree, fix one of
them in the same change. A change that old clients would misread is format v2, in `spec/v2/`.
Each implementation keeps a copy of the files it tests with, and a CI step fails when its copy
differs from this repository's `main` ([list of copies](spec/v1/README.md#copies)).

## License

Apache License 2.0, see [LICENSE](LICENSE). Copyright 2026 RedirHub.
