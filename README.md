# Streamlike integration gem

A [Gemini gem](https://gemini.google.com) that teaches Gemini how to build on Streamlike: which
door to use for what, authentication, catalog reading, playback, security, analytics and feeds.

It is written for the developers and teams who integrate Streamlike into their own products. It
contains nothing confidential: everything here is public documentation, public OpenAPI
descriptions, public SDKs, and behaviour verified against the production platform.

The same content ships as a Claude skill, from the same source and under the same version number:
[streamlike-integration-skill](https://github.com/Streamlike/streamlike-integration-skill).

## Contents

```
GEM.md                What goes in the gem's instructions field
knowledge/            The seven files to attach as knowledge:
                        webservices.md, api.md, player.md, security.md,
                        analytics-and-feeds.md, examples.md, cookbook.md
scripts/              fetch-openapi.sh, openapi_lookup.py — query a 2.5 MB API description
                      without reading it whole. For your terminal; a gem cannot run them
CHANGELOG.md          What moved between versions
```

A gem takes a limited number of knowledge files, which is why the reference material is bundled
into seven rather than shipped one file per subject. Each bundle keeps the original sections and
names them.

## Installing it

1. open [gemini.google.com](https://gemini.google.com) and create a new gem,
2. name it `Streamlike integration`,
3. paste the whole of `GEM.md` into the instructions,
4. attach the seven files of `knowledge/` as knowledge files,
5. save.

Then ask it what you would ask a colleague who knows the platform: "read this playlist server
side", "embed the player with a saved configuration", "why does my token-protected media show a
broken player".

## Using it without a gem

The files are plain Markdown. Point any assistant at `GEM.md`, or read them yourself — they work
as documentation.

**On the command line**, the scripts stand alone:

```bash
scripts/fetch-openapi.sh
scripts/openapi_lookup.py search subtitle
scripts/openapi_lookup.py show /medias/{media_id}/audio-tracks
```

## Before you start building

An integration needs a Streamlike account, its `company_id`, API access enabled, an API key, and —
for a few webservices — your server's IP whitelisted. Accounts are opened by Mediatech/Streamlike;
there is no self-service signup. Ask your account manager. `GEM.md` lists exactly what to request.

## Versions

`VERSION` holds the version of this distribution, `CHANGELOG.md` says what moved between them, and
every release is tagged `v<version>` here on GitHub. The Claude skill is published from the same
source under the same number, so the two never disagree.

## Official resources

- REST API description — https://api.streamlike.com/openapi.json
- Webservices description — https://cdn.streamlike.com/openapi.json
- SDKs — https://github.com/Streamlike (`js-streamlike-sdk`, `php-api-sdk`, `php-ws-sdk`)
- Technical blog, articles for developers — https://www.streamlike.fr/blog/
- Back office — https://bo.streamlike.com
