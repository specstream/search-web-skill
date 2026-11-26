# SpecStream Pack for Aeon

A community skill pack for [Aeon](https://github.com/aeonfun/aeon). It gives an Aeon agent live web search through the [SpecStream](https://specstream.ai) Search API.

| Skill | What it does | Default schedule |
| --- | --- | --- |
| `specstream-web-search` | Searches the live web for the skill's `var` and delivers a short answer with the page it was read from, then the top sources | on demand |

## Install

```bash
bin/install-skill-pack specstream/search-web-skill --path aeon
```

Then set up the key:

1. Create a SpecStream API key under Agents & API at https://specstream.ai/workspace. New accounts start with $10 of credit.
2. Add it as the repository secret `SPECSTREAM_API_KEY`.
3. Add `"SPECSTREAM_API_KEY":${{ toJSON(secrets.SPECSTREAM_API_KEY) }}` to the `ALL_SECRETS` blob in `.github/workflows/aeon.yml` and `.github/workflows/messages.yml`. Aeon injects only the keys named there.

The skill installs disabled. Enable it in `aeon.yml` and pass the query as its `var`.

## What it costs

One search of up to 10 results is $0.005, and a search that returns nothing is not charged.

## Without a key

The skill does not fail. It runs the same query through Aeon's built-in WebSearch and says in the output that it did, with the reason.
