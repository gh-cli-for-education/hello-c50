## 1. Set up the organization (one-time, on github.com)

The CLI doesn't create the org for you. Do these once via the GitHub web UI:

1. **Create the organization** at <https://github.com/account/organizations/new>.
2. **Create a template assignment repo.** 

Any repo flagged as a template (Settings → "Template repository") works. 
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

![Demo: gh teacher login](images/gh_teacher_auth.gif)

This shells out to `gh auth login -s admin:org` and opens a browser to authorize. If you haven't logged in to `gh` before, it performs the initial login and grants `admin:org` in one shot; if you have, it re-authenticates with the new scope appended.

If you skip this step and have no token at all, the CLI detects the missing token and runs `gh teacher login` automatically. If a token exists but lacks `admin:org`, commands like `gh teacher invite` will fail with an error instructing you to run `gh teacher login` to grant the scope.

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

**Collect token.** Supply a fine-grained PAT with **Contents: read** on org repos whose names match `<classroom>-*`. Store it only via the `CLASSROOM50_COLLECT_TOKEN` environment variable or a hidden stdin prompt — there is no `--collect-token` flag (command-line PATs leak via shell history and process listings). Use an org-owned service account, not a personal teacher account; pass `--service-account-confirm` to silence the reminder. Rotate before expiry (fine-grained PATs support up to 1 year; 90 days is a common rotation interval) with:

```sh
gh teacher rotate-collect-token <org>
```

**What `init` sets up:** org-level member defaults (`default_repository_permission: none` so new members don't get implicit cross-repo access, and `members_can_create_public_repositories: false` so members can't accidentally publish student work — both via a single `PATCH /orgs/{org}`; warns and continues if an enterprise policy locks the fields), private `classroom50` repo with `auto_init`, embedded workflows (`publish-pages.yaml`, `collect-scores.yaml`, reusable `autograde-runner.yaml`), GitHub Pages (workflow build, visibility set to **public** so students can fetch published `assignments.json` unauthenticated; non-default `--autograder` YAML shims, when registered, are also fetched from Pages), branch protection on the default branch, workflow `GITHUB_TOKEN` permissions (409 tolerated when the org enforces a stricter policy — skeleton workflows declare their own workflow-level `permissions:` blocks), reusable-workflow access for other repos in the org (so student shims can `uses:` the runner), and the repo-level `CLASSROOM50_COLLECT_TOKEN` Actions secret.

**Plan check.** `init` warns when the org is not on Team or Enterprise Cloud (required for Pages from a private repo). The warning is advisory; you can still proceed.

After `init` completes, the CLI prints the future Pages URL (`https://<org>.github.io/classroom50/`) and suggests `gh teacher classroom add` as the next step.


### gh teacher init ULL-ESIT-PL-2627

```
➜  c50 gh teacher whoami
crguezl
```

```
➜  ull-esit-pl-2627 gh teacher init ULL-ESIT-PL-2627
Warning: ULL-ESIT-PL-2627: couldn't tighten org member defaults (HTTP 422 (https://api.github.com/orgs/ULL-ESIT-PL-2627)); set them manually at https://github.com/organizations/ULL-ESIT-PL-2627/settings/member_privileges — Base permissions: No permission AND Repository creation: uncheck Public.
ULL-ESIT-PL-2627/classroom50: already exists, continuing setup
ULL-ESIT-PL-2627/classroom50: skeleton already present, skipping commit
ULL-ESIT-PL-2627/classroom50: Pages already enabled
Warning: ULL-ESIT-PL-2627/classroom50: couldn't set Pages visibility to public (HTTP 400: Private pages is not enabled for this repository. All Pages will be public. (https://api.github.com/repos/ULL-ESIT-PL-2627/classroom50/pages)); toggle it manually at https://github.com/ULL-ESIT-PL-2627/classroom50/settings/pages → Visibility if students see 404s on the Pages URL
ULL-ESIT-PL-2627/classroom50: branch protection applied to main (no force-push, no delete)
ULL-ESIT-PL-2627/classroom50: org default workflow permissions are "read"; skeleton workflows grant workflow-level write where needed. To raise the org default: gh api -X PUT /orgs/ULL-ESIT-PL-2627/actions/permissions/workflow -F default_workflow_permissions=write
ULL-ESIT-PL-2627/classroom50: reusable-workflow access enabled (organization)
Note: the collect token should belong to an org-owned service account, not a personal teacher account. Pass --service-account-confirm to silence this notice.
CLASSROOM50_COLLECT_TOKEN (input hidden, ends with Enter): 
ULL-ESIT-PL-2627/classroom50: stored CLASSROOM50_COLLECT_TOKEN
Pages will serve at https://ULL-ESIT-PL-2627.github.io/classroom50/ once publish-pages completes its first run.
Next: gh teacher classroom add ULL-ESIT-PL-2627 <short-name>
```