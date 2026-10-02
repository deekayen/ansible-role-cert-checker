# deekayen.cert_checker

[![CI](https://github.com/deekayen/ansible-role-cert-checker/actions/workflows/ci.yml/badge.svg)](https://github.com/deekayen/ansible-role-cert-checker/actions/workflows/ci.yml) [![Ansible Galaxy](https://img.shields.io/badge/galaxy-deekayen.cert__checker-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/deekayen/cert_checker/) [![Project Status: Inactive – The project has reached a stable, usable state but is no longer being actively developed; support/maintenance will be provided as time allows.](https://www.repostatus.org/badges/latest/inactive.svg)](https://www.repostatus.org/#inactive) ![Apache 2.0 license](https://img.shields.io/badge/license-Apache%202.0-blue)

An Ansible role that checks each Linux or Windows host for a TLS listener on one port, reads the certificate served there, and e-mails a single report of the certificates that expire within a notification window. The report is an HTML table in the message body plus an `expiring_certificates.csv` attachment.

Each host first checks that something listens on `cert_checker_tls_port` at its own `127.0.0.1`, using `wait_for` on Linux or `win_wait_for` on Windows. The certificate is then read with `community.crypto.get_certificate`, connecting to the host's `ansible_facts.fqdn` on that port; see [Known issues](#known-issues) for where that connection comes from. The controller collects matching rows into two temporary files, sends one message for the whole play with `community.general.mail`, and deletes the files.

The Galaxy name is `deekayen.cert_checker`, with an underscore, while the repository is `ansible-role-cert-checker`.

## Requirements

- ansible-core 2.15 or newer on the controller.
- The `community.crypto` and `community.general` collections, plus `ansible.windows` for Windows hosts.
- The Python `cryptography` library, version 1.6 or newer, on the controller. `get_certificate` requires it, and the certificate lookups run there.
- Network access from the controller to each host's FQDN on `cert_checker_tls_port`, and to `cert_checker_email_host` on `cert_checker_email_port`.
- Fact gathering left on. The role branches on `ansible_facts.os_family` and uses `ansible_facts.fqdn` and `ansible_facts.date_time`.
- No privilege escalation. The controller-side tasks set `become: false`, so a play with `become: true` does not try sudo on the controller.

The temporary files, the report, and the cleanup run once per play on the controller through `run_once` and `delegate_to: 127.0.0.1`.

## Supported platforms

`meta/main.yml` declares GenericLinux and Windows, all versions. CI lints the role and runs `ansible-playbook --syntax-check`; it does not apply the role to a host.

## Installation

From Ansible Galaxy:

```bash
ansible-galaxy role install deekayen.cert_checker
ansible-galaxy collection install community.crypto community.general ansible.windows
```

Or pin it in `requirements.yml`:

```yaml
---
roles:
  - name: deekayen.cert_checker
    src: https://github.com/deekayen/ansible-role-cert-checker.git
    scm: git
    version: main

collections:
  - name: ansible.windows
  - name: community.crypto
  - name: community.general
```

```bash
ansible-galaxy install -r requirements.yml
```

## Role variables

| Variable | Default | Description |
| --- | --- | --- |
| `cert_checker_email` | none, required | Comma-delimited list of report recipients. Each entry must look like an e-mail address; the role asserts this. |
| `cert_checker_email_host` | none, required | SMTP relay the controller sends the report through. |
| `cert_checker_email_port` | `25` | SMTP relay port, 1 to 65535. |
| `cert_checker_notification_window` | `45` | Report certificates that expire in fewer than this many days. Must be at least 1. |
| `cert_checker_tls_port` | `443` | Port to check for a TLS listener and read the certificate from, 1 to 65535. |

`cert_checker_email` and `cert_checker_email_host` default to `undef()` with a hint, so the play fails until you set them. `owner_id`, which has no entry in `defaults/main.yml`, prefixes the subject line as `<owner_id> expiration report` and falls back to `Certificate`.

`vars/main.yml` sets the sender, `cert_checker_email_sender`, to `Ansible_cert_checker_by_deekayen`. It is an internal value, and role vars outrank play and inventory vars.

## Behavior

- The report is sent on every run, even when no certificate falls inside the window. In that case the table and the CSV hold only their header rows.
- A Linux host with nothing listening on `cert_checker_tls_port` fails at the listener check after three seconds and drops out of the rest of the play.
- Expiry is counted in whole days from the managed host's clock (`ansible_facts.date_time`).
- Running with `-vv` or more prints each host's raw `get_certificate` result.

## Dependencies

None. The collections are requirements, not role dependencies.

## Example playbook

```yaml
---
- name: Report TLS certificates that expire within 31 days.
  hosts: web_servers
  gather_facts: true

  roles:
    - role: deekayen.cert_checker
      vars:
        cert_checker_email: pki-team@example.internal,oncall@example.internal
        cert_checker_email_host: smtp.example.internal
        cert_checker_notification_window: 31
        owner_id: Web tier
```

The `example.internal` addresses and relay are placeholders.

## Tags

| Tag | Tasks |
| --- | --- |
| `cert_checker_ssh_port_check` | Linux listener check. |
| `cert_checker_winrm_port_check` | Windows listener check. |
| `cert_checker_get_certificate` | Both certificate lookups. |
| `cert_checker_tempfile` | Creating the two temporary files. |
| `cert_checker_body_header`, `cert_checker_csv_header` | Header rows. |
| `cert_checker_set_fact` | Days-to-expiry calculation. |
| `cert_checker_body_row`, `cert_checker_csv_row` | Report rows. |
| `cert_checker_copy` | Copying the CSV to `expiring_certificates.csv`. |
| `cert_checker_mail` | Sending the report. |
| `cert_checker_cleanup` | Deleting the temporary files. |

Input validation in `tasks/assert.yml` is tagged `always`.

## Known issues

- `tasks/main.yml:76` picks the host for the Linux certificate lookup with `crypto_requirement.not_found is defined`. `community.general.python_requirements_info` always returns `not_found`, empty or not, so the lookup always runs on the controller. Certificates are read across the network from the controller, not from the host's loopback, so a listener the controller cannot reach fails that host.
- `tasks/main.yml:87-97`, the fallback lookup, registers `cert` even when it is skipped. On a Linux host that has `cryptography` installed, the fallback is skipped and its skip result replaces the certificate from the first lookup, so that host never appears in the report. Linux hosts without `cryptography`, and Windows hosts, are reported through the fallback. Confirmed on ansible-core 2.21.4 that a skipped task overwrites an earlier registered result.
- `tasks/main.yml:12` and `:22` register `listener`, which no later task reads.
- `tasks/main.yml:133` labels the third CSV column `expiration`, but the rows put the number of days until expiry there. The HTML table labels the same column `days`.

## Development

CI runs on every push to `main` and every pull request (see `.github/workflows/ci.yml`). It installs the collections from `tests/requirements.yml`, runs `ansible-lint --profile production`, and syntax-checks `tests/test.yml`. To run the same checks locally:

```bash
pip3 install ansible-lint
ansible-galaxy install -r tests/requirements.yml
ansible-lint --profile production
mkdir -p .ansible/roles && ln -sfn "$PWD" .ansible/roles/deekayen.cert_checker
ANSIBLE_ROLES_PATH=.ansible/roles:~/.ansible/roles ansible-playbook --syntax-check tests/test.yml -i tests/inventory
```

The repository also has a `.pre-commit-config.yaml`; run `pre-commit run --all-files` before pushing.

### Repository layout

| Path | Purpose |
| --- | --- |
| `tasks/main.yml` | Listener checks, certificate lookups, report files, e-mail, and cleanup. |
| `tasks/assert.yml` | Recipient, port, and window validation, tagged `always`. |
| `defaults/main.yml` | Every user-facing variable with a default. |
| `vars/main.yml` | The report sender name. |
| `meta/argument_specs.yml` | Argument spec, including `owner_id`. |
| `tests/` | Syntax-check playbook, inventory, and collection requirements used by CI. |
| `.github/workflows/` | `ci.yml` for lint and syntax check, `release.yml` for Galaxy import. |

## Releases

Pushing a git tag runs `.github/workflows/release.yml`, which imports the tagged commit into Ansible Galaxy as `deekayen.cert_checker`. The import needs a `GALAXY_API_KEY` repository or organization secret.

## License

Apache 2.0. See [LICENSE](LICENSE).

## Author

[David Norman](https://github.com/deekayen). Sponsorship links are in [.github/FUNDING.yml](.github/FUNDING.yml).
