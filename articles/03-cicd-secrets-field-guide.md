# Hardcoded Secrets in Your CI/CD Pipeline: A Field Guide to Cleaning Up the Mess

*Somewhere in your build YAML, there's a Personal Access Token sitting in plain text. I know because I found one in mine.*

---

Nobody hardcodes secrets on purpose. Nobody wakes up and thinks "today I'm going to commit a Personal Access Token to source control where the entire development team can see it." It happens the way most security problems happen — someone needed the pipeline to work, the proper approach involved three layers of configuration they didn't have time for, and the quick fix became the permanent fix.

I know this because I recently found exactly this situation in a production pipeline. This is the story of finding it, understanding the blast radius, and fixing it properly — with a general approach you can apply to your own CI/CD infrastructure.

## How It Happens

Here's the anatomy of a typical pipeline secret leak. You have a YAML-based build pipeline with multiple stages: build, test, package, deploy. Most stages authenticate using the pipeline's built-in identity — in Azure DevOps, that's `$(System.AccessToken)`, an OAuth token that's automatically generated per run and scoped to the build service account.

But then you hit a stage that needs to do something the build service account can't do by default. Maybe it's triggering a release pipeline in another project. Maybe it's pushing to a repository in a different organization. The built-in token doesn't have the right permissions.

The proper fix involves configuring service principals, adjusting account permissions, or setting up cross-organization trust relationships. That takes time, possibly admin access you don't have, and definitely a conversation with your DevOps team.

The quick fix takes 30 seconds: generate a Personal Access Token from your own account, paste it into the YAML, and the pipeline works. Ship it. Move on. You'll fix it later.

You won't fix it later.

## What I Found

During a routine review of our build configuration, I was scanning through a pipeline YAML file — a couple hundred lines of build stages, PowerShell tasks, and artifact definitions. One stage used the system token correctly for package creation. The next stage, which triggered a QA release pipeline, had a different approach:

```yaml
# Stage 1 - Package Creation (correct)
env:
  PAT: $(System.AccessToken)

# Stage 2 - Release Trigger (not correct)
env:
  PAT: a]3kT9x...the_rest_of_a_real_token
```

A full Personal Access Token, hardcoded in plain text, committed to source control. In a file that every developer on the team could read.

## Assessing the Blast Radius

Before you panic and start rotating every credential in your organization, you need to understand what the exposed token can actually do. Not all PATs are created equal — their scope depends on what permissions were granted when they were created.

Here's the assessment framework I used:

**What API does the token call?** In this case, it was the Release Management API for a specific project. That tells you the minimum scope — the token can create releases in that project.

**What artifacts does it reference?** The release creation payload included build artifacts from the main application, the database project, the test automation suite, and the QA dataset pipeline. That's the reach of a single API call using this token.

**Who created the token?** If it's tied to a developer's personal account (it usually is), it inherits that developer's permissions — which often extend well beyond what the pipeline needs. A senior developer's PAT might have access to every repository and release pipeline in the organization.

**How long has it been in source control?** Check your git history. Every developer who's cloned or pulled that repository has the token in their local history, even after you remove it from the current version.

**Has it expired?** Tokens have expiration dates, and an expired token is less of an active threat but still a policy violation that indicates a process gap.

## The Fix: Zero Manually-Managed Tokens

The goal isn't just to remove this one hardcoded token. The goal is to reach a state where no pipeline depends on a manually-managed credential. Here's the hierarchy of solutions, from best to acceptable:

### For Same-Organization Access

**Use the pipeline's built-in identity.** `$(System.AccessToken)` in Azure DevOps, `${{ secrets.GITHUB_TOKEN }}` in GitHub Actions, the `CI_JOB_TOKEN` in GitLab CI. These are automatically generated, automatically scoped, and automatically expire when the pipeline run ends. No rotation, no management, no risk of committing them to source control.

The catch is that built-in tokens often have limited permissions by default. The fix for that isn't to reach for a PAT — it's to grant the build service account the specific permissions it needs. If the pipeline needs to create releases, grant the build service identity release creation permissions in that project. This is a one-time configuration change, not an ongoing credential management burden.

### For Cross-Organization Access

When a pipeline needs to reach into a different organization, built-in tokens don't work. Your options, in order of preference:

**Service Principals with federated credentials.** If your platform supports identity federation (Azure DevOps with Entra ID, GitHub with OIDC), this is the gold standard. No secrets at all — the pipeline proves its identity through the platform's trust relationship.

**Service account tokens in secret storage.** Create a dedicated service account (not tied to any individual developer), generate a token, and store it in your platform's secret management system — Azure Key Vault, GitHub Secrets, HashiCorp Vault. The pipeline references the secret by name; the actual value never appears in code.

**Variable groups with key vault backing.** In Azure DevOps, variable groups can be linked directly to Azure Key Vault, which gives you centralized rotation and audit logging. The pipeline YAML references the variable name, and the platform resolves it at runtime.

### What You Never Do

Never hardcode tokens in pipeline YAML, script files, configuration files, Dockerfiles, or anywhere else that ends up in source control. "But it's a private repo" is not a security boundary — it's a permissions boundary, and those change over time.

## After the Fix: Clean Up the History

Removing the token from the current version of the file isn't enough. Git remembers everything. The token exists in your commit history, in every clone, in every fork, and in every backup.

Your remediation checklist:

**Revoke the token immediately.** Don't wait until the fix is deployed. Revoke first, fix second. A broken pipeline is better than an exposed credential.

**Audit the token's access.** Check the token creator's permissions and review recent activity. If the token was still active, determine whether it was used outside of the pipeline.

**Rotate related credentials.** If the exposed token could access other systems (and it often can, because PATs tend to be over-scoped), rotate those credentials as well.

**Consider git history cleanup.** For highly sensitive tokens, tools like `git filter-branch` or BFG Repo-Cleaner can rewrite history to remove the secret. But be aware that this is disruptive — it changes commit hashes and requires every developer to re-clone.

**Add preventive controls.** Enable secret scanning in your repository platform (GitHub, Azure DevOps, GitLab all offer this). Add pre-commit hooks that check for patterns matching tokens, API keys, and connection strings. Make it harder for the next quick fix to become a security incident.

## The Bigger Problem

This isn't really a story about one hardcoded PAT. It's a story about the gap between how CI/CD pipelines should be secured and how they actually are in most organizations.

Most pipeline security issues come from the same root cause: authentication was the last thing configured, not the first. The pipeline was built to make software flow from commit to deployment, and the credential management was bolted on afterward — often by whichever developer happened to be debugging the "access denied" error at 6 PM on a Friday.

The fix is cultural as much as technical. Treat pipeline credentials with the same rigor you'd treat production database passwords. Document which service accounts exist, what permissions they have, and when their credentials were last rotated. Review pipeline YAML in pull requests with security as a first-class concern, not an afterthought.

Your CI/CD pipeline is a production system. It has access to your source code, your build artifacts, your deployment infrastructure, and your secrets. Secure it like one.

---

*Gregory Rebisz is a software developer and technical content writer who spends his days building enterprise SaaS and his evenings writing about the lessons learned in the process. He's based in South Africa and available for freelance technical writing.*
