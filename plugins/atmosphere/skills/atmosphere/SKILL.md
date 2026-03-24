---
name: atmosphere
description: "Development workflow and contribution guidelines for the Atmosphere private cloud platform. Covers conventional commits, DCO sign-off, Helm chart modification via vendored upstream + patches, release notes with reno, Ansible role defaults formatting, and PR conventions. Use when contributing to, reviewing, or asking about the Atmosphere project."
---

# Atmosphere Development Guidelines

Atmosphere is a simple and easy private cloud platform featuring VMs, Kubernetes, and bare-metal provisioning. Repository: [vexxhost/atmosphere](https://github.com/vexxhost/atmosphere).

## Commits

Use conventional commits with DCO sign-off (`-s` flag):

```
type(scope): message

Signed-off-by: Name <email>
```

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`

## Pull Requests

PR titles must follow conventional commits format — they are used as the squash merge commit message.

## Release Notes

Generate a new release note with `reno new <slug>`. Keep notes brief and concise; detailed usage belongs in documentation.

Categories: `features`, `issues`, `upgrade`, `deprecations`, `critical`, `security`, `fixes`, `other`

Rules:
- Write natural English, not technical jargon (e.g. "DHCP client" not "dhclient")
- Wrap paths and technical terms in RST backticks (e.g. `` ``/etc/resolv.conf`` ``)
- Write from the project's perspective (e.g. "The default images..." not "Atmosphere default images...")
- Must pass `vale` linting

## Ansible Role Defaults

Brief description, blank line, then commented example, then default value:

```yaml
# Brief description of what this does.
#
# my_var:
#   - name: example
#     key: value
my_var: []
```

## Documentation

Always update documentation when adding features or improvements.

## Helm Charts

Atmosphere uses a vendor + patch workflow for reproducible, auditable chart modifications.

For detailed chart modification procedures, see [chart workflow](./chart-workflow.md).

### Quick Reference

1. Upstream charts are defined in `.charts.yml` (source of truth for versions)
2. Patches in `charts/patches/` track all customizations as discrete, reviewable changes
3. `charts/` contains the final result (upstream + patches applied)

To modify a chart: edit `charts/` and add corresponding patch to `charts/patches/`.
