# Windows 10 DISA STIG

## Configure a Windows 10 Enterprise system to be [DISA STIG](https://public.cyber.mil/stigs/downloads/) compliant.

### Based on [Windows DISA STIG Version 3, Rel 6 released on January 5th, 2026](https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_MS_Windows_10_V3R6_STIG.zip)

---

![Org Stars](https://img.shields.io/github/stars/ansible-lockdown?label=Org%20Stars&style=social)
![Stars](https://img.shields.io/github/stars/ansible-lockdown/Windows-10-STIG?label=Repo%20Stars&style=social)
![Forks](https://img.shields.io/github/forks/ansible-lockdown/Windows-10-STIG?style=social)
![Followers](https://img.shields.io/github/followers/ansible-lockdown?style=social)
[![X URL](https://img.shields.io/twitter/url/https/x.com/AnsibleLockdown.svg?style=social&label=Follow%20%40AnsibleLockdown)](https://x.com/AnsibleLockdown)

![Discord Badge](https://img.shields.io/discord/925818806838919229?logo=discord)

![Release Branch](https://img.shields.io/badge/Release%20Branch-Main-brightgreen)
![Release Tag](https://img.shields.io/github/v/tag/ansible-lockdown/Windows-10-STIG?label=Release%20Tag&color=success)
![Main Release Date](https://img.shields.io/github/release-date/ansible-lockdown/Windows-10-STIG?label=Release%20Date)

![Devel Commits](https://img.shields.io/github/commit-activity/m/ansible-lockdown/Windows-10-STIG/devel?color=dark%20green&label=Devel%20Branch%20Commits)

![Issues Open](https://img.shields.io/github/issues-raw/ansible-lockdown/Windows-10-STIG?label=Open%20Issues)
![Issues Closed](https://img.shields.io/github/issues-closed-raw/ansible-lockdown/Windows-10-STIG?label=Closed%20Issues&color=success)
![Pull Requests](https://img.shields.io/github/issues-pr/ansible-lockdown/Windows-10-STIG?label=Pull%20Requests)

![License](https://img.shields.io/github/license/ansible-lockdown/Windows-10-STIG?label=License)

---

## Looking For Support?

[Lockdown Enterprise](https://www.lockdownenterprise.com#GH_AL_WINDOWS_10_stig)

[Ansible Support](https://www.mindpointgroup.com/cybersecurity-products/ansible-counselor#GH_AL_WINDOWS_10_stig)

### Community

Join us on our [Discord Server](https://www.lockdownenterprise.com/discord) to ask questions, discuss features, or just chat with other Ansible-Lockdown users.

### Contributing

Bug reports and feature requests are welcome from everyone, please raise an issue.

Pull requests are accepted from approved contributors only. To be onboarded, join the [Discord Server](https://www.lockdownenterprise.com/discord) and request contributor access. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full process.

---

## Caution(s)

This role **will make changes to the system** which may have unintended consequences. This is not an auditing tool but rather a remediation tool to be used after an audit has been conducted.

Check Mode is not supported! The role will complete in check mode without errors, but it is not supported and should be used with caution.

This role was developed against a clean install of Windows 10. If you are implementing to an existing system please review this role for any site specific changes that are needed.

To use release version please point to main branch and relevant release for the STIG benchmark you wish to work with.

---

## Domain Members

Account policy is domain scoped. On a domain joined host the Default Domain Policy owns
`[System Access]`, so the 13 secedit backed controls in this role cannot hold there. `prelim.yml`
detects domain membership, skips those controls and warns once, rather than writing settings the
domain will not keep. Set them in the Default Domain Policy instead.

Everything outside `[System Access]` applies normally on a domain member.

This behaviour was verified on a domain joined workstation during Windows Fleet testing, where a
complete hardening run left all `[System Access]` values byte-identical to the pre-run baseline. It
has not been separately measured on Windows 10.

---

## Matching A Security Level For STIG

It is possible to only run controls that are based on a particular security level for STIG.
This is managed using tags:

- CAT1
- CAT2
- CAT3

The controls found in defaults/main also need to reflect those control numbers due to aligning every control to the audit component.

## Coming From A Previous Release

STIG releases always include changes, so it is highly recommended to review the new references and available variables. This process has evolved significantly since the initial release of Ansible-Lockdown.

Further details can be seen in the [Changelog](./CHANGELOG.md)

## Auditing (new)

Currently, this release does not have an auditing tool.

## Compliance facts

With `create_benchmark_facts` enabled (the default), the role writes a record of what it applied to:

```
C:\ProgramData\ansible\facts.d\compliance_facts.json
```

It captures the benchmark release, the run date and which CAT levels were enabled.

Windows has no default local-facts directory, so unlike the Linux roles this file is **not**
collected automatically. Ask for it explicitly:

```yaml
- name: Read the compliance facts
  ansible.windows.setup:
    fact_path: 'C:\ProgramData\ansible\facts.d'
```

It then appears as `ansible_compliance_facts` - not under `ansible_local`, which is the Linux
convention. Set `ansible_facts_path` to relocate the directory, or `create_benchmark_facts: false`
to skip writing it.

## Documentation

- [Read The Docs](https://ansible-lockdown.readthedocs.io/en/latest/)
- [Getting Started](https://www.lockdownenterprise.com/docs/getting-started-with-lockdown#GH_AL_WINDOWS_10_stig)
- [Customizing Roles](https://www.lockdownenterprise.com/docs/customizing-lockdown-enterprise#GH_AL_WINDOWS_10_stig)
- [Per-Host Configuration](https://www.lockdownenterprise.com/docs/per-host-lockdown-enterprise-configuration#GH_AL_WINDOWS_10_stig)
- [Getting the Most Out of the Role](https://www.lockdownenterprise.com/docs/get-the-most-out-of-lockdown-enterprise#GH_AL_WINDOWS_10_stig)

## Requirements

**General:**

- Basic knowledge of Ansible, below are some links to the Ansible documentation to help get started if you are unfamiliar with Ansible

  - [Main Ansible documentation page](https://docs.ansible.com)
  - [Ansible Getting Started](https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html)
  - [Tower User Guide](https://docs.ansible.com/ansible-tower/latest/html/userguide/index.html)
  - [Ansible Community Info](https://docs.ansible.com/ansible/latest/community/index.html)
- Functioning Ansible and/or Tower Installed, configured, and running. This includes all of the base Ansible/Tower configurations, needed packages installed, and infrastructure setup.
- Please read through the tasks in this role to gain an understanding of what each control is doing. Some of the tasks are disruptive and can have unintended consequences in a live production system. Also familiarize yourself with the variables in the defaults/main/main.yml file.

**Technical Dependencies:**

- Windows 10 Enterprise 22H2 - Other versions are not supported
- Running Ansible/Tower setup. This role requires ansible-core 2.16.1 or newer; the role asserts this at run time.
- Python3 Ansible run environment
- pywinrm

`pywinrm` is required on the controller host that executes Ansible; it is the connection library Ansible uses to reach a Windows target.

## Role Variables

### Breaking change in this release

Two security tunables still carried the legacy `wn10stig_` prefix. Role behaviour variables and
security tunables now all share the `win10stig_` prefix, matching the sibling Windows roles. If you
override either of these in inventory, group_vars or extra vars, rename them - the old names are no
longer read and your setting will be silently ignored.

- `wn10stig_internet_based_apps_to_check` becomes `win10stig_internet_based_apps_to_check`
- `wn10stig_pass_age_administrator` becomes `win10stig_pass_age_administrator`

Rule toggles are unchanged. They keep the `wn10_<control id>` form, for example
`wn10_au_000010`.

This role is designed so that the end user should not have to edit the tasks themselves. All customizing should be done via the defaults/main/main.yml file or with extra vars within the project, job, workflow, etc.

## Tags

There are many tags available for added control precision. Each control has its own set of tags noting what level, what OS element it relates to, whether it's a patch or audit, and the rule number. Additionally, NIST references follow a specific conversion format for consistency and clarity.

### Conversion Format for NIST References:

  1. Standard Prefix:

    - All references are prefixed with "NIST".

  2. Standard Types:

    - "800-53" references are formatted as NIST800-53.
    - "800-53A" references are formatted as NIST800-53A.
    - "800-53r4" references are formatted as NIST800-53R4 (with 'R' capitalized).

  3. Details:

    - Section and subsection numbers use periods (.) for numeric separators.
    - Parenthetical elements are separated by underscores (_), e.g., IA-5(1)(d) becomes IA-5_1_d.
    - Subsection letters (e.g., "b") are appended with an underscore.

### Example of Tag Usage:
Below is an example of the tag section from a control within this role. Using this example, if you set your run to skip all controls with the tag CCI-000018, this task will be skipped. Conversely, you can choose to run only controls tagged with CCI-000018.

```sh
tags:
      - WN10-AU-000040
      - CAT2
      - CCI-000018
      - CCI-000172
      - CCI-001403
      - CCI-001404
      - CCI-001405
      - CCI-002130
      - CCI-002234
      - SRG-OS-000004-GPOS-00004
      - SV-220752r958368_rule
      - V-220752
      - NIST800-53_AC-2_4
      - NIST800-53_AU-12_c
      - NIST800-53A_AC-2_4.1_i&ii
      - NIST800-53A_AU-12.1_iv
      - NIST800-53R4_AC-2_4
      - NIST800-53R4_AU-12_c
      - NIST800-53R4_AU-6_9
```
### Conversion Examples in Use:
  - NIST SP 800-53 :: AC-2 (4) -> NIST800-53_AC-2_4
  - NIST SP 800-53A :: AC-2 (4).1 (i&ii) -> NIST800-53A_AC-2_4.1_i&ii
  - NIST SP 800-53 Revision 4 :: AC-2 (4) -> NIST800-53R4_AC-2_4
  - NIST SP 800-53 :: AU-12 c -> NIST800-53_AU-12_c
  - NIST SP 800-53A :: AU-12.1 (iv) -> NIST800-53A_AU-12.1_iv
  - NIST SP 800-53 Revision 4 :: AU-12 c -> NIST800-53R4_AU-12_c

By maintaining this consistent tagging structure, it becomes easier to filter and manage tasks based on specific controls and compliance requirements.

## Community Contribution

Pull requests are accepted from approved contributors only, and issues are welcome from everyone.
See [CONTRIBUTING.md](CONTRIBUTING.md) for the onboarding process, the rules, and the commit signing
requirements (GPG signature and Signed-off-by on every commit).

## Local Testing

- Ansible
  - ansible-core 2.16.1 or newer, with Python 3
- `pywinrm` on the controller, which is the connection library Ansible uses to reach a Windows target

## Credits and Thanks

Massive thanks to the fantastic community and all its members.

This includes a huge thanks and credit to the original authors and maintainers.

