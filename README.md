| authors | Matthew Morcaldi (mmorcaldi99@gmail.com) |
| ------- | ---------------------------------------- |
| state   | draft                                    |

# ReadME

## What

steps connecting Grafana to Auth0 OIDC.

## Why

to show prequisite steps to setup testing environment as well as steps for Github Actions. 

## File Structure

| File | Purpose |
|------|---------|
| `versions.tf` | Provider config |
| `variables.tf` | domain, environment, API identifier, scopes |
| `main.tf` | Resource configs for Auth0 & Grafana |
| `outputs.tf` | Client ID/secret, audience, token endpoint |
| `terraform.tfvars.example` | sample of `terraform.tfvars` structure |

## Preqs

install the following on your mac for local testing.

```bash
brew install terraform   # tf install
brew install jq          # JSON parsing credential flow validation
brew install direnv      # for .envrc env vars
brew install gh          # install GitHub CLI
brew install grafana     # install grafana
```

authorize github.

```bash
gh auth login
git remote set-url origin https://github.com/heyseus1/supah-awesome-teleporter.git
```

Connect TF to auth0.

Terraform requires an M2M app authorized from the Auth0 [application section of the admin console](https://manage.auth0.com/dashboard/us/dev-20hu8r2wgzbkc0sf/applications). Select **Create → Machine to Machine** and authorize it against the Management API client access with the following scopes:

```
read:tenant_settings
update:tenant_settings
read:clients
create:clients
update:clients
delete:clients
read:client_keys
update:client_keys
read:connections
update:connections
read:guardian_factors
update:guardian_factors
```

once generated extract your OIDC credentials and store your Env Vars in a .envrc file.

```bash
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
touch .envrc
```

paste the contents below in the .envrc.

```bash
export AUTH0_DOMAIN="your-tenant.us.auth0.com"
export AUTH0_CLIENT_ID="..."
export AUTH0_CLIENT_SECRET="..."
```

then run.

```bash
direnv allow
```

## Verify the client-credentials flow

validate access token.

```bash
curl -s -X POST "https://$AUTH0_DOMAIN/oauth/token" \
  -H 'content-type: application/json' \
  -d '{"client_id":"'"$AUTH0_CLIENT_ID"'","client_secret":"'"$AUTH0_CLIENT_SECRET"'","audience":"https://'"$AUTH0_DOMAIN"'/api/v2/","grant_type":"client_credentials"}' \
  | jq '.access_token != null'
```

validate scopes.

```bash
curl -s -X POST "https://$AUTH0_DOMAIN/oauth/token" \
  -H 'content-type: application/json' \
  -d '{"client_id":"'"$AUTH0_CLIENT_ID"'","client_secret":"'"$AUTH0_CLIENT_SECRET"'","audience":"https://'"$AUTH0_DOMAIN"'/api/v2/","grant_type":"client_credentials"}' \
  | jq -r '.access_token' \
  | jq -R 'split(".") | .[1] | @base64d | fromjson | .scope'
```

## Local Run

```bash
terraform init
terraform plan
terraform apply
```

## Github Actions

The steps above are for local testing and debugging. The MVP runs entirely via GitHub Actions.

Store credentials in [GitHub Actions secrets](https://github.com/heyseus1/supah-awesome-teleporter/settings/secrets/actions) under **Settings → Secrets and variables → Actions** as the EnVars. 

```bash
AUTH0_DOMAIN
AUTH0_CLIENT_ID
AUTH0_CLIENT_SECRET
```

The GH action workflow will be located in .github/workflows/terraform.yml Pushing or opening a PR triggers the workflow, which runs the same Terraform commands. reference [Terraform template](https://github.com/heyseus1/supah-awesome-teleporter/new/main?filename=.github%2Fworkflows%2Fterraform.yml&workflow_template=deployments%2Fterraform) for design structure tips. 

```yaml
name: 'Terraform'

# Triggers: run on pushes to main AND on any pull request.
on:
  push:
    branches: [ "main" ]
  pull_request:

# Least-privilege: this workflow's token can only read repo contents.
permissions:
  contents: read

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest

    # Auth0 credentials, pulled from GH Actions secrets as Envars.
    env:
      AUTH0_DOMAIN: ${{ secrets.AUTH0_DOMAIN }}
      AUTH0_CLIENT_ID: ${{ secrets.AUTH0_CLIENT_ID }}
      AUTH0_CLIENT_SECRET: ${{ secrets.AUTH0_CLIENT_SECRET }}

    defaults:
      run:
        shell: bash

    steps:
      # Pull the repo code onto the runner.
      - name: Checkout
        uses: actions/checkout@v4

      # Install the Terraform CLI on the runner.
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      # Initialize the working directory.
      - name: Terraform Init
        run: terraform init

      # Fail the run if any .tf file isn't formatted properly.
      - name: Terraform Format
        run: terraform fmt -check

      # run a plan.
      - name: Terraform Plan
        run: terraform plan -input=false

      # On push to "main", build or change infrastructure according to Terraform configuration files
      # Note: It is recommended to set up a required "strict" status check in your repository for "Terraform Cloud". See the documentation on "strict" required status checks for more information: https://help.github.com/en/github/administering-a-repository/types-of-required-status-checks
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve -input=false
```