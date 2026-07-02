| authors | Matthew Morcaldi (mmorcaldi99@gmail.com) |
| ------- | ---------------------------------------- |
| state   | draft                                    |

# ReadME

## What

steps connecting Grafana to Auth0 OIDC.

## Why

to show prequisite steps to setup testing environment. 

## File Structure

| File | Purpose |
|------|---------|
| `versions.tf` | Provider config |
| `variables.tf` | domain, environment, API identifier, scopes |
| `main.tf` | Resource configs for Auth0 & Grafana |
| `outputs.tf` | Client ID/secret, audience, token endpoint |
| `terraform.tfvars.example` | sample of `terraform.tfvars` structure |

## Preqs

install the following on your mac.

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

## Run

```bash
terraform init
terraform plan
terraform apply
```