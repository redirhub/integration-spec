# Rules file format v1: the golden sample

These files are **normative for format v1**: one worked example of a workspace rules file, how a
client must answer requests with it, the report lines it sends, and what the edge logs from
them. Every implementation tests against them, so the rules writer, the clients and the edge
cannot drift apart. The prose is in the [README](../../README.md); where the two disagree, fix
one of them in the same change.

| File | What it is |
| --- | --- |
| `rules.txt` | A complete workspace rules file with two hostnames, byte for byte as RedirHub writes it, plus two lines it does not write yet (below). |
| `requests.tsv` | Requests a client gets on a server that loaded `rules.txt`, with the answer it must give and the report line it sends. |
| `reports.txt` | The report lines (the body posted to `@hits`) for the requests of `requests.tsv` that are reported, in order. |
| `reports.expected.tsv` | For each line of `reports.txt`, in order, the fields the edge logs from it. |

All hostnames are reserved `example.test` names; all tokens (`check=`, the `@notify` topic and
JWT) are fake.

## rules.txt

The directives in the order RedirHub writes them (`@format`, `@refresh`, `@notify`, `@hits`,
`@check`, `@powered-by`, `@host` lines sorted by hostname), then the rules, hostname by hostname,
each hostname's lines sorted bytewise. It covers an exact rule (`old-pricing`), the root
(`shop.example.test/`), the catch-all with `pq` (`shop.example.test/*`), statuses 302, 307 and
308, a non-ASCII path written percent-encoded (`preise/%C3%BCber-uns`), a target with a query and
`q` (`promo`), a frame page (`tour`), a link with options a client may not carry out (`vip`:
further destinations, a password, a plugin), and two lines RedirHub does not write today, there to
keep clients forward-compatible:

- `promo.example.test/spring ... 307<TAB><TAB>future=1`: an empty flags column and an unknown
  option. Options are may-ignore: the rule redirects.
- `promo.example.test/beta ... 301<TAB>x`: an unknown flag. Flags are must-understand: the rule is
  skipped and the site answers.

RedirHub's own test renders this file from a seeded workspace and compares it with these two
lines left out (the option dropped from `spring`), and the tokens normalised.

## requests.tsv

Tab-separated, one header line, no quoting (a cell never holds a tab; `-` means none). Columns:

| Column | Meaning |
| --- | --- |
| `name` | What the case shows. |
| `method`, `url` | The request; `url` is absolute (scheme, hostname as sent, maybe a port, path and query as sent). |
| `referer`, `user_agent` | The request's headers. |
| `site` | The status the site itself answers this request with when asked, with an empty body. |
| `outcome` | `redirect` (a rule answers), `frame` (a rule answers with a frame page), `site` (the site's own answer passes through unchanged) or `check` (the check path answer). |
| `status`, `location` | The status and `Location` the client must send. |
| `body` | The check path's body, `token=<check token> etag=<ETag of the rules in use>` (the cases serve `rules.txt` with `ETag: "spec-v1"`); for a frame page `src=<its iframe's src>`, the page holding exactly one `<iframe>` with that `src` (HTML-escaped). |
| `report` | The line of `reports.txt` (1-based) the request is reported as. |

A client that answers frame rules with a `302` to the same address instead (README, "Frame
pages") checks the `frame` cases as redirects: status `302`, the `src` as `Location`, and the
reported line with status `302`.

Every request comes from `203.0.113.9` over `HTTP/1.1` at `31/Oct/2026:19:10:00 +0000`. Every
redirect and frame page from the rules carries `X-Powered-By: RedirHub (integrated)` (`@powered-by`); the site's
own answers and the check path do not. Reporting follows `report404` on the `@host` line (the
default of every client).

## reports.txt

One Apache combined line per reported request, as posted to `@hits`: the absolute URL with the
hostname as listed (lowercase, no port) and the request target as sent, the status, the body
size (`0` for a redirect and a frame page; the site's 404 body, empty here), the Referer and
User-Agent (`-` when missing, `"` and `\` backslash-escaped). A redirect or frame page then
appends ` file=<the rule's source column, verbatim>` and ` destination=<the Location sent, or the
frame's src>`; a 404 appends nothing. A frame page is reported with status `200`.

## reports.expected.tsv

Tab-separated, one header line, `-` for none: `client_ip`, `method`, `url`, `status`, `referer`
and `user_agent` as the edge logs them (escapes kept), `file` (the `file=` value without a
trailing `/*`, percent-decoded: the link's file) and `destination`.

## Copies

Each copy lives in another repository, with a CI step that downloads these files from
`https://raw.githubusercontent.com/redirhub/integration-spec/main/spec/v1/` and fails when its
copy differs:

| Repository | Copy | Files |
| --- | --- | --- |
| `redirhub/caddy-redirects` (Caddy) | `spec/v1/` | `rules.txt`, `requests.tsv`, `reports.txt`, `reports.expected.tsv` |
| `redirhub/nextjs-redirects` (Next.js) | `spec/v1/` | `rules.txt`, `requests.tsv`, `reports.txt` |
| `redirhub/cloudflare-redirects` (Cloudflare Workers) | `spec/v1/` | `rules.txt`, `requests.tsv`, `reports.txt` |
| `redirhub/backend` (the rules writer) | `tests/Fixtures/integration-spec-v1/` | `rules.txt`, `reports.txt` |
| `redirhub/om-script` (the edge) | `redirector/internal/handlers/testdata/spec-v1/` | `reports.txt`, `reports.expected.tsv` |

The backend renders the rules file from a seeded workspace and checks the setup prompts' report
line; the edge's hits handler is fed `reports.txt` and checked against `reports.expected.tsv`.

Change a file here first, then update the copies in the same round of pull requests. A new
implementation adds its row here.
