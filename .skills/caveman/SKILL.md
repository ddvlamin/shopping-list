Project-specific instructions for future interactions to interact with this code base. 

**Always use caveman skill**

## 1. Project Purpose

- Set up Docker containers and custom software with Ansible.
- Write custom code to fit needs of **Searching Pi**, a consultancy company.

## 2. Architecture

- Each new system is deployed in Docker containers to Hetzner CX-23 Ubuntu 24.4 located in Nürnberg.
- If a database is needed, reuse postgresql database in Docker container app-db.
- For each new system, add folders and files to backup to backups_paths in Ansible backups role. All databases are automatically backed up.
- Each Docker container is joining a reverse Caddy proxy on 80/443
- An e-mail bridge is exposed on apps_mail_bridge
- Wireguard is needed to access the server

## 3. Key Systems & Services:

following Ansible roles exist, some have Docker containers, for instance app_db role deploys container app-db.
- **app_db**: `app-db`: Shared PostgreSQL Docker database for application stacks.
- **aws**: AWS credentials and regional API keys for the LLM Gateway.
- **backup_validate**: Validate backups.
- **backups**: Encrypted backups of configured files and databases to backup storage on a schedule.
- **common**: Base server configuration: packages, admin access, timezone, SSH hardening, and reboots after upgrades.
- **dns**: dnsmasq configuration for local DNS behavior.
- **docker**: Docker and Docker Compose on the host.
- **gcp**: Google Cloud service account access for the LLM Gateway from local machine.
- **gitea**: `gitea`: Self-hosted Gitea service with database and container configuration.
  - git.searchingpi.com
- **invoice_processor**: `invoice-processor`: Build, credentials, and deployment for the invoice processing service.
- **llm_gateway**: `llm-gateway-apisix`: APISIX-based secure proxy serving LLM models with custom usage token metrics tracking.
  - llm.searchingpi.com
- **m365**: Microsoft 365 apps, certificates, SharePoint access, mail connectors, and related cloud configuration from local machine.
- **odoo**: Odoo 18 Community Edition instances, secrets, Docker configuration, and optional custom/OCA module builds.
  - `searchingpi_odoo18`: circle.searchingpi.com - production
  - `searchingpi_odoo18_test`: odoo-test.circle.searchingpi.com - test
  - `humdot_odoo18`: circle.humdot.com - 2nd production
- **open_webui**: `open-webui`: User-facing chat interfaces integrated with Entra ID OIDC and mounted to SharePoint folders via rclone.
  - chat.searchingpi.com
- **proxy**: `caddy_proxy`: Caddy reverse proxy, app exposure, and Route53-based certificate configuration.
- **rclone_token_refresh**: Scheduled refresh of expired rclone tokens used for sync with Sharepoint.
- **security**: Security baseline: firewall rules, fail2ban, unattended upgrades, and SMTP relay firewall refreshes.
- **smtp_relay**: `smtp_relay`: Local SMTP relay container, credentials, and mail delivery tests for Microsoft 365 and Azure.
- **wireguard**: `wireguard`: WireGuard VPN stack and optional peer sync from Odoo data.
- **wireguard_distribute**: WireGuard client configs and installers, SharePoint upload, and optional email notifications.

## 4. Repository Structure

- **`custom_modules/`**: Custom Odoo 18.0 modules (LGPL-3). Uses "Drive Fetch" patterns to pull documents from SharePoint.
- **`invoice_processor/`**: Python service utilizing Claude Sonnet 4.5 via Azure AI Foundry to parse bills and create Odoo vendor bills via XML-RPC.
- **`social_entry_processor/`**: Python service extracting Belgian accounting entries from PDFs using pdfplumber and Polars.
- **`visa_processor/`**: Python service processing Visa statements from PDFs.
- **`ansible/`**: Deployment, provisioning, and synchronization playbooks/roles.
- **`scripts/`**: Administration scripts (e.g., VPN management, Gitea setup, database marker checks).
- **`templates/`**: Custom email templates (Dutch/English edi invoices).

## 5. Environment & Configuration

### Common Variables
```bash
INVENTORY="ansible/inventories/production/hosts.yml"
SSH_HOST="ops@10.69.0.1"
PASS_CMD="PASSWORD_STORE_DIR=~/.work-password-store pass"
```

### Credentials Store (`pass`)
All keys/secrets must be retrieved via `$PASS_CMD`. Details on paths are described in **`docs/credentials-reference.md`**.
```bash
# Example retrieval:
$PASS_CMD show circle.searchingpi.com/odoo/api-user
```

### Key Ansible Playbooks
```bash

# LLM Gateway deployment (fastest to slowest):
ansible-playbook -i $INVENTORY ansible/playbooks/reload-llm-gateway.yml    # ~5s, fast redeploy of routes, static files, custom Lua, and HTML contents (restarts container on changes)
ansible-playbook -i $INVENTORY ansible/playbooks/deploy.yml --tags llm_gateway  # ~30-40s, full role deploy

# Full Deploy / Refresh Stack
ansible-playbook -i $INVENTORY ansible/playbooks/deploy.yml
```

## 6. Behavioral guidelines to reduce common LLM coding mistakes.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

### 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.


## 7. Behavioral guidelines to reduce common LLM coding mistakes.

### Shell Command Rules
- Double quote all file paths containing spaces: `"path/to/My File.txt"`
- Never use backslash-escaped whitespace: `path/to/My\ File.txt`
- Always validate Bash scripts with ShellCheck.

### Ansible/Jinja2 Template Rules
- Do not use JS template literals `` `text ${var}` `` in `.j2` templates. Use string concatenation instead: `'text ' + var`.
- Every Ansible role must be idempotent. If you run an Ansible role twice and it reports "changed", it is not idempotent.
- Run `ansible-lint` afer modifying an ansible template: `ANSIBLE_COLLECTIONS_PATH=$(ansible-galaxy collection list --format json 2>/dev/null | jq -r 'keys | join(":") | gsub("/ansible_collections"; "")') ansible-lint <playbook>`.