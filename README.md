# A CI/CD pipeline without stored secrets

This pipeline gets every secret it needs from Idira Secrets Manager at run
time. GitHub stores none of them.

- **No copy in GitHub.** The pipeline's scan token, registry login, signing
  key and webhook URL live in the vault only. Nobody pastes them into the
  repo's settings, so a leak of those settings exposes no secret.
- **No key to log in.** Each run proves who it is with a short-lived token
  that GitHub makes for that job. Nobody stores a password or an API key for
  the pipeline.
- **Each environment gets only its own secrets.** A dev run reads the dev
  secrets. Only a run from `main` reads the prod secrets.
- **One change, in one place.** When a secret changes in the vault, the next
  run gets the new value. Nothing changes in GitHub.
- **Every fetch is recorded.** Idira's audit log shows which repo and which
  environment read each secret, and when.

> **Every secret value in this repo is fake.** The pipeline stages are
> simulated, and the log prints each value on purpose, only to prove that it
> arrived. Each address uses the reserved domain `.example`, so it can never
> point at a real system.

## The four workflows

Run them in this order, from the Actions tab.

| Workflow | What it shows |
| --- | --- |
| [1. Before: secrets stored in GitHub](.github/workflows/1-before-stored-secrets.yml) | The usual way: the secrets are copies in the repo's settings. GitHub masks them in the log, but any workflow in the repo can read them, and one line of script prints them anyway. Here they are placeholders |
| [2. Show what GitHub tells Idira](.github/workflows/2-show-token-claims.yml) | The job's identity: the claims in the token that GitHub makes for the job |
| [3. After: secrets fetched from Idira](.github/workflows/3-after-idira-secrets.yml) | The same pipeline, with every secret fetched from Idira at run time. A push to `dev` runs it with the dev secrets. A push to `main` runs it with the prod secrets |
| [4. Dev asks for a prod secret](.github/workflows/4-dev-asks-for-prod.yml) | The dev pipeline asks for a prod secret, and Idira refuses. The run goes green when the refusal happens |

Compare workflow 1 and workflow 3 side by side. The stages are the same, and
they read the same environment variables. Each `secrets.NAME` is gone. In its
place, a step fetches the value from Idira:

```yaml
- name: "Scan: fetch the SonarQube token from Idira"
  uses: cyberark/conjur-action@f15cb4a7a519b66fd902e615840dd881c01839c5 # v3.0.1
  with:
    url: ${{ vars.CONJUR_URL }}
    account: ${{ vars.CONJUR_ACCOUNT }}
    authn_id: ${{ vars.CONJUR_AUTHN_ID }}
    audience: idira-secrets-manager
    secrets: data/vault/${{ vars.SAFE }}/fake-sonarqube-token/password|SONAR_TOKEN
```

The settings are GitHub variables, not secrets: the vault's address and the
authenticator's name are not secret. The "after" pipeline uses no GitHub
secret at all.

## How a run gets its secrets

1. The job asks GitHub for a token. The workflow allows this with
   `permissions: id-token: write`. GitHub signs the token, and it is valid for
   a short time only.
2. The job sends the token to Idira Secrets Manager. Idira checks GitHub's
   signature and the token's issuer, so it knows that the token came from
   GitHub.
3. Idira reads the token's `sub` claim. It names the org, the repo and the
   environment:

   ```
   repo:panw-isp-ca@330504653/idira-cicd-without-stored-secrets@1436298515:environment:dev
   ```

   Idira looks for a workload identity of that name. No identity, no login.
   The numbers are GitHub's IDs for the org and the repo. GitHub never reuses
   them, so a renamed or re-created repo with the same name does not match.
4. Idira checks two more claims on every identity: the org's ID
   (`repository_owner_id`) and the branch (`ref`). The dev identity logs in
   only from the `dev` branch, and the prod identity only from `main`.
5. Each identity may read one set of secrets. The dev identity reads the dev
   safe, and the prod identity reads the prod safe.

The job picks its environment from the branch:

```yaml
environment: ${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}
```

Each GitHub environment has a variable, `SAFE`, with the name of its safe, so
one workflow file serves both environments. The `prod` environment runs only
from `main`, and the prod identity in Idira requires `main` too.

## Why the dev pipeline sees "404"

When the dev pipeline asks for a prod secret, Idira answers HTTP 404, "not
found", and not 403, "forbidden". To a caller with no rights on a secret,
Idira does not confirm that the secret exists. So nobody can learn which
secrets are real by asking for them.

## Where the secrets come from

The secrets live in Idira Privilege Cloud, the vault. A security team creates
and changes them there. Idira copies each change to Secrets Manager, where the
pipeline reads it, in about a minute.

| Stage | Secret | Path in Secrets Manager |
| --- | --- | --- |
| Scan | SonarQube token | `data/vault/fake-payments-ci-<env>/fake-sonarqube-token/password` |
| Publish | Registry username and password | `data/vault/fake-payments-ci-<env>/fake-registry-push/username` and `.../password` |
| Sign | Signing key | `data/vault/fake-payments-ci-<env>/fake-signing-key/password` |
| Notify | Slack webhook URL | `data/vault/fake-payments-ci-<env>/fake-slack-webhook/password` |

`<env>` is `dev` or `prod`. The pipeline's safes hold only what the pipeline
needs, not the passwords that the app uses at run time.

## How it scales

One authenticator in Idira trusts the GitHub org. Each repo adds one workload
identity per environment, and the access rules of its safes. The
authenticator stays the same for every repo in the org, and the workflow file
stays the same for every environment.

## Names

Idira Secrets Manager was called Conjur before. The old name stays in the
`cyberark/conjur-action` action and in the `CONJUR_*` variables.

## Learn more

- [GitHub Actions integration](https://docs.cyberark.com/secrets-manager-saas/latest/en/content/integrations/github-actions.htm), Idira Secrets Manager docs
- [Use JWT authentication with GitHub Actions](https://docs.cyberark.com/secrets-manager-saas/latest/en/content/operations/services/cjr-authn-jwt-github-uc.htm), Idira Secrets Manager docs
- [cyberark/conjur-action](https://github.com/cyberark/conjur-action), the action that fetches the secrets
