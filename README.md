Windows Server 2016 DISA STIG
=========

![Build Status](https://img.shields.io/github/workflow/status/ansible-lockdown/Windows-10-STIG/CommunityToDevel?label=Devel%20Build%20Status&style=plastic)
![Build Status](https://img.shields.io/github/workflow/status/ansible-lockdown/Windows-10-STIG/DevelToMain?label=Main%20Build%20Status&style=plastic)
![Release](https://img.shields.io/github/v/release/ansible-lockdown/Windows-10-STIG?style=plastic)

Configure a Windows 10 system to be DISA STIG compliant. All findings will be audited by default. Non-disruptive CAT I, CAT II, and CAT III findings will be corrected by default. ~Disruptive finding remediation can be enabled by setting `win10stig_disruption_high` to `yes`.

This role is based on Windows 10 DISA STIG: [Version 2, Rel 2 released on April 27, 2021](https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_MS_Windows_10_V2R2_STIG.zip).

Updating
--------

As with all releases and updates, It is suggested to test and align controls.
This contains rewrites and ID reference changes as per STIG documentation.

Requirements
------------

Windows 10 - Other versions are not supported.

**General:**

- Basic knowledge of Ansible, below are some links to the Ansible documentation to help get started if you are unfamiliar with Ansible

  - [Main Ansible documentation page](https://docs.ansible.com)
  - [Ansible Getting Started](https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html)
  - [Tower User Guide](https://docs.ansible.com/ansible-tower/latest/html/userguide/index.html)
  - [Ansible Community Info](https://docs.ansible.com/ansible/latest/community/index.html)
- Functioning Ansible and/or Tower Installed, configured, and running. This includes all of the base Ansible/Tower configurations, needed packages installed, and infrastructure setup.
- Please read through the tasks in this role to gain an understanding of what each control is doing. Some of the tasks are disruptive and can have unintended consequences in a live production system. Also familiarize yourself with the variables in the defaults/main.yml file or the [Main Variables Wiki Page](https://github.com/ansible-lockdown/Windows-10-STIG/wiki/Main-Variables).

Documentation
-------------

- [Repo GitHub Page](https://ansible-lockdown.github.io/RHEL8-STIG/)
- [Getting Started](https://www.lockdownenterprise.com/docs/getting-started-with-lockdown)
- [Customizing Roles](https://www.lockdownenterprise.com/docs/customizing-lockdown-enterprise)
- [Per-Host Configuration](https://www.lockdownenterprise.com/docs/per-host-lockdown-enterprise-configuration)
- [Getting the Most Out of the Role](https://www.lockdownenterprise.com/docs/get-the-most-out-of-lockdown-enterprise)
- [Wiki](https://github.com/ansible-lockdown/Windows-10-STIG/wiki)

Dependencies
------------

The following packages must be installed on the controlling host/host where ansible is executed:

- passlib (or python2-passlib, if using python2)
- python-lxml
- python-xmltodict
- python-jmespath
- pywinrm

Package 'python-xmltodict' is required if you enable the OpenSCAP tool installation and run a report. Packages python(2)-passlib and python-jmespath are required for tasks with custom filters or modules. These are all required on the controller host that executes Ansible.

Role Variables
--------------

This role is designed that the end user should not have to edit the tasks themselves. All customizing should be done via the defaults/main.yml file or with extra vars within the project, job, workflow, etc. These variables can be found [here](https://github.com/ansible-lockdown/Windows-10-STIG/wiki/Main-Variables) in the Main Variables Wiki page. All variables are listed there along with descriptions.


Tags
----

There are many tags available for added control precision. Each control has it's own set of tags noting the control number as well as what parts of the system that control addresses.

Below is an example of the tag section from a control within this role. Using this example if you set your run to skip all controls with the tag ssh, this task will be skipped. The
opposite can also happen where you run only controls tagged with ssh.

```sh
tags:
    - WIN10-00-000050
    - volumes
    - ntfs
```

Branches
-------

- **devel** - This is the default branch and the working development branch. Community pull requests will pull into this branch
- **main** - This is the release branch
- **gh_pages** - github pages
- **all other branches** - Individual community member branches

Community Contribution
----------------------

We encourage you (the community) to contribute to this role. Please read the rules below.

- Your work is done in your own individual branch. Make sure to Signed-off and GPG sign all commits you intend to merge.
- All community Pull Requests are pulled into the devel branch
- Pull Requests into devel will confirm your commits have a GPG signature, Signed-off, and a functional test before being approved
- Once your changes are merged and a more detailed review is complete, an authorized member will merge your changes into the main branch for a new release.