| authors | Matthew Morcaldi (mmorcaldi99@gmail.com) |
| ------- | ---------------------------------------- |
| state   | draft                                    |

# RFD 1 - Auth0 Tenant Automation

## What

To leverage GitHub Actions using Terraform to configure Auth0 and Grafana via OIDC

## Why

To use IaC to allow readable and repoducible configuration setting rather than clickops.
Thus removing manual credential settings and enforcing MFA 

## Scope

**In scope:** Three systems.

1. Github Action hosted Tf configures the tenant and enforces phishing resistant MFA for apps.
2. Tf adds Grafana as an OIDC app.
3. A golang script that provisions users via API.

**Out of scope (deliberate cuts):**

- Level 4 (Temporal, Docker Compose, Go CLI, e2e tests).

## Infrastructure Configuration

Terraform manages:

- `auth0 tenant` — settings and MFA policy.
- `auth0 guardian` — enrollment factors (Authn enforced, vulnerable factors disabled).
- `auth0 client` — Grafana app.
- `auth0 client credentials` — Grafana client creds.
- `auth0 connection` - login connection to Grafana.

Grafana consumes the client ID, client secret, and the tenant's
`/authorize`, `/oauth/token`, and `/userinfo` endpoints via its
`[auth.generic_oauth]` configuration.

## APIs Used

TF operations interact with the Auth0 Management API. Credentials are two
m2m apps with exact scopes. 

**Terraform M2M** — manages resources

```
read:tenant_settings   update:tenant_settings
read:clients           create:clients        update:clients   delete:clients
read:client_keys       update:client_keys
read:connections       update:connections
read:guardian_factors  update:guardian_factors
```

**User-registration M2M** — Registers users via golang:

```
read:users   create:users
```

neither credential can perform the other's job, so a leak limits blast radius. 


## Security Considerations

**Phishing-resistant authentication is the primary control.** Passwords and
OTP codes factors are prone to phishing; WebAuthn / passkeys are not. The tenant
enforces Authn as the primary factor and disables risky factors (SMS,
email OTP, push). Enforcement is at the tenant level rather than individual 
applications.

**Approval gate.** Infrastructure changes require a pull request review.
Terraform runs `plan` on pull requests so the change will be reviewable before it
touches the tenant, and `apply` runs only on merge to main. `-auto-approve`
answers Terraform's own prompt in CI; human approval happens at PR review,
so no change will be implemented without review.

**Credential handling.** All credentials will be stored in GH Actions secrets and
injected as EnVars. No secret is written to a
`.tfvars` file, committed to the repo, or printed to workflow logs. The two M2M
credentials are scoped as above (least privilege).


## Edge Cases

- **Idempotency.** Terraform is idempotent by design. The provisioning
  script will check if the user exists before creating.
- **Secret rotation.** Rotating a client secret or M2M credential is stored via enVars, code
  change is not required.
