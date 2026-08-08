# docs/

Scratch/planning area, not a user-facing docs site and not where GraphQL
docs get generated (despite `lib/tasks/graphql_docs.rake` defaulting its
`output_dir` to `docs` — that task hasn't been run against this directory
recently; the two files actually here predate it). Currently holds:

- `android-passkey-assetlinks.local.json` /
  `android-passkey-assetlinks.production.json` — the Digital Asset Links
  payloads served by `AndroidAssetlinksController` (see
  `config/routes.rb`'s `.well-known/assetlinks.json` route) for passkey/App
  Links verification against `com.mmroz.homestead`. These are real runtime
  data, not documentation — don't move them without updating the controller.

Planning/implementation-plan markdown docs (e.g. design docs for an
in-progress feature) also land here from time to time; they're
point-in-time notes for humans/agents, not maintained reference docs — feel
free to add one when doing significant design work, but don't treat an
existing one as still-accurate without checking it against the current code.
