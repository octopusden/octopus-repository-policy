# octopus-repository-policy

Repository policy for octopusden: branch rulesets, required status checks and repository settings,
applied by the repo provisioning job.

Nothing here is applied by this repository itself. The provisioning job (`create_repo.py`, run as
*Create Octopusden Repo* on TeamCity) checks this repository out and reconciles one octopusden
repository against it on every create or sync. Changing the policy therefore means a reviewed pull
request here, and the change reaches a repository on its next create or sync — or everywhere at
once through *Sync Repository Policy* (dry-run first, then apply).

## Layout

```
policies/
  main-protection.json    baseline, applied to every repository
  checks-common.json      policy/checks-common   -> gate/merge
  checks-teamcity.json    policy/checks-teamcity -> Build Validation
  checks-sonar.json       policy/checks-sonar    -> sonar / sonar/analysis
registry.json             which rulesets a repository gets
settings.json             repository settings, by visibility
```

`policies/*.json` use GitHub's
[ruleset schema](https://docs.github.com/en/rest/repos/rules#create-a-repository-ruleset) — a
mistake there comes back from GitHub as a 422. `registry.json` and `settings.json` are our own
formats; the provisioning job validates them before it writes anything, and rejects unknown keys so
a typo fails loudly instead of being ignored.

## Which rulesets a repository gets

A repository is classified by its own GitHub topics:

| Topic | Rulesets | Required checks |
|---|---|---|
| every managed repository | `main-protection` | none — 2 approvals, code-owner review, dismiss stale reviews, linear history, no force-push, no deletion |
| `hybrid-flow` | `policy/checks-common`, `policy/checks-teamcity` | `gate/merge`, `Build Validation` |
| `public-flow` | `policy/checks-common` | `gate/merge` |
| `gradle` or `maven` | `policy/checks-sonar` | `sonar / sonar/analysis` |

Only repositories carrying a class topic are managed at all; the rest of the account belongs to
other teams and the provisioning job does not touch them. `classes` in `registry.json` are mutually
exclusive: a repository carrying both `hybrid-flow` and `public-flow` is an error, and nothing on it
is changed until its topics are fixed. `topic_overlays` add on top of the class — `checks-sonar`
there also decides which repositories get a SonarCloud project.

Topics can be edited by anyone with push access, so on sync a repository that has posted
`Build Validation` is treated as `hybrid-flow` whatever its topics say, and the report asks for the
topic to be fixed. Relabelling a repository cannot drop its TeamCity gate.

A required check is only worth requiring where something produces it. GitHub waits indefinitely for
a check that never arrives, and the pull request stays blocked, so a class should only require what
every repository of that class runs. On sync, the provisioning report lists every required check a
repository has not produced on its recent pull requests.

### Per-repository changes

Tightening needs no justification:

```json
"additions": [
  { "name": "octopus-foo", "rulesets": ["checks-teamcity"], "note": "why" }
]
```

Loosening does. A waiver drops overlays a repository would otherwise get, never `main-protection`,
and must carry a reason and a review date; the report flags it once that date has passed:

```json
"waivers": [
  { "name": "octopus-foo", "drop": ["checks-sonar"],
    "reason": "Sonar onboarding pending", "ticket": "CD-1234",
    "review_by": "2026-12-31" }
]
```

### Adding a ruleset

Add `policies/<stem>.json` named `policy/<stem>`, holding only `required_status_checks` rules, then
reference `<stem>` from `registry.json`. Do not declare `bypass_actors` in any file: the provisioning
job adds its self-merge list to `main-protection` only, so that a bypass can never skip a required
check.

### Removing a ruleset

Take it out of `registry.json`. The next sync deletes any `policy/*` ruleset a repository should no
longer have, but only after every ruleset it should have has landed, so the gate is never down in
between. Rulesets named outside `main-protection` and `policy/*` are someone's hand-made rules and
are never touched — except names listed in `retired_rulesets`, which the provisioning job used to
create itself.

## Repository settings

`settings.json` groups settings by the endpoint that writes them — `repo`, `security`,
`code_scanning_default_setup`, `actions_workflow` — and each group by visibility: `all`, then
`public` or `private`, where the visibility-specific value wins. A group with nothing declared for a
repository's visibility is skipped; that is how private repositories skip secret scanning and code
scanning, which GitHub does not offer them.
