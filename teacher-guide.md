## 1. Set up the organization (one-time, on github.com)

The CLI doesn't create the org for you. Do these once via the GitHub web UI:

1. **Create the organization** at <https://github.com/account/organizations/new>.
2. **Create a template assignment repo.** 

Any repo flagged as a template (`Settings → "Template repository"`) works. 
**The template must be public** so students can read it: 
the "No permission" baseline that `gh teacher init` applies in step 3 blocks org members from reading private repos they aren't explicit collaborators on, and a private template would 404 on `gh student accept`. 

- The Free and Team plans don't have a way around this. 
- (GitHub Enterprise Cloud has a third visibility called "internal" that all enterprise members can read without per-repo collaboration; on that plan an internal template works without going public — see [GitHub's docs on internal repositories](https://docs.github.com/en/enterprise-cloud@latest/repositories/creating-and-managing-repositories/about-repositories#about-internal-repositories).) 
- See [Assignment Templates](Assignment-Templates) for the expected file structure; copy that layout into your own template repo.

The base permission ("No permission") and public-repo-creation lockdown are now applied by `gh teacher init` (step 3 below) — no manual web-UI tweak required for those.

## 2. Log in with the right scopes

Org invitations require the `admin:org` OAuth scope, which `gh auth login` doesn't grant by default. Run once:

```sh
gh teacher login
```

![Demo: gh teacher login](https://github.com/foundation50/classroom50/wiki/images/gh_teacher_auth.gif)

This shells out to `gh auth login -s admin:org` and opens a browser to authorize. If you haven't logged in to `gh` before, it performs the initial login and grants `admin:org` in one shot; if you have, it re-authenticates with the new scope appended.

If you skip this step and have no token at all, the CLI detects the missing token and runs `gh teacher login` automatically. If a token exists but lacks `admin:org`, commands like `gh teacher invite` will fail with an error instructing you to run `gh teacher login` to grant the scope.

### Login

I am using the `GITHUB_TOKEN` environment variable:

```
➜  hello-c50 git:(main) env | grep -i github_to
GITHUB_TOKEN=ghp_your-github-token-goes-here
➜  c50 gh teacher whoami
crguezl
```


## 3. Bootstrap the classroom50 config repo

Run once per teaching org to create `<org>/classroom50` — the private config repo that will hold classroom metadata, published assignment manifests, and collected scores:

```sh
CLASSROOM50_COLLECT_TOKEN=github_pat_... gh teacher init <org>
```

Or omit the env var and the command prompts for the token interactively:

```sh
gh teacher init <org>
```

`init` is idempotent: re-running picks up where a prior run left off (it does not overwrite teacher edits to the skeleton).

**Collect token.** 

- Supply a fine-grained PAT with **Contents: read** on org repos whose names match `<classroom>-*`. 
- Store it only via the `CLASSROOM50_COLLECT_TOKEN` environment variable or a hidden stdin prompt
— there is no `--collect-token` flag (command-line PATs leak via shell history and process listings). 
- Use an org-owned service account, not a personal teacher account; pass `--service-account-confirm` to silence the reminder. Rotate before expiry (fine-grained PATs support up to 1 year; 90 days is a common rotation interval) with:

```sh
gh teacher rotate-collect-token <org>
```

See [organization-token.md](organization-token.md) for details on generating a fine-grained PAT for an organization.

**What `init` sets up:** 
- org-level member defaults (`default_repository_permission: none` so new members don't get implicit cross-repo access, and `members_can_create_public_repositories: false` so members can't accidentally publish student work — both via a single `PATCH /orgs/{org}`; 
- warns and continues if an enterprise policy locks the fields), private `classroom50` repo with `auto_init`, embedded workflows (`publish-pages.yaml`, `collect-scores.yaml`, reusable `autograde-runner.yaml`), 
- GitHub Pages (workflow build, visibility set to **public** so students can fetch published `assignments.json` unauthenticated; 
- non-default `--autograder` YAML shims, when registered, are also fetched from Pages), branch protection on the default branch, workflow 
- `GITHUB_TOKEN` permissions (409 tolerated when the org enforces a stricter policy — skeleton workflows declare their own workflow-level `permissions:` blocks), 
- reusable-workflow access for other repos in the org (so student shims can `uses:` the runner), and the repo-level `CLASSROOM50_COLLECT_TOKEN` Actions secret.

**Plan check.** `init` warns when the org is not on Team or Enterprise Cloud (required for Pages from a private repo). The warning is advisory; you can still proceed.

After `init` completes, the CLI prints the future Pages URL (`https://<org>.github.io/classroom50/`) and suggests `gh teacher classroom add` as the next step.


### gh teacher init ULL-ESIT-PL-2627

```
➜  ull-esit-pl-2627 gh teacher init ULL-ESIT-PL-2627
Warning: ULL-ESIT-PL-2627: 
  couldn't tighten org member defaults (HTTP 422 (https://api.github.com/orgs/ULL-ESIT-PL-2627)); set them manually at https://github.com/organizations/ULL-ESIT-PL-2627/settings/member_privileges 
  — Base permissions: No permission AND Repository creation: uncheck Public.
ULL-ESIT-PL-2627/classroom50: already exists, continuing setup
ULL-ESIT-PL-2627/classroom50: skeleton already present, skipping commit
ULL-ESIT-PL-2627/classroom50: Pages already enabled
Warning: 
  ULL-ESIT-PL-2627/classroom50: couldn't set Pages visibility to public 
  (HTTP 400: Private pages is not enabled for this repository. 
  All Pages will be public. (https://api.github.com/repos/ULL-ESIT-PL-2627/classroom50/pages)); 
  toggle it manually at https://github.com/ULL-ESIT-PL-2627/classroom50/settings/pages 
  → Visibility if students see 404s on the Pages URL
ULL-ESIT-PL-2627/classroom50: branch protection applied to main (no force-push, no delete)
ULL-ESIT-PL-2627/classroom50: org default workflow permissions are "read"; skeleton workflows grant workflow-level write where needed. 
To raise the org default: 
    gh api -X PUT /orgs/ULL-ESIT-PL-2627/actions/permissions/workflow -F default_workflow_permissions=write
ULL-ESIT-PL-2627/classroom50: reusable-workflow access enabled (organization)

Note: the collect token should belong to an org-owned service account, not a personal teacher account. 
Pass --service-account-confirm to silence this notice.

CLASSROOM50_COLLECT_TOKEN (input hidden, ends with Enter): 
ULL-ESIT-PL-2627/classroom50: stored CLASSROOM50_COLLECT_TOKEN
Pages will serve at https://ULL-ESIT-PL-2627.github.io/classroom50/ once publish-pages completes its first run.
Next: gh teacher classroom add ULL-ESIT-PL-2627 <short-name>
```

See the resulting repo at [ULL-ESIT-PL-2627/classroom50](https://github.com/ULL-ESIT-PL-2627/classroom50/tree/main)

## 4. Add a classroom

Each classroom is a directory at the root of `<org>/classroom50` holding four files:

- `classroom.json` — public name / term / org metadata.
- `assignments.json` — assignment manifest (published via Pages, fetched by `gh student accept` and by the autograde-runner workflow on every submission).
- `students.csv` — private roster.
- `scores.json` — private collected scores.

Plus, optionally:

- `autograder.py` at the classroom root — the **classroom default autograder**, used by every assignment that doesn't have its own override. Drop it via `gh teacher autograder set-default` (no scaffold by default — classrooms work without one, the runner just publishes a vacuous-pass result).
- `autograders/<slug>/` subdirectories — **per-assignment overrides**. One folder per assignment slug containing `autograder.py` (the entrypoint) and any sibling fixtures or helpers.

Foundation50-managed pieces (the runner-side bootstrap `.github/scripts/runner.py`, the runner workflow, the publish-pages allow-list) live at the org level, not per-classroom; the autograder shim that lands in each student repo is embedded in `gh-student` and never has to be edited by a teacher.

Scaffold one with:

```sh
gh teacher classroom add <org> <short-name> --name "<full name>" --term <term>
```

For example:

```sh
gh teacher classroom add cs50-fall-2026 cs-principles --name "CS Principles" --term Spring-2026
```

The `<short-name>` must match `^[a-z0-9][a-z0-9-]{1,38}$` (2-39 chars, lowercase letters/digits/hyphens, starting with a letter or digit) because it flows into student repo names like `<short-name>-<assignment>-<username>`. `--name` and `--term` are optional but recommended — they're written into `classroom.json` and surface in the published Pages site (forthcoming) and in `gh teacher download` summaries.

The command commits all four paths in a single Tree commit on the default branch. If `<org>/classroom50` doesn't exist yet, it prints `run gh teacher init <org> first` and exits non-zero. If the `<short-name>` directory already exists, it refuses to overwrite rather than clobbering an in-progress classroom — modify it via `gh teacher roster add` (step 6) and `gh teacher assignment add` (step 7) instead.

Run this command once per classroom you teach in the org. You can have several classrooms side by side in the same `classroom50` repo.

###  Example

```
  ull-esit-pl-2627 gh teacher classroom add ULL-ESIT-PL-2627 ull-esit-pl-2627 --name 'Procesadores de Lenguajes 26/27' --te
rm 'segundo-cuatrimestre-26-27'
ULL-ESIT-PL-2627/classroom50: added classroom ull-esit-pl-2627 (4 files)
View at https://github.com/ULL-ESIT-PL-2627/classroom50/tree/main/ull-esit-pl-2627
Next: gh teacher roster add ULL-ESIT-PL-2627 ull-esit-pl-2627 <username>
```

## 5. Invite students to the org

The fastest way to add students is `gh teacher roster add` (next step) — it registers them in the classroom roster *and* sends an org invite in one shot. Use the bare `gh teacher invite` only for ad-hoc cases (e.g., inviting a TA who shouldn't be in the student roster, or bringing in someone before the roster is set up):

```sh
gh teacher invite <org> <username>
```

![Demo: gh teacher invite](https://github.com/foundation50/classroom50/wiki/images/gh_teacher_invite.gif)

The student gets an email invitation. They can accept it by visiting `https://github.com/<org>`, or skip ahead and let `gh student accept` auto-accept the pending invite when they accept their first assignment.

Common API failures (missing scope, not an admin, org not found, already a member, pending invite) surface as actionable messages instead of raw HTTP errors.

To invite a teaching assistant as an org admin instead:

```sh
gh teacher invite --admin <org> <username>
```

To invite someone to a single repo rather than the whole org (e.g. a TA on a specific assignment):

```sh
gh teacher invite <org>/<repo> <username>                 # default: push
gh teacher invite -p maintain <org>/<repo> <username>     # other permissions
```

Permission options for `-p`: `pull`, `triage`, `push`, `maintain`, `admin`. Re-running with a different `-p` updates the existing collaborator's permission in place.

### Example

```
  hello-c50 git:(main) ✗ gh teacher invite ULL-ESIT-PL-2627 casiano-rodriguez
ULL-ESIT-PL-2627: invited casiano-rodriguez as direct_member
Advise casiano-rodriguez to sign in to https://github.com as casiano-rodriguez, then visit https://github.com/ULL-ESIT-PL-2627 to accept the invitation at the top of the page.
```

### gh roster import help

```
➜  hello-c50 git:(main) ✗ gh teacher roster import --help
Read <path-to-csv> and upsert every row into
<org>/classroom50/<classroom>/students.csv. 
```

The local CSV header must be 

   `username,first_name,last_name,email,section`

(the canonical 5 columns). 

- A trailing `github_id` column is accepted but its value is ignored 
— the CLI re-resolves github_id from `GET /users/{username}` at import time 
so the on-disk roster always carries the GitHub-authoritative ID. 
- The `email` column may have empty values per row.

The whole file is written in one Tree commit, not one `PUT`
per row, so partial-import states can't appear on the repo.
After the commit lands, any student who isn't already in
the org (and doesn't have a pending invite) is invited.

```
Usage:
  gh-teacher roster import <org> <classroom> <path-to-csv> [flags]

Examples:
  gh teacher roster import cs50-fall-2026 cs-principles ./section-1.csv

Flags:
  -h, --help   help for import

Global Flags:
  -v, --verbose   Show operational details (per-step API/git output)
```