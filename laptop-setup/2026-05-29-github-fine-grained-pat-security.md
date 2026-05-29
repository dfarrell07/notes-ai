# GitHub Fine-Grained PAT Security Analysis

Threat model: phone with a fine-grained PAT scoped to a single private task-queue
repo. Phone physically compromised (Cellebrite extraction). Attacker gets the PAT.

## 1. Can a fine-grained PAT be scoped to a SINGLE repository?

Yes. When creating a fine-grained PAT, you select "Only select repositories" and
pick exactly one. The token is then restricted to that repo for all granted
permissions.

Minimum permission set for "create issues, read issue comments, nothing else":

| Permission          | Level       | What it enables                                    |
|---------------------|-------------|----------------------------------------------------|
| Issues              | Read/Write  | Create issues, read/write comments, manage labels  |
| Metadata            | Read        | Auto-added when any other permission is selected   |

You cannot do "Issues: Write" without also getting "Issues: Read." The Metadata
permission is mandatory and automatic -- it grants read access to basic repo info
(name, description, languages, topics, license, stars, tags, contributors list).

**Important: Issues Read-only is sufficient for a private repo** if you only need
to read issues and comments but not create them. If you need to create issues,
you must select Issues: Read and Write.

There is no way to grant "create issues" without also granting "read all issues
and comments" -- the permission is all-or-nothing within the read/write tiers.

**The true minimum for "create issues + read comments" is:**
- Issues: Read and Write
- Metadata: Read (auto-added)
- Contents: NOT needed (you do not need contents permission at all)

## 2. Can the attacker access OTHER private repos?

**No, with caveats.**

Fine-grained PATs are scoped to explicitly selected repositories. The attacker
cannot clone, read content, push, or interact with any repo not selected when the
token was created. GitHub returns 404 (not 403) for repos outside the token's
scope, which prevents confirming whether other private repos even exist.

**Known information-disclosure bug (unpatched as of early 2026):** A fine-grained
PAT scoped to "Only select repositories" may still be able to *enumerate/list*
repository names via `GET /user/repos`, even though clone/read/write operations
are correctly denied. This is a reported bug that community members have flagged
as a security issue, but GitHub has not publicly confirmed a fix.

**Critical UI bug (unpatched):** If you ever *edit* a fine-grained PAT through
the GitHub web UI, the "Repository access" radio button silently defaults back to
"All repositories." Clicking "Update token" without noticing this broadens the
scope to every repo the user can access, including all private repos. This
completely undermines the security model. Mitigation: never edit a fine-grained
PAT -- delete and recreate instead.

No known server-side bypass has been publicly disclosed that would allow a
repo-scoped fine-grained PAT to read contents of other repos.

## 3. Can the attacker escalate via GitHub Actions?

**Partially -- depends on the permissions granted.**

Modifying workflow files (`.github/workflows/`) requires TWO specific permissions:
- Contents: Read and Write
- Workflows: Read and Write

If the PAT only has Issues: Read/Write (the recommended minimum), the attacker
**cannot** modify workflow files, trigger workflow dispatches, or interact with
Actions in any way. They cannot push code or modify any files.

**However, if Contents: Write was also granted** (which is NOT needed for the
task-queue use case), the attacker could potentially:
- Modify shell scripts or Makefiles called by existing workflows
- Push to branches that trigger workflows
- Still could NOT modify `.github/workflows/*.yml` files without the separate
  Workflows permission

**Self-hosted runner risk:** If the task-queue repo has self-hosted runners AND
the attacker can trigger workflows (requires Contents + Workflows permissions),
they could execute arbitrary code on any machine running a self-hosted runner for
that repo. This is a well-documented attack path (Praetorian Red Team research,
Shai-Hulud worm). But with Issues-only permissions, this attack is not possible.

**Bottom line for the task-queue scenario:** With only Issues: Read/Write +
Metadata: Read, the attacker cannot interact with GitHub Actions at all.

## 4. Can the attacker discover info about other repos, orgs, or the account?

**Yes, some information leaks.**

The `GET /user` endpoint requires NO permissions and works with any fine-grained
PAT. It returns the **public** profile of the authenticated user:

| Leaked field      | Value                                              |
|-------------------|----------------------------------------------------|
| login             | GitHub username                                    |
| name              | Display name                                       |
| company           | Company field (if set publicly)                    |
| email             | Public email (if set)                              |
| bio               | Profile bio                                        |
| public_repos      | Count of public repos                              |
| public_gists      | Count of public gists                              |
| followers         | Follower count                                     |
| organizations_url | URL template to list orgs (but access depends on   |
|                   | org permission)                                    |

Fine-grained PATs do NOT have a `user` scope equivalent, so private fields like
`total_private_repos`, `owned_private_repos`, `two_factor_authentication`, `plan`,
and `disk_usage` are NOT returned.

**What the attacker can do:**
- Confirm who owns the token (username, name, company)
- See public profile information
- Call `GET /user/orgs` -- but fine-grained PATs have no org-level permissions, so
  this returns only the user's public org memberships (same as any unauthenticated
  visitor could see)
- Call meta endpoints (`GET /meta`, `GET /`, `GET /versions`) -- these require no
  auth and reveal nothing user-specific
- Access any public repos (same as unauthenticated, but at the higher 5,000 req/hr
  authenticated rate limit)

**What the attacker cannot do:**
- List private repos (except the enumeration bug noted in #2)
- See org membership for private orgs
- See private profile fields
- Access internal/private repos beyond the scoped one

**Audit log gap:** The `GET /user` and `GET /user/repos` calls do NOT appear in
GitHub audit logs. An attacker can identify the token owner and probe access
without generating audit signals.

## 5. PAT expiration best practices

Fine-grained PATs support expiration and it is configurable at creation time:

| Option           | Details                                            |
|------------------|----------------------------------------------------|
| 7 days           | Preset option                                      |
| 30 days          | Preset option                                      |
| 60 days          | Preset option                                      |
| 90 days          | Preset option                                      |
| Custom           | Up to 366 days (1 year)                            |
| No expiration    | Allowed, but org/enterprise admins can block this  |

**Organization policy:** Org owners can enforce a maximum lifetime (1-366 days).
The default policy is 366 days maximum, meaning fine-grained PATs cannot be
created with infinite lifetime against org repos unless the admin explicitly
allows it.

**Best practice for the task-queue scenario:**
- Set expiration to 30 days
- Use a secret manager or calendar reminder to rotate
- If org-owned repo, ask the org admin to enforce a short maximum lifetime policy
- Fine-grained PATs do NOT auto-rotate; you must manually create a new one

**There is no auto-renewal mechanism.** When the token expires, it stops working.
You must create a new one.

## 6. Can the PAT be IP-restricted?

**Not at the token level. Only at the organization level, and only on GitHub
Enterprise Cloud.**

GitHub does not support per-token IP restrictions for fine-grained PATs. The IP
allow list is an org/enterprise-level feature:

- Available on GitHub Enterprise Cloud only
- Applies to ALL access methods (web UI, API, Git) for the org
- Uses CIDR notation for IP ranges
- Covers all tokens, not selectable per-token

**For the task-queue scenario:** If the repo is in a GitHub Enterprise Cloud org,
the org admin could restrict API access to known IPs, which would block an
attacker using the PAT from an unknown IP. But for personal repos or repos in
free/team orgs, there is no IP restriction capability.

**Alternative mitigation:** Use a GitHub App instead of a PAT. GitHub Apps
support IP allow lists at the app level.

## 7. Rate limiting

Fine-grained PATs share the standard authenticated rate limits:

| Limit type           | Threshold                                       |
|----------------------|-------------------------------------------------|
| Primary (REST)       | 5,000 requests per hour per user                |
| Secondary            | 100 concurrent requests                         |
| Secondary            | 900 points per minute (REST)                    |
| Content creation     | ~80 per minute, ~500 per hour                   |
| CPU time             | 90 seconds per 60 seconds real time             |

**Can an attacker DoS the task queue?**

Not really. Rate limits are per-user (the token owner), not per-repository. The
attacker would exhaust the token owner's 5,000 req/hr budget, but this only
affects API calls made with that user's credentials -- it does not affect other
users accessing the same repo, and it does not affect the repo itself.

The attacker could create issues at ~80/minute (~500/hour) until rate-limited.
This could spam the task queue but would not prevent other users from accessing
the repo via the web UI or their own tokens.

Exceeding limits returns 403/429 and can result in temporary banning of the
integration.

## 8. Can the attacker create webhooks?

**Only if the Webhooks permission was explicitly granted.**

Creating webhooks requires: Webhooks repository permission with Write access.

If the PAT only has Issues: Read/Write + Metadata: Read (the recommended
minimum), the attacker CANNOT create, modify, or list webhooks.

If Webhooks: Write were granted, an attacker could create a webhook pointing to
their own server and receive real-time notifications about repo events (pushes,
issues, PRs, etc.) -- a serious data exfiltration vector.

**Recommendation:** Never grant Webhooks permission on the task-queue PAT.

## 9. Do GitHub API metadata endpoints leak info?

**Minimal leakage, but some exists.**

Endpoints requiring no permissions (work with any authenticated request):
- `GET /` -- hypermedia links to API resources (not user-specific)
- `GET /meta` -- GitHub's IP ranges, SSH keys, API URLs (not user-specific)
- `GET /versions` -- API version list (not user-specific)
- `GET /octocat` -- ASCII art (not user-specific)
- `GET /user` -- public profile of the token owner (see #4 above)

With the auto-granted Metadata: Read permission, the attacker can also access:
- `GET /repos/{owner}/{repo}` -- repo name, description, language, topics,
  visibility, default branch, star/fork/watcher counts, license, creation date
- Commit history metadata, contributor list, branch/tag names
- Code frequency stats, participation stats
- But NOT file contents (requires Contents permission)

**The Metadata permission does not grant access to other repos.** It only applies
to the single repo the token is scoped to.

## 10. Detecting a compromised PAT

### Built-in GitHub capabilities

**Audit logs (Enterprise Cloud only):**
- All API calls include `token_id` in audit log entries
- You can search by `hashed_token` to find all actions by a specific token
- Filter: `hashed_token:"SHA256_HASH"` in the audit log search
- To get the hash: `echo -n TOKEN | openssl dgst -sha256 -binary | base64`

**Gaps in audit logging:**
- `GET /user` and `GET /user/repos` calls do NOT appear in audit logs
- `git.clone` events only available via REST API / streaming, not the web UI
- User-level endpoint calls generate no audit signal
- Free/Team org plans have very limited audit log capabilities

**Secret scanning:**
- If the PAT is accidentally committed to any public GitHub repo, GitHub's secret
  scanning will detect it and can automatically revoke it
- Push protection can block commits containing PATs before they reach the repo
- Credential Revocation API (GA as of April 2025): anyone can submit a PAT for
  revocation via unauthenticated API (rate-limited to 60 req/hr, 1000 tokens per
  request)

**Token management page:**
- GitHub shows "last used" timestamp on the token management page
  (Settings > Developer settings > Personal access tokens > Fine-grained tokens)
- If you see unexpected "last used" activity, the token may be compromised

### What you should do

1. Set short expiration (30 days) so a compromised token has limited lifetime
2. Monitor the "last used" timestamp on your token management page
3. If you suspect compromise, immediately delete the token from GitHub settings
   (Settings > Developer settings > Personal access tokens)
4. Review the repo's issue list for any unexpected issues created
5. If on Enterprise Cloud, search audit logs for the token's hash
6. Enable email notifications for repo activity to catch anomalous issue creation
7. Consider using a GitHub App instead of a PAT -- apps have better audit trails
   and support IP allow lists

## Summary: Recommended minimum-privilege configuration

For a task-queue repo where you need to create issues and read comments:

```
Repository access: Only select repositories -> [task-queue-repo]
Permissions:
  Issues: Read and Write
  (Metadata: Read is auto-added)
  Everything else: No access
Expiration: 30 days
```

What this grants the attacker if the PAT is compromised:
- Create/read/edit issues and comments in the task-queue repo
- Read basic repo metadata (name, description, topics, stats)
- Identify the token owner (username, public profile)
- Access public repos at authenticated rate limits

What this denies the attacker:
- Access to ANY other private repo
- Reading file contents (even in the task-queue repo)
- Modifying code, workflows, or any files
- Creating webhooks
- Accessing GitHub Actions
- Seeing private profile info, private org memberships
- Escalating to other systems via self-hosted runners

## Sources

- [GitHub: Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [GitHub: Permissions required for fine-grained PATs](https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens)
- [GitHub: Endpoints available for fine-grained PATs](https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens)
- [GitHub: Rate limits for the REST API](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
- [GitHub: Token expiration and revocation](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/token-expiration-and-revocation)
- [GitHub Blog: Introducing fine-grained PATs](https://github.blog/security/application-security/introducing-fine-grained-personal-access-tokens-for-github/)
- [GitHub Blog: Fine-grained PATs are now GA](https://github.blog/changelog/2025-03-18-fine-grained-pats-are-now-generally-available/)
- [GitHub Blog: PAT rotation policies and optional expiration](https://github.blog/changelog/2024-10-18-new-pat-rotation-policies-preview-and-optional-expiration-for-fine-grained-pats/)
- [GitHub Blog: Credential Revocation API GA](https://github.blog/changelog/2025-04-29-credential-revocation-api-to-revoke-exposed-pats-is-now-generally-available/)
- [GitHub Blog: Secret scanning on-demand revocation](https://github.blog/changelog/2024-10-02-secret-scanning-on-demand-revocation-for-github-pats-public-beta/)
- [GitHub: Setting a PAT policy for your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization)
- [GitHub: IP allow lists for organizations](https://docs.github.com/enterprise-cloud@latest/organizations/keeping-your-organization-secure/managing-allowed-ip-addresses-for-your-organization)
- [GitHub: Identifying audit log events by access token](https://docs.github.com/en/enterprise-cloud@latest/admin/monitoring-activity-in-your-enterprise/reviewing-audit-logs-for-your-enterprise/identifying-audit-log-events-performed-by-an-access-token)
- [GitHub Community: Read-only Issues PAT can create issues in public repos](https://github.com/orgs/community/discussions/180063)
- [GitHub Community: Editing PAT silently reverts to All repositories](https://github.com/orgs/community/discussions/188472)
- [Praetorian: Self-hosted GitHub runner to self-hosted backdoor](https://www.praetorian.com/blog/self-hosted-github-runners-are-backdoors/)
- [Detecting Bad Actors via GitHub Audit Logs](https://dxrf.com/blog/2026/02/10/detecting-bad-actors-github-audit-log/)
- [GitGuardian: Fine-grained PAT detection](https://docs.gitguardian.com/secrets-detection/secrets-detection-engine/detectors/specifics/github_fine_grained_pat)
