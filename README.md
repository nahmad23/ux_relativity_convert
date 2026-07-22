# uxa_relativity_v2 — Ansible conversion

Ansible port of the Chef cookbook **`uxa_relativity_v2`** (Relativity e-discovery
platform post-build configuration for Windows Server 2016/2019). It reproduces
every active resource from the cookbook, keeps all variables/parameters, and
converts every ERB template to Jinja2.

## What this deploys

The cookbook configured different Relativity server roles selected by a single
`Function` variable. That dispatch (originally `recipes/default.rb`) is preserved
in `roles/uxa_relativity_v2/tasks/main.yml`:

| Function        | Role                     | Task files run (besides base + harden) |
|-----------------|--------------------------|----------------------------------------|
| `AGT`/`CNV`/`DTS` | Agent / Conversion / DTSearch | `relativity_agent.yml`            |
| `ANA`           | Analytics                | `relativity_analytics.yml`             |
| `APP`/`WRK`     | Worker                   | `relativity_worker.yml`                |
| `SST`           | Secret Store             | `relativity_secretstore.yml`           |
| `SQL`           | SQL                      | `relativity_sql.yml`                   |
| `WEB`/`RDC`     | Web / RDC                | `relativity_web.yml`                    |
| `MSB`           | Message Broker (RabbitMQ)| `relativity_servicebus.yml`            |
| `ELM`/`ELD`/`ELC` | DataGrid               | `relativity_datagrid.yml`              |

Every function also runs `relativity_base.yml` first and `relativity_harden.yml`
last, matching the cookbook.

## Layout

```
site.yml                       # top-level playbook (hosts: relativity)
ansible.cfg                    # inventory path, hash_behaviour = merge
requirements.yml               # ansible.windows, community.windows, community.hashi_vault
inventory/hosts.ini           # one group per Function; WinRM connection vars
group_vars/
  all.yml                      # ALL scalr.variables + vault connection (base defaults)
  rel_sql.yml                  # example per-group override (SQL)
  rel_msb.yml                  # example per-group override (Message Broker)
roles/uxa_relativity_v2/
  defaults/main.yml            # every Chef attribute (directory/software/registry/...)
  vars/main.yml                # SCHANNEL protocol+cipher data; uxo_win_harden overrides
  meta/main.yml                # metadata.rb + dependency mapping notes
  handlers/main.yml            # Restart RabbitMQ handler
  tasks/
    main.yml                   # Function dispatch (recipes/default.rb)
    relativity_base.yml        # recipes/relativity_base.rb
    relativity_agent.yml       # recipes/relativity_agent.rb
    relativity_analytics.yml   # recipes/relativity_analytics.rb
    relativity_worker.yml      # recipes/relativity_worker.rb
    relativity_web.yml         # recipes/relativity_web.rb
    relativity_sql.yml         # recipes/relativity_sql.rb
    relativity_secretstore.yml # recipes/relativity_secretstore.rb
    relativity_servicebus.yml  # recipes/relativity_servicebus.rb
    relativity_datagrid.yml    # recipes/relativity_datagrid.rb
    relativity_harden.yml      # recipes/relativity_harden.rb
    audit.yml                  # recipes/audit.rb (+ attributes/audit.rb)
    service_password.yml       # uxo_service_password placeholder
    _disk_init.yml             # shared Rescan/Init/Format disk helper
    _install_guarded_package.yml   # windows_package + not_if guard helper
    _install_zip_package.yml   # download→unzip→delete→install helper
    _vault_pki_certificate.yml # resources/rel_vault_pki_certificate.rb
  templates/                   # all ERB → Jinja2 (.j2)
```

## Variables

**All variables from the cookbook are preserved.** They live in two places:

* **`roles/uxa_relativity_v2/defaults/main.yml`** — the cookbook's *attribute*
  defaults (`attributes/*.rb`): `directory`, `software`, `network`, `registry`,
  `acl`, `pagefile`, `agent`, `web`, `sql`, `javaheap`, `audit`, etc. The nested
  key layout matches Chef 1:1, so `node['software']['erlang']['installer']['name']`
  becomes `software.erlang.installer.name`.
* **`group_vars/all.yml`** — the per-deployment `scalr.variables.*` values (which
  in Chef came from Scalr / the kitchen.yml suites) and the HashiCorp Vault
  connection (which was the `secrets/vault` data bag). Override these per host or
  per group.

A handful of values were **computed at runtime** in Chef via `powershell_out!`
(total memory, pagefile max/add, the service-user SID, Java heap min/max). Those
cannot be static, so they are calculated in the tasks and exposed as facts
(`pagefile_max`, `pagefile_add`, `serviceuser_sid`, `javaheap_min`,
`javaheap_max`).

## Chef → Ansible resource mapping

| Chef resource | Ansible module |
|---|---|
| `remote_file` (http) | `ansible.windows.win_get_url` |
| `archive_file` (extract) | `community.windows.win_unzip` |
| `directory` | `ansible.windows.win_file` (+ `win_acl` for `rights`) |
| `registry_key` | `ansible.windows.win_regedit` |
| `windows_pagefile` | `community.windows.win_pagefile` |
| `windows_package` | `ansible.windows.win_package` |
| `windows_feature` | `ansible.windows.win_feature` |
| `windows_firewall_rule` | `community.windows.win_firewall_rule` |
| `windows_env` / `windows_path` | `ansible.windows.win_environment` / `win_path` |
| `windows_service` | `ansible.windows.win_service` (handler) |
| `windows_uac` | `win_regedit` (EnableLUA / PromptOnSecureDesktop / ConsentPromptBehaviorAdmin) |
| `group ... :modify/append` | `ansible.windows.win_group_membership` |
| `execute` / `powershell_script` | `ansible.windows.win_command` / `win_shell` / `win_powershell` |
| `file` (content) | `ansible.windows.win_copy` (content:) |
| `template` | `ansible.windows.win_template` |
| `replace_or_add` / `append_if_no_line` (line cookbook) | `community.windows.win_lineinfile` |
| `data_bag_item('secrets','vault')` | `group_vars` `vault:` (encrypt with ansible-vault) |
| `rel_read_vault_secret` | `community.hashi_vault.vault_read` |
| `rel_vault_pki_certificate` | `tasks/_vault_pki_certificate.yml` (`community.hashi_vault.vault_write`) |
| `not_if`/`only_if` guards | a preceding `win_stat`/`win_powershell` check + `when:` |

## Notes / decisions

* **Commented-out installer blocks.** Large parts of the cookbook (the actual
  Relativity / Invariant / Secret Store / CAAT / RDC installer copy-extract-run
  steps, and the BCP share) were commented out in the source recipes. Those are
  preserved as commented tasks and, importantly, **all their ERB templates were
  still converted** (`templates/Response-*.txt.j2`, `Response-CAAT.properties.j2`,
  `Client-Registration.ps1.j2`, `Map-Analytics-Drive.ps1.j2`) so nothing is lost
  and they can be re-enabled.
* **External dependency cookbooks** (`uxs_7zip`, `uxs_notepadplusplus`,
  `uxs_notes`, `uxs_splunk`, `uxo_service_password`, `uxo_win_harden`,
  `uxo_base_hardened`) are not part of this cookbook. Package installs have
  `win_package` stand-ins (marked `ignore_errors` until you point them at real
  sources); the rest are documented in `meta/main.yml` and
  `tasks/service_password.yml` as porting placeholders. The `uxo_win_harden`
  control toggles are kept in `vars/main.yml` (`uxo_win_harden_overrides`).
* **`policy_group`** replaces Chef's `node['policy_group']`. Set it to `local`
  to run the Test-Kitchen-only disk initialisation blocks, or `prod` otherwise.

## Running

```bash
ansible-galaxy collection install -r requirements.yml

# encrypt the vault token / winrm creds first, e.g.:
#   ansible-vault encrypt group_vars/all.yml

ansible-playbook -i inventory/hosts.ini site.yml
ansible-playbook -i inventory/hosts.ini site.yml --limit rel_sql   # one function
```

Populate `inventory/hosts.ini` with real hosts, set WinRM credentials
(`ansible_user`/`ansible_password`) via an encrypted vars file, and replace the
Vault token in `group_vars/all.yml`.
