---
name: specstream-web-search
description: Web search through the SpecStream Search API, with a short answer read from a named page plus ranked sources; needs SPECSTREAM_API_KEY and falls back to WebSearch without it
metadata:
  title: SpecStream Web Search
  mode: read-only
  category: basics
  var: ""
  tags:
    - web
    - search
    - research
  requires:
    - SPECSTREAM_API_KEY
---

> **${var}**: search query or question. When empty, uses a general search for current notable developments across tracked areas.

Today is ${today}. Search the live web for **${var}** through the SpecStream Search API and deliver the results.

## Overview

One `GET https://api.specstream.ai/v1/web-search` call returns two things this skill uses:

- **`answer`**: a short answer with the page it was read from (`answer.value`, `answer.page`). It is present when the query asks for one fact; for a list query it is `null`.
- **`sources`**: the ranked results, each with `title`, `url`, `snippet` and, when the page prints one, `published`.

Search operators hold on every result: `site:`, `-site:`, `-word`, `"exact phrase"`, `filetype:`, `after:` and `before:` (`YYYY`, `YYYY-MM` or `YYYY-MM-DD`). Write them into the query as you would on a search engine.

A call of up to 10 results is $0.005, and a call that returns no results is not charged. The reply's `price_usd` is what the call cost.

## Phase 1: Execute Search

### Key check

Read the "Keys set for this run" line in your run notes, never a shell test (those are refused and cost a turn). If `SPECSTREAM_API_KEY` is not on that line, skip the API call: set `SOURCE_PATH=websearch` with `REASON=NO_KEY` and go to [WebSearch Fallback](#websearch-fallback).

### API Call

`QUERY` = `${var}`, or `current notable developments in AI, crypto, and technology` when it is empty. Make exactly one call, with QUERY written in literally (`--data-urlencode` does the URL encoding):
```bash
./secretcurl -s -o /tmp/specstream-search.json -w 'specstream http=%{http_code} bytes=%{size_download}\n' \
  --max-time 30 -G "https://api.specstream.ai/v1/web-search" \
  --data-urlencode "q=<QUERY>" --data-urlencode "num=10" \
  -H "Authorization: Bearer {SPECSTREAM_API_KEY}" \
  -H "User-Agent: aeonfun-aeon/specstream-web-search"
```

The `-w` label prints HTTP (`000` on a network failure) and the body size.

### Response Processing

On `HTTP=200` with a non-empty body, extract the answer and the sources with one call:

```bash
jq -r '
  ( if (.data.answer.value // "") != "" then [["answer", (.data.query // ""), (.data.answer.page // ""), .data.answer.value, ""]] else [] end )
  + [ .data.sources[]? | ["source", (.title // ""), (.url // ""), ((.snippet // "") | gsub("\\s+"; " ")), (.published // "")] ]
  | .[] | @tsv
' /tmp/specstream-search.json
```

`RESULT_COUNT` = the number of `source` rows it printed (state `Extracted <N> search results`, and whether an answer came back). If `jq` failed, the parse failed and `RESULT_COUNT` = 0. On any other HTTP, `RESULT_COUNT` = 0.

Then decide (in prose, no `if`/`case` block):
- `RESULT_COUNT` above 0: `SOURCE_PATH=specstream`.
- `RESULT_COUNT` = 0: `SOURCE_PATH=websearch`, with `REASON` = `EMPTY` (HTTP 200, parse OK), `PARSE` (HTTP 200, `jq` failed), `NETWORK` (HTTP `000`), else `HTTP_<code>`. Record it with one call, values written in literally:
  ```bash
  echo "specstream fallback=websearch reason=<REASON>"
  ```

### WebSearch Fallback

When `SOURCE_PATH=websearch`, don't end the run empty. Run the same query (`QUERY`) through the built-in **WebSearch** tool and continue to Phase 2 with those results. The log line above (or `reason=NO_KEY` from the key check) is the record of why.

## Phase 2: Format Results

- **Answer** (when present): one line with the value and a link to its page. It leads the output.
- **Each source**: title, URL, snippet, and the `published` date when there is one.
- Drop results with no URL and near duplicates from the same domain.

```
*SpecStream Web Search - ${today}*

Query: "${var}"
Source: ${source}
Answer: <answer.value> ([page](<answer.page>))
Results: ${result_count} found

1. **[Title](URL)**
   Snippet...
   Published: Date

---
API Status: ${http_status}
```

Leave out the `Answer:` line when there is no answer. `${source}` is exactly one of `SpecStream Search API` or `WebSearch (fallback: ${reason})`, matching `SOURCE_PATH`. Never print both.

## Phase 3: Delivery and Logging

### Notification

Send the formatted results via `./notify`: the query, the answer with its page when there is one, the top 5 to 7 sources, and the source path. When the fallback ran, say so and give the reason (e.g. "via WebSearch, SpecStream returned HTTP 401").

### Memory Integration

1. **Log record**: this skill is `read-only`, so the workflow's read-only guard writes its `### specstream-web-search` log entry from your captured output. Don't append to `memory/logs/` yourself; put this record in your **final output**:
   ```
   ### specstream-web-search
   - Query: "${var}"
   - Source: ${source}
   - Answer: yes/no
   - Results: N found, M delivered
   - Status: HTTP ${code}
   ```

2. **Update search memory**: add successful searches to `memory/searches.md` with a one-line `node -e` writer, e.g. `node -e 'require("fs").appendFileSync("memory/searches.md", process.argv[1] + "\n")' "<line>"`.

## Error Handling

Every failure below ends in the [WebSearch fallback](#websearch-fallback), so a run always has results:

- **No key**: `reason=NO_KEY`. Tell the operator to add `SPECSTREAM_API_KEY` as a repository secret and to the `ALL_SECRETS` list in `.github/workflows/aeon.yml` and `messages.yml` (create a key under Agents & API at https://specstream.ai/workspace).
- **Invalid key (401)**: `reason=HTTP_401`. Tell the operator to check `SPECSTREAM_API_KEY`.
- **Out of credit (402)**: `reason=HTTP_402`. The balance is empty or the account's monthly spending limit is reached; tell the operator to add credit.
- **Key limit (403)**: `reason=HTTP_403`. The call would pass the spending limit set on this key.
- **Busy (503 with `Retry-After`)**: the account used its calls for the minute. Nothing is charged; `reason=HTTP_503`.
- **Other server errors (5xx)**: transient, `reason=HTTP_<code>`.
- **Network failures**: `reason=NETWORK`.
- **Malformed responses**: a `200` whose body `jq` can't parse, `reason=PARSE`.
- **Empty results**: a `200` with zero sources, `reason=EMPTY`. If WebSearch is also empty, suggest broader terms.

## Network

The call goes through `./secretcurl` with the `{SPECSTREAM_API_KEY}` placeholder, never a bare `$SPECSTREAM_API_KEY` on the line.

**Shell shape.** Each Bash call is one allowed command (pipes are fine). The run refuses a call that sets a variable (`X=$(...)`), uses `if` / `case` / `[ ... ] &&` tests, contains `$(...)`, or joins commands with `;`, `&&` or `||`. So the key check reads the run notes, the search is one `./secretcurl` call with every value written in literally, and the fallback decision is a prose step. This skill is read-only: any file it keeps is written with a one-line `node -e` writer, never `>` or `>>`.

## Environment Variables

- **`SPECSTREAM_API_KEY`** (required): SpecStream API key, sent as `Authorization: Bearer`. Create one under Agents & API at https://specstream.ai/workspace; a new account starts with $10 of credit. Without it the skill runs WebSearch only.

## Constraints

- **Never expose credentials** in logs or notifications.
- **Always attribute the source**: say whether results came from SpecStream or the WebSearch fallback, and never present fallback results as SpecStream results.
- **Keep the answer's page**: an answer is delivered with the page it was read from, or not at all.
- **Never end a run empty**: on any SpecStream failure, fall back to WebSearch and log `specstream fallback=websearch reason=<REASON>`.

## Skills Integration

Works well with **digest**, **article** and **mention-radar** when they need sourced web context, and beside `you-web-search` when an operator wants a second search provider.
