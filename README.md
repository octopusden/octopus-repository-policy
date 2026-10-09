# octopus-repository-policy

Repository policy for octopusden: branch rulesets, required status checks and repository settings,
applied by the repo provisioning job.

Nothing here is applied by this repository itself. The provisioning scripts in
`releng/gh-permissions-granting` check this repository out and reconcile octopusden repositories
against it, through TeamCity:

| Build | When | Does |
|---|---|---|
| **Validate Repository Policy** | every push to a pull request here | dry-run of the PR's policy against every managed repository; result on the PR as the `policy/validate` status |
| **Apply Repository Policy** | every change to `main` | applies `main` to every managed repository — rulesets and settings |
| **Create Octopusden Repo** | by hand, one repository | create or sync, along with everything else that job provisions |
| **Sync Repository Policy** | by hand | the fleet-wide run, dry-run or apply |

Changing the policy therefore means a pull request here, and **merging it applies it to every
managed repository** — see [After merging](#after-merging). The review is the pull request: its
approvals and its `policy/validate` report.

- [Layout](#layout)
- [Which repositories are managed](#which-repositories-are-managed)
- [Which rulesets a repository gets](#which-rulesets-a-repository-gets)
- [How to](#how-to)
- [Testing a change](#testing-a-change)
- [After merging](#after-merging)
- [Reference](#reference)

## Layout

```
policies/
  main-protection.json    baseline, applied to every managed repository
  checks-common.json      policy/checks-common   -> gate/merge
  checks-build.json       policy/checks-build    -> build/integration-a
  checks-wl.json          policy/checks-wl       -> security/wl-a
  checks-sonar.json       policy/checks-sonar    -> sonar / sonar/analysis
registry.json             which rulesets a repository gets
settings.json             repository settings, by visibility
```

## Which repositories are managed

Only repositories carrying exactly one **class topic** — `hybrid-flow` or `public-flow` — are managed.
The rest of the account belongs to other teams: the scripts refuse to create or sync them and the
fleet-wide sync skips them without a single call.

The class comes from the topic alone. A repository with both class topics is an error and nothing on
it is changed until its topics are fixed. Fixing a wrong class is a topic change on the repository,
not a change here.

## Which rulesets a repository gets

| The repository has | Rulesets | Required checks |
|---|---|---|
| any class topic | `main-protection` | none — 2 approvals, code-owner review, dismiss stale reviews, linear history, no force-push, no deletion |
| `hybrid-flow` | `policy/checks-common`, `policy/checks-build`, `policy/checks-wl` | `gate/merge`, `build/integration-a`, `security/wl-a` |
| `public-flow` | `policy/checks-common`, `policy/checks-wl` | `gate/merge`, `security/wl-a` |
| `gradle` or `maven` topic | `policy/checks-sonar` | `sonar / sonar/analysis` |

In order: `main-protection`, then the rulesets of its class (`classes`), then those of any matching
topic (`topic_overlays`), then per-repository `additions`, minus per-repository `waivers`.

`checks-sonar` also decides SonarCloud: a repository whose final list holds it gets a SonarCloud
project on its next create or sync, and one that doesn't gets none. Waiving `checks-sonar` therefore
also stops Sonar provisioning for that repository.

**A required check is only worth requiring where something produces it.** GitHub waits indefinitely
for a check that never arrives: no error, no timeout, the pull request just stays blocked. Before
requiring a check for a class, confirm every repository of that class produces it (open a recent PR
in each and look at its checks), or give the ones that don't a waiver.

## How to

Every change below is a pull request here. Run a dry-run of it before merging
([Testing a change](#testing-a-change)).

### Require a new check for a whole class

Example: every `hybrid-flow` repository must also pass `GitGuardian Security Checks`.

1. Either add the context to an existing overlay the class already gets — here
   `policies/checks-common.json` — or create a new overlay, `policies/checks-gitguardian.json`:

   ```json
   {
     "name": "policy/checks-gitguardian",
     "target": "branch",
     "enforcement": "active",
     "conditions": {
       "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] }
     },
     "rules": [
       {
         "type": "required_status_checks",
         "parameters": {
           "required_status_checks": [
             { "context": "GitGuardian Security Checks" }
           ],
           "strict_required_status_checks_policy": false
         }
       }
     ]
   }
   ```

   The file stem and the name must match (`checks-gitguardian` → `policy/checks-gitguardian`).
   `context` is the check's name exactly as it appears on a pull request.

2. Reference it from the class in `registry.json`:

   ```json
   "classes": {
     "hybrid-flow": ["checks-common", "checks-build", "checks-gitguardian"],
     "public-flow": ["checks-common"]
   }
   ```

   To tie it to a topic instead of a class, add it under `topic_overlays`
   (`"kotlin": ["checks-gitguardian"]`).

### Require a check on one repository only

```json
"additions": [
  { "name": "octopus-foo", "rulesets": ["checks-build"], "note": "has a TeamCity build of record" }
]
```

`name` is the repository name without `octopusden/`. Tightening needs no justification; `note` is
for the reader.

### Relax one repository, temporarily

```json
"waivers": [
  { "name": "octopus-foo", "drop": ["checks-build"],
    "reason": "no TeamCity build posts build/integration-a yet",
    "ticket": "CD-1234", "review_by": "2026-12-31" }
]
```

A waiver drops overlays the repository would otherwise get — never `main-protection`. `reason` and
`review_by` (`YYYY-MM-DD`) are required; once the date has passed, every run on that repository
reports the waiver as overdue. Remove the entry when the reason no longer holds.

### Change what every repository gets

`policies/main-protection.json` applies to every managed repository: approvals, code-owner review,
linear history and so on. Change it with care — the next sync rewrites it everywhere.

Self-merge bypasses are not declared here. The provisioning job adds the logins listed in its own
`self-merge-bypass.txt` to `main-protection` at run time, as `bypass_mode: pull_request`.

### Stop requiring a check

- **One context out of an overlay:** delete it from that overlay's `required_status_checks`.
- **A whole overlay:** remove its stem from `registry.json`, then delete the file.

The next sync drops it only after everything the repository should still require has landed, so a
check moving from one ruleset to another stays required throughout.

### Change a repository setting

Edit `settings.json`. Example — let pull requests auto-merge on every repository:

```json
"repo": {
  "all": { "allow_auto_merge": true }
}
```

Settings are written on every create, sync and fleet-wide `apply`, so the policy value wins over
any change made by hand in the GitHub UI.

## Testing a change

**On the pull request.** Every push to a pull request here runs *Validate Repository Policy*: a
dry-run of your branch's policy against every managed repository, reading only. It reports on the PR
as `policy/validate`; open *Details* for the build and its `policy_sync_report.txt` artifact, which
lists per repository the rulesets it would create, update or delete, the classic branch protection
being replaced, and any check that classic protection required which the policy does not. Read that
before approving: it is what the merge will do.

`policy/validate` is red whenever anything fails — a policy that does not load, a script error, or a
repository the sync could not handle (for example one carrying both class topics). A red status
blocks the merge.

**Locally, before pushing** — optional. You need a checkout of `releng/gh-permissions-granting`
next to your branch of this repository.

```bash
cd gh-permissions-granting
# the file checks every run starts with -- no token needed
python3 -c 'import create_repo as cr, pathlib; cr.load_policy(pathlib.Path("../octopus-repository-policy")); print("policy OK")'

# the same dry-run the PR gets -- needs the octopusden token, as rulesets, classic
# protection and settings are only readable by the owner
export GITHUB_TOKEN=$(vault kv get -field=OCTOPUS_REPO_CREATOR_TOKEN octopus/GitHub)
python3 create_repo.py --all --policy-dir ../octopus-repository-policy
```

## After merging

A change to `main` starts *Apply Repository Policy*, which applies it to every managed repository —
no further approval. Its `policy_sync_report.txt` lists what it changed; a red build means at least
one repository could not be brought in line, and the report says which and why. Applies run one at a
time, and running one again right after reports no changes.

Everything else stays as it was: a repository created or synced through *Create Octopusden Repo*
gets the policy of `main` at that moment.

## Reference

### `policies/*.json`

GitHub [repository ruleset](https://docs.github.com/en/rest/repos/rules#create-a-repository-ruleset)
bodies. Checked before anything is written:

| Rule | Why |
|---|---|
| `main-protection.json` must exist | it is the baseline every managed repository gets |
| `name` is `main-protection` for the baseline, `policy/<file stem>` for every other file | the prefix is how the scripts tell their rulesets from hand-made ones, which they never touch |
| `target` is `branch` | |
| `rules` is a non-empty list of objects | |
| a `required_status_checks` rule has `parameters.required_status_checks` as a list of objects | |
| overlays (every file but the baseline) hold only `required_status_checks` rules | review and branch rules live in the baseline alone |
| no file declares `bypass_actors` | a bypass lets its user skip every rule of that ruleset; it is added to the baseline only, by the scripts |

Other mistakes in the ruleset body itself come back from GitHub as HTTP 422 on the first repository
it is written to.

### `registry.json`

Unknown top-level keys are rejected, so a typo fails instead of being ignored.

| Key | Shape | Meaning |
|---|---|---|
| `classes` | `{ "<topic>": ["<stem>", ...] }` | class topics and the overlays each class gets. A repository must carry exactly one of these topics to be managed |
| `topic_overlays` | `{ "<topic>": ["<stem>", ...] }` | overlays added when the repository carries the topic, on top of its class |
| `additions` | `[{ "name", "rulesets", "note"? }]` | overlays added to one repository |
| `waivers` | `[{ "name", "drop", "reason", "review_by", "ticket"? }]` | overlays removed from one repository. `reason` and `review_by` (`YYYY-MM-DD`) are required |
| `retired_rulesets` | `["<ruleset name>", ...]` | rulesets the scripts used to create under other names; deleted on sync once the policy's own have landed. Cannot name `main-protection` or a `policy/*` ruleset |

Every `<stem>` must be a file in `policies/`. Topics must be valid GitHub topics (lowercase letters,
digits, hyphens).

### `settings.json`

Grouped by the endpoint that writes them; within a group, by repository visibility.

| Group | Written with | Fields |
|---|---|---|
| `repo` | `PATCH /repos/{owner}/{repo}` | [repository fields](https://docs.github.com/en/rest/repos/repos#update-a-repository) — merge methods, auto-merge, delete-on-merge, issues, wiki… |
| `security` | `PATCH /repos/{owner}/{repo}` | the `security_and_analysis` object |
| `code_scanning_default_setup` | `PATCH /repos/{owner}/{repo}/code-scanning/default-setup` | [default setup](https://docs.github.com/en/rest/code-scanning/code-scanning#update-a-code-scanning-default-setup-configuration), e.g. `state` |
| `actions_workflow` | `PUT /repos/{owner}/{repo}/actions/permissions/workflow` | [workflow permissions](https://docs.github.com/en/rest/actions/permissions#set-default-workflow-permissions-for-a-repository) |

Each group takes `all`, `public` and `private` blocks. For a repository, `all` applies first and its
own visibility's block overrides it. A group that declares nothing for a repository's visibility is
skipped for it — that is how private repositories skip secret scanning and code-scanning default
setup, which GitHub does not offer them. Unknown groups and scopes are rejected, but field names
inside a block are not checked — and GitHub silently ignores a field it doesn't know, so a misspelt
setting simply never takes effect. Check names against the linked GitHub docs.
