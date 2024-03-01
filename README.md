# Certificate expiration checker

[![Molecule](https://github.com/deekayen/ansible-role-cert-checker/actions/workflows/ci.yml/badge.svg)](https://github.com/deekayen/ansible-role-cert-checker/actions/workflows/ci.yml) [![Project Status: Inactive – The project has reached a stable, usable state but is no longer being actively developed; support/maintenance will be provided as time allows.](https://www.repostatus.org/badges/latest/inactive.svg)](https://www.repostatus.org/#inactive)

This is a role used to inspect remote machines for listening TLS services based on their localhost loopback. This solves the problem of doing remote scans for services, which may be filtered to only listen to specific load balancers or security groups in public clouds.

## Requirements


## Role Variables


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
        - deekayen.cert_checker

## Configurations


## License and Authors

License:
Distributed under the Apache 2.0 license.

Author:
David Norman [deekayen](https://github.com/deekayen)
