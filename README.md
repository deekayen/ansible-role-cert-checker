# Certificate expiration checker

[![Molecule](https://github.com/deekayen/ansible-role-cert-checker/actions/workflows/ci.yml/badge.svg)](https://github.com/deekayen/ansible-role-cert-checker/actions/workflows/ci.yml) [![Project Status: Inactive – The project has reached a stable, usable state but is no longer being actively developed; support/maintenance will be provided as time allows.](https://www.repostatus.org/badges/latest/inactive.svg)](https://www.repostatus.org/#inactive)

This is a role used to inspect remote machines for listening TLS services based on their localhost loopback. This solves the problem of doing remote scans for services, which may be filtered to only listen to specific load balancers or security groups in public clouds.

## Requirements


## Role Variables

```
# Comma-delimited list of email addresses
cert_checker_email: ''

# Hostname of an email relay offering SMTP services
cert_checker_email_host: ''

# WHich port to use on email_host for SMTP
cert_checker_email_port: 25

# In days, send alerts for any certificates which expire in fewer than this
cert_checker_notification_window: 45

# Which port to interrogate for a TLS listener
cert_checker_tls_port: 443
```

## Dependencies


## Example Playbook

    ---

    # cert_checker.yml
    # Blame David Norman

    - name: Query for expiring certificates on TLS listeners.
      hosts: platform_undefined:linux:rhel7_64Guest:rhel8_64Guest
      # gather_facts to query the ansible_fqdn
      gather_facts: true

      roles:
        - role: deekayen.cert_checker
          vars:
            cert_checker_email: 'david@example.com,norman@example.com'
            cert_checker_email_host: 'mail.example.com'
            cert_checker_notification_window: 31

## Configurations


## License and Authors

License:
Distributed under the Apache 2.0 license.

Author:
David Norman [deekayen](https://github.com/deekayen)
