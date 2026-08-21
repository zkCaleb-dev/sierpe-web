---
title: AI assistants & agents
weight: 45
description: Machine-readable docs at /llms.txt, how to point an agent at the API, and why the honesty contract is what makes agent answers trustworthy.
---

Sierpe's API was designed for consumers that cannot shrug — dashboards,
backends, and increasingly, AI agents. This page covers both directions:
teaching an assistant about Sierpe, and letting an agent query a running
instance.

## Machine-readable documentation

The site publishes its documentation in the
[llms.txt](https://llmstxt.org/) convention:

| URL | Contents |
|---|---|
| [`/llms.txt`](/llms.txt) | Index: what Sierpe is, the facts assistants get wrong, links to every page |
| [`/llms-full.txt`](/llms-full.txt) | Every documentation page concatenated into one plain-text file |

Both are generated at build time from the same source files as the pages
you are reading, so they cannot drift. Paste `/llms-full.txt` into any
assistant's context — or point tools that fetch `llms.txt` automatically
at the site root.

For the API itself, give the agent the authoritative contract, not prose
about it:

```text
https://raw.githubusercontent.com/zkCaleb-dev/sierpe/main/docs/openapi.yaml
```

Swagger-literate agents can generate correct calls from that file alone.

## Letting an agent query your instance

An agent needs three things: the base URL, credentials, and one paragraph
of ground rules. A system-prompt block that works:

```text
You can query a Sierpe instance (a self-hosted Stellar contract indexer)
at https://YOUR-INSTANCE. Reads use GET; if HTTP_BASIC_AUTH is set, send
those credentials. Endpoints: /v1/contracts (list),
/v1/contracts/{id}/{events|state|transfers|trustlines|movements}.

Rules you must follow:
- Check `coverage` and `scanStatus` on every response. An empty page with
  partial coverage means "not indexed here", NOT "it never happened" —
  say which, based on the declared range.
- Page with the `cursor` value; never combine a cursor with other filters.
- Movements are evidence of token events, not a balance. Never sum
  amounts across different tokenContractId values — they are raw base
  units of different tokens.
- Amounts are exact integers in raw token units (i128); do not round.
```

Registering contracts (POST/DELETE) needs the `ADMIN_TOKEN` bearer. Only
hand that to an agent if you want it registering contracts on its own;
read-only agents do not need it.

## Why this works better than most APIs

Agents fail loudest when an API leaves silence ambiguous — an empty list
that could mean "nothing exists" or "we did not look". Sierpe never
leaves that ambiguous, by design:

- **Coverage per (contract, kind)** states exactly which ledger range the
  instance can vouch for, so an agent can qualify its answer instead of
  guessing.
- **`scanStatus`** distinguishes a complete scan from a truncated page,
  a range beyond the tip, and history this instance cannot serve.
- **In-band caveats**: the movements endpoint carries its "this is not a
  balance" note in the response itself, where an agent will actually
  read it — not in documentation it never fetched.

The honesty contract was built so that dashboards do not lie. It turns
out to be exactly what keeps language models from lying, too.

## Roadmap

An MCP server — exposing a running instance as tools an assistant can
call natively, instead of raw HTTP — is a natural next step we are
exploring. It is not committed yet; if you would use one,
[say so in Discussions](https://github.com/zkCaleb-dev/sierpe/discussions).
