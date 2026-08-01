+++
date = '2026-08-01T12:00:00+02:00'
draft = false
title = '3 Steps to Versioned API Contracts with Grapity'
description = 'From a spec file in your repo to versioned, drift-checked contracts in every consumer, using a GitHub Action, a commit, and grapity materialize'
tags = ['api', 'governance', 'openapi', 'github-actions', 'ci-cd', 'open-source']
categories = ['projects']
authors = ['msegoviadev']
showHero = true
heroStyle = "basic"
+++

[Grapity](https://grapity.dev) is an open-source registry for API contracts. Producers, the teams that own an API, push versioned specs to it. Consumers, the teams that depend on that API, pull those specs into their own repos. I just finished the example that walks through that whole loop: two folders, `examples/producer/` and `examples/consumer/`, each with one GitHub Actions workflow and a README that tells you exactly what to replace. No more guessing how the pieces fit together from scattered docs.

Writing it forced me to justify why the example needs to exist at all. The honest answer: a spec can drift out from under a consumer with a change that looks harmless, an added optional field, a widened enum, and nothing in the process tells the consuming team it happened. Nobody did anything wrong. The spec changed, and nothing told anyone it had. If you want the broader argument for why that gap matters, I wrote about it separately in [why I built Grapity in the first place](/posts/why-i-built-grapity/).

> **Key Takeaways**
> - A single GitHub workflow on the producer side diffs every PR against the registry and blocks breaking changes before merge
> - Version numbers aren't typed by hand: the registry classifies each change as safe or breaking and derives the semver bump itself
> - `grapity materialize` pins a consumer repo to an exact spec version with a lockfile, and `--check` tells you when that pin has gone stale

Three pieces close that gap, and you already understand all of them: a GitHub Action, a commit, and a CLI command. Here's the flow end to end, producer to consumer.

<figure>
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 800 200" width="100%" role="img" aria-labelledby="flow-diagram-title">
    <title id="flow-diagram-title">Producer to consumer flow: producer repo pushes a versioned spec to the Grapity registry, which the consumer repo pulls via materialize</title>
    <defs>
      <linearGradient id="flowgrad" x1="0%" y1="0%" x2="100%" y2="0%">
        <stop offset="0%" stop-color="#6366f1"/>
        <stop offset="100%" stop-color="#06b6d4"/>
      </linearGradient>
      <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
        <path d="M0,0 L10,5 L0,10 z" fill="#64748b"/>
      </marker>
    </defs>
    <rect x="20" y="60" width="180" height="80" rx="8" fill="none" stroke="url(#flowgrad)" stroke-width="2"/>
    <text x="110" y="95" text-anchor="middle" font-family="system-ui, sans-serif" font-size="15" font-weight="600" fill="currentColor">Producer repo</text>
    <text x="110" y="115" text-anchor="middle" font-family="system-ui, sans-serif" font-size="12" fill="#64748b">spec + contract.yml</text>
    <line x1="200" y1="100" x2="300" y2="100" stroke="#64748b" stroke-width="2" marker-end="url(#arrow)"/>
    <text x="250" y="90" text-anchor="middle" font-family="system-ui, sans-serif" font-size="11" fill="#64748b">push on merge</text>
    <rect x="310" y="60" width="180" height="80" rx="8" fill="none" stroke="url(#flowgrad)" stroke-width="2"/>
    <text x="400" y="95" text-anchor="middle" font-family="system-ui, sans-serif" font-size="15" font-weight="600" fill="currentColor">Grapity registry</text>
    <text x="400" y="115" text-anchor="middle" font-family="system-ui, sans-serif" font-size="12" fill="#64748b">diff, classify, version</text>
    <line x1="490" y1="100" x2="590" y2="100" stroke="#64748b" stroke-width="2" marker-end="url(#arrow)"/>
    <text x="540" y="90" text-anchor="middle" font-family="system-ui, sans-serif" font-size="11" fill="#64748b">materialize</text>
    <rect x="600" y="60" width="180" height="80" rx="8" fill="none" stroke="url(#flowgrad)" stroke-width="2"/>
    <text x="690" y="95" text-anchor="middle" font-family="system-ui, sans-serif" font-size="15" font-weight="600" fill="currentColor">Consumer repo</text>
    <text x="690" y="115" text-anchor="middle" font-family="system-ui, sans-serif" font-size="12" fill="#64748b">grapity.yaml + lockfile</text>
  </svg>
  <figcaption>The three pieces: a producer pipeline that publishes, a registry that versions, a consumer command that pins.</figcaption>
</figure>

## Step 1: Wire the producer pipeline

Your API spec (the OpenAPI or AsyncAPI file describing it, maybe already rendered through something like [Grapity Hub](https://hub.grapity.dev/) if you're juggling more than one protocol) already lives in your repo, and you're reviewing changes to it in pull requests, right? The only thing you add is a GitHub workflow, and you don't even have to write it: [examples/producer/contract.yml](https://github.com/grapitydev/grapity/blob/main/examples/producer/contract.yml) is ready to copy. It diffs your spec against the registry on every pull request and fails on breaking changes; on merge it publishes a new version. The file opens with a commented header listing exactly which variables and secrets to set, so setup is a five-minute copy-paste. This is the same setup the Grapity repo runs for its own registry spec, in [registry-dogfood.yml](https://github.com/grapitydev/grapity/blob/main/.github/workflows/registry-dogfood.yml), through the same [actions/grapity](https://github.com/grapitydev/grapity/blob/main/actions/grapity/action.yml) composite action.

The compatibility report shows up in your CI logs before anything is merged, not after consumers break.

## Step 2: Make a commit

This is the part where you do... nothing new. Change your spec, open a PR, review the compatibility result, merge.

What happens on merge is where the registry earns its keep. It diffs the new spec against the previous version, classifies every change as safe or breaking, and assigns the semver version automatically. Added an optional field? Patch or minor, decided by the rules, not by someone remembering to bump a string. Removed an endpoint? That's breaking: the push is rejected unless the endpoint went through a sunset period or you explicitly force it with a reason.

<!-- [UNIQUE INSIGHT] -->
No manual version bumping. No "did anyone update the version field?" in review comments. The version is derived from what actually changed in the contract, which is the only honest way to version an API.
<!-- /[UNIQUE INSIGHT] -->

## Step 3: Materialize in the consumer repo

On the consumer side, one command pulls the contract into the repo:

```bash
grapity materialize payments-api
```

This writes three things:

- `grapity/specs/payments-api.yaml`: the spec itself, ready for your tooling
- `grapity.yaml`: declares which specs this repo consumes, with the resolved version pinned
- `grapity-lock.json`: records what was requested, what resolved, what's latest, and when it was fetched

Grapity deliberately does not generate code. You keep your generators and your conventions:

```bash
npx openapi-typescript ./grapity/specs/payments-api.yaml \
  --output ./src/generated/payments-api.ts
```

The config-plus-lockfile pair mirrors what your package manager already does: `grapity.yaml` captures intent, `grapity-lock.json` captures resolved state. Always commit both. The materialized spec files are your call: commit them too and contract changes show up as reviewable diffs in your pull requests, or gitignore them and run `grapity materialize` before codegen, locally and in CI. Either way the manifest and lockfile pin the exact version you're building against.

## Bonus: never drift again

Pinning is only half the story. The other half is knowing when the registry has moved forward. That's what `grapity materialize --check` is for: it compares your lockfile against the Grapity registry and reports what's stale, without touching any files.

<figure>
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 800 260" width="100%" role="img" aria-labelledby="drift-diagram-title">
    <title id="drift-diagram-title">Drift check: the consumer lockfile's pinned version is compared against the Grapity registry's latest version, and the result is posted as a pull request comment</title>
    <defs>
      <linearGradient id="driftgrad" x1="0%" y1="0%" x2="100%" y2="0%">
        <stop offset="0%" stop-color="#6366f1"/>
        <stop offset="100%" stop-color="#06b6d4"/>
      </linearGradient>
      <marker id="drift-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
        <path d="M0,0 L10,5 L0,10 z" fill="#64748b"/>
      </marker>
    </defs>
    <rect x="20" y="60" width="200" height="80" rx="8" fill="none" stroke="url(#driftgrad)" stroke-width="2"/>
    <text x="120" y="95" text-anchor="middle" font-family="system-ui, sans-serif" font-size="15" font-weight="600" fill="currentColor">Consumer lockfile</text>
    <text x="120" y="115" text-anchor="middle" font-family="system-ui, sans-serif" font-size="12" fill="#64748b">pinned version</text>
    <rect x="580" y="60" width="200" height="80" rx="8" fill="none" stroke="url(#driftgrad)" stroke-width="2"/>
    <text x="680" y="95" text-anchor="middle" font-family="system-ui, sans-serif" font-size="15" font-weight="600" fill="currentColor">Grapity registry</text>
    <text x="680" y="115" text-anchor="middle" font-family="system-ui, sans-serif" font-size="12" fill="#64748b">latest version</text>
    <line x1="220" y1="100" x2="580" y2="100" stroke="#64748b" stroke-width="2" marker-end="url(#drift-arrow)"/>
    <text x="400" y="90" text-anchor="middle" font-family="system-ui, sans-serif" font-size="11" fill="#64748b">materialize --check compares</text>
    <line x1="400" y1="100" x2="400" y2="190" stroke="#64748b" stroke-width="2" marker-end="url(#drift-arrow)"/>
    <rect x="300" y="195" width="200" height="55" rx="8" fill="none" stroke="url(#driftgrad)" stroke-width="2"/>
    <text x="400" y="218" text-anchor="middle" font-family="system-ui, sans-serif" font-size="14" font-weight="600" fill="currentColor">PR comment</text>
    <text x="400" y="235" text-anchor="middle" font-family="system-ui, sans-serif" font-size="12" fill="#64748b">fresh or stale, updated on each push</text>
  </svg>
  <figcaption>The drift check: pinned version vs. Grapity registry, reported as a sticky pull request comment.</figcaption>
</figure>

Drop [examples/consumer/materialize-check.yml](https://github.com/grapitydev/grapity/blob/main/examples/consumer/materialize-check.yml) into your consumer repo and every pull request gets a sticky comment reporting contract freshness, updated in place on each push, plus inline `::warning::` annotations in the checks UI. It runs on the same composite action as the producer side, this time with `command: check`, and flips to an all-clear note once you re-materialize and push. Against public specs it works with zero credentials, just a `GRAPITY_REGISTRY_URL` variable; authenticated registries add a Keycloak client with the `specs:read` scope, commented into the same file. It warns by default; add `fail-on-stale: "true"` to the inputs to block the pipeline instead (the comment still posts on a red run).

You can see all of this working on a real pull request in the [live example consumer repo](https://github.com/grapitydev/example-consumer-github), including the [drift warning comment](https://github.com/grapitydev/example-consumer-github/pull/2) itself.

## FAQ

### What counts as a breaking change in Grapity?

Removing an endpoint is the clearest example: the registry rejects that push unless the endpoint went through a sunset period first, or you explicitly force it with a reason. Adding an optional field, by contrast, is classified as a patch or minor change and versioned automatically. The rules decide, not a person remembering to flag it in review.

### Do I have to commit the generated spec files to git?

No, but committing them is the better default. `grapity materialize` writes `grapity/specs/*.yaml`, `grapity.yaml`, and `grapity-lock.json`. The manifest and lockfile must be committed either way; the spec files are your call. Commit them and a version bump shows up in review as the actual contract diff, and your builds never need registry access. Gitignore them and you re-run `grapity materialize` before codegen, locally and in CI, to restore the exact pinned versions.

### Does the drift-check workflow need authentication?

Not against public specs: `materialize-check.yml` works with zero credentials beyond a `GRAPITY_REGISTRY_URL` variable. Authenticated registries need a Keycloak client with the `specs:read` scope, which the workflow file's header walks you through setting up.

## Try it out

The two folders I mentioned at the start, [examples/producer/](https://github.com/grapitydev/grapity/tree/main/examples/producer) and [examples/consumer/](https://github.com/grapitydev/grapity/tree/main/examples/consumer), are the fastest way in. Stand up the registry with the [quickstart guide](https://grapity.dev/docs/getting-started/quickstart), copy one producer workflow, materialize in one consumer. The whole thing takes minutes. Everything is open source at [github.com/grapitydev](https://github.com/grapitydev), including the registry server, the CLI, and the [live consumer repo](https://github.com/grapitydev/example-consumer-github) you can crib from.

If you give it a spin, I'd genuinely like to hear what breaks, what's missing, and what governance rule your team would add first. That's how Grapity gets better.
