## Ansible <> Palo Alto Networks Strata Cloud Manager (SCM)

An Ansible automation example that authenticates to Palo Alto Networks Strata Cloud Manager API & completes tasks

### What this repo does

- Authenticates to SCM using OAuth2 client credentials and sets a bearer token for subsequent API calls
- Provides a simple, local `localhost` inventory and a minimal project structure to add new tasks if needed

### Project structure

- `tasks/authenticate.yml`: Re-usable task file that exchanges client credentials for an access token and sets `api_access_token`
- `group_vars/all/vars.yml`: Non-secret variables (API base URLs, scope)
- `group_vars/all/vault.yml`: Encrypted secrets (e.g., client credentials header); managed via Ansible Vault
- `inventory`: Local inventory targeting `localhost`
- `.vault_pass.example`: Example .vault_pass file that you can close & put a vault password inside of.
- `create_security_rule.yml`: Main playbook that authenticates and creates the security rule

### Prerequisites

- Ansible installed (easiest way is through python & pip)
- Access to an SCM tenant
- OAuth2 client ID/secret for SCM
- Ansible Vault available for storing secrets

### Configuration

1. Populate non-secret variables in `group_vars/all/vars.yml`:

```yaml
api_auth_url: "https://auth.apps.paloaltonetworks.com/oauth2/access_token"
api_base_url: "https://api.sase.paloaltonetworks.com"
api_scope: "tsg_id:<your_tsg_id>"
```

2. Store secrets in `group_vars/all/vault.yml` with Ansible Vault. The authenticate task expects an `Authorization` header value in `api_auth_token` for the token endpoint. For PANW OAuth2 Client Credentials, this is typically `Basic <base64(client_id:client_secret)>`.

Create and encrypt the secret value:

```bash
# Build the Basic auth header value
echo -n "<client_id>:<client_secret>" | base64

# Create/update the vault file (you will be prompted for a vault password)
ansible-vault edit group_vars/all/vault.yml
```

Inside the editor, add:

```yaml
api_auth_token: "Basic <paste_base64_value_here>"
```

Optionally, store your Vault password in a local file (this is what I do):

```bash
echo "<your_vault_password>" > .vault_pass
chmod 600 .vault_pass
```

Security note: Do not commit `.vault_pass`. It's already added to the gitignore, but just as an FYI

3. Inventory

The repo includes a simple `localhost` inventory in `inventory`. No changes are needed for local runs.

### Run the playbook

Using a vault password file:

```bash
ansible-playbook -i inventory create_security_rule.yml --vault-password-file ./.vault_pass
```

### What gets created

The playbook posts a rule named "Ansible Rule" with these defaults:

### Customizing the rule

Edit `create_security_rule.yml` and adjust the `body` values under the `ansible.builtin.uri` task to fit your policy and environment (e.g., `folder`, `action`, `application`, `service`).

### How authentication works

- `tasks/authenticate.yml` sends a POST to `api_auth_url` with `grant_type=client_credentials` and `scope=api_scope`
- It uses the `Authorization: {{ api_auth_token }}` header (from Vault)
- The task registers the response and sets `api_access_token` to `auth_response.json.access_token`
- Subsequent API calls include `Authorization: Bearer {{ api_access_token }}`

### TLS validation

The playbook sets `validate_certs: true`. If you are testing against endpoints with self-signed certs, change this to `false` temporarily. Prefer trusted certs in production.

### Notes

- Keep secrets exclusively in `group_vars/all/vault.yml`
- Ensure `.vault_pass` (if used) is ignored by `.gitignore` and not committed
- Update `api_scope` to match your tenant scope (`tsg_id:<your_tsg_id>`)

---

If you have suggestions or run into issues, feel free to open an issue or PR.
