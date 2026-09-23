# Changes to WIN10STIG

## Based on STIG v3.6.0 - ALD Windows Alignment Updates

### check_mode coverage

- FIXED: **the role produced nothing usable under `--check`.** `check_mode: false` makes a task run
  during a check pass; without it an audit task is skipped, its `register` is empty, and every later
  task keyed off that variable behaves as though the host reported nothing. This role had the
  annotation on 2 of its 118 registering shell tasks, against 91 to 118 across the Windows Fleet.
  112 read-only audit tasks now carry it, placed before `register:` to match the fleet pattern.
  Raised by `dsmorse` as a community contribution.

- FIXED: **the two `WN10-00-000085` audit tasks reported changed on every run.** Both only read
  local accounts, but neither declared `changed_when: false`. They now carry it, and
  `check_mode: false`.

- `WN10-00-000105` (SNMP permission fix) and `WN10-00-000140` (firewall remediation) keep no
  `check_mode: false`, deliberately: both change the system, where the annotation would make
  `--check` modify the host.

### Domain membership detection

- FIXED: **the role assumed domain joined whenever the membership fact was missing.**
  `discovered_domain_joined` is seeded `true` and `prelim.yml` only overwrites it
  `when: ansible_facts['windows_domain_member'] is defined`, so on any run that did not gather that
  fact the seeded value stood and roughly 35 gates across `WN10-00`, `WN10-CC`, `WN10-SO` and
  `WN10-UR` behaved as though the host were domain joined. Standalone-only branches never ran and
  domain-only settings were applied to standalone hosts. `prelim.yml` now gathers the
  `windows_domain` subset itself when the fact is absent, so the value is established rather than
  assumed.

- CHANGED: **`discovered_domain_joined` now defaults to `false`.** With the gather above in place
  the seeded value is only reached when membership genuinely cannot be determined, and treating
  such a host as standalone is the safer reading: standalone controls apply and domain-only
  settings do not. This is a behaviour change for any run where the fact is unavailable, which is
  why it is a separate commit from the gather. Matches the Windows Fleet.

- `README.md`: added a `Domain Members` section documenting that account policy is domain scoped,
  which controls are skipped on a domain joined host, and the instruction to set them in the
  Default Domain Policy instead.

### Runtime defects found in the Windows Fleet and carried across

A Windows Fleet role was run against a live domain member workstation in September 2026 and
five defects surfaced, every one of them present here in the same shape. None are reachable by
`ansible-lint`, `yamllint` or `--syntax-check`; each only fails at runtime, which is why a static
sweep of this role had not caught them.

**These fixes are ported, not independently exercised. They have not been run against a Windows 10
host.** Each was host proven on the fleet test host; here they are verified by static equivalence, lint and
a syntax check only.

- FIXED: **WN10-00-000140 would abort the play on the ordinary configuration.**
  `discovered_00_000140_firewall_installed` is set by a task behind a `when`, then referenced bare
  by the next task's `when`. The check keys off `Get-CimInstance root\SecurityCenter2
  FirewallProduct`, which enumerates third party firewall products only, so a host running nothing
  but Windows Defender Firewall returns an empty set, the guarded `set_fact` is skipped, and the
  run stops with `'discovered_00_000140_firewall_installed' is undefined`. That is the common case.
  The variable and its three siblings are now seeded in `vars/main.yml`. `WN10-00-000190` keeps its
  guard, because its `set_fact` runs the output through `from_json`, which fails on an empty
  string; the seeded default is what makes that guard safe.

- FIXED: **WN10-SO-000251 would abort the play on a bad expression.** It tested
  `register.values | length`, but `ansible.windows.win_reg_stat` returns `exists`, `properties` and
  `sub_keys` and has no `values` field. Because the register is a dictionary, `.values` silently
  resolves to the Python dictionary method rather than raising, so the mistake only surfaces when a
  filter is applied to it: `object of type 'method' has no len()`. Now tests `sub_keys`, which is
  what "populated" means for `Calais\Readers` and `Calais\SmartCards`. The six `.stdout_lines`
  references on the same two registers, which also do not exist on a `win_reg_stat` result, are
  corrected the same way.

- FIXED: **WN10-CC-000037 would sever the control connection.** It writes
  `LocalAccountTokenFilterPolicy=0`, filtering the privileged token of local accounts on network
  logon, so a run connected over WinRM or psrp as a local administrator loses the host as it
  applies. It presents as `the specified credentials were rejected by the server` rather than a
  dropped connection, because the port keeps answering and the service keeps running.

- FIXED: **WN10-UR-000070 would sever both transports at once.** It sets `SeDenyNetworkLogonRight`
  to `Guests, Enterprise Admins, Domain Admins, Local account`, removing the network logon right
  from every local account. Unlike WN10-CC-000037 this takes SSH with it, because Windows OpenSSH
  password authentication uses the same logon path. On the fleet test host every remote transport was tried
  after the equivalent control applied and all six failed, including SSH public key and WinRM
  certificate authentication; the console was the only way back.

  Note the deny list also names the two privileged domain groups, so **a Domain Admin does not
  survive it either**. The principal that does is a domain account that is neither a Domain Admin
  nor an Enterprise Admin, and that is a member of the local Administrators group on the target.

  Both controls now carry `not win_skip_for_test`, are listed with the other connection severing
  controls in `defaults/main/main.yml`, and are skipped automatically when the run connects as a
  local account, from two new facts in `prelim.yml`. The facts are deliberately separate:
  `discovered_control_account_is_local` is about the principal and applies on any transport, while
  `discovered_local_account_network_logon` adds the transport test. Collapsing them would either
  leave WN10-UR-000070 able to sever an SSH run, or stop WN10-CC-000037 applying over SSH where it
  is safe. The non-domain branch of WN10-UR-000070, which sets the right to `Guests` alone, is
  untouched.

- FIXED: **WN10-UR-000075 and WN10-UR-000080 would halt the play on an invalid group name.** Both
  passed `Enterprise Admin` and `Domain Admin` to `ansible.windows.win_user_right`. Neither is a
  group name - the Active Directory groups are `Enterprise Admins` and `Domain Admins` - so on a
  domain joined host the module fails with `Failed to translate the account 'Enterprise Admin' to a
  SID` and the run stops. `WN10-UR-000070` in the same file already used the correct plurals. A
  sweep of every account name passed to `win_user_right` across this role now returns only valid
  names, with no singular forms remaining.

- NOTE: the local account detection builds its backslash from a YAML single quoted variable rather
  than a Jinja literal. In a Jinja expression neither `'\'` nor `'\\'` matches one backslash in
  this position - the first is a syntax error, the second tests for two - so `DOMAIN\user` went
  undetected. The YAML form takes the character literally and never reaches Jinja's string parser.

- `README.md`: added the `Compliance facts` section, which the rest of the Windows Fleet already carried and this role did not. It documents where the facts
  file is written, that Windows collects no local facts by default so `fact_path` must be passed
  explicitly, and that the result appears as `ansible_compliance_facts` rather than under
  `ansible_local`. The Windows Fleet now carries an identical section.

- CHANGED: the compliance facts file is now JSON. `templates/compliance_facts.ps1.j2` is replaced by
  `templates/compliance_facts.json.j2`, and the role writes
  `C:\ProgramData\ansible\facts.d\compliance_facts.json` instead of `...\compliance_facts.ps1`.
  This aligns the ALD Windows roles on a single facts format. The
  file is now parsed rather than executed to produce the fact.

  **The fact name and its keys are unchanged.** `ansible.windows.setup` builds the key from the
  file's `BaseName`, so it is still `ansible_compliance_facts`, and the five existing keys plus the
  conditional `Cat_N_tag_run` entries keep their names and meanings. Anything already reading this
  fact continues to work.

  Every value goes through `| to_json`, so the CAT toggles are now real JSON booleans rather than
  PowerShell `$true` / `$false`. The conditional tag entries use a leading comma, because a trailing
  comma after `cat_3_hardening_enabled` would emit invalid JSON on the default untagged run.

- ADDED: a task removing a superseded `compliance_facts.ps1` before the new file is written. Both
  extensions are collected and both produce the same fact key, so on a host hardened by an earlier
  release the stale file would otherwise be a second source for `ansible_compliance_facts` with no
  guaranteed precedence.

- ADDED: a `managed_by` key. The PowerShell file opened with a "managed by ansible" comment banner
  and JSON cannot hold comments, so that provenance is recorded as a field instead of being lost. Its
  value is derived from `file_managed_by_ansible`, so the wording stays in one place and the variable
  remains in use.

- `win_template` now sets `newline_sequence: "\r\n"`, so the file is written with CRLF line endings.
  **Not executed.** No Windows inventory was available, so the file is never actually collected by
  `ansible.windows.setup` here. The reasoning about fact naming comes from reading the module source.
  The rendered output was validated offline against all eight CAT tag combinations.

- ADDED: **WN10-CC-000052 had no way to opt out of severing the connection.** It writes
  `EccCurves = NistP384 NistP256` to the Schannel configuration, restricting the curve list the TLS
  handshake can negotiate. Elsewhere in the fleet the equivalent control broke the WinRM TLS handshake on the
  following task and left the host unreachable mid-run. It now carries `not win_skip_for_test | bool`
  and is listed with the other connection affecting controls in `defaults/main/main.yml`, matching
  the fleet guard. `win_skip_for_test` still defaults to `false` here, so an ordinary run is
  unchanged; what is new is that a run over WinRM can opt out.

### Breaking changes

- BREAKING: `discovered_cloud_based_system` and `win10stig_cloud_vendors` are removed. Setting either
  in inventory, group_vars or extra vars is now **silently ignored** rather than raising an error.
  They existed only to choose between two implementations of the account lockout trio
  (`WN10-AC-000005`, `WN10-AC-000010`, `WN10-AC-000015`), and those two copies had converged on
  writing the same keys in the same order, so the choice no longer changed anything. The
  `cat2_cloud_lockout_order` **tag is also gone**; a run selecting it now matches nothing.

### Account policy scope and lockout ordering

- FIXED: the thirteen `[System Access]` controls written with `community.windows.win_security_policy`
  are now skipped on a domain joined host, gated on a new `discovered_account_policy_is_domain_scoped`
  fact set in `prelim.yml`. On such a host the Default Domain Policy owns that section and overwrites
  it at every policy refresh, so a local write reverted within roughly two hours while the run
  reported success, and an audit taken straight afterwards reported a compliance that did not last.
  The operator is warned once and the warning is counted through `warning_facts.yml`. The thirteen are
  the lockout trio, `AC-000020` through `AC-000045`, and `SO-000010`, `SO-000020`, `SO-000025` and
  `SO-000140`.
- The new fact is keyed on the observed `ansible_facts['windows_domain_member']`, deliberately not on
  `discovered_domain_joined`, which defaults to `true`. Keying it on that default would have skipped
  all thirteen controls on any host where the fact was missing.
- REMOVED: `tasks/cat2_cloud_lockout_order.yml`. The lockout trio now has a single implementation in
  `tasks/Cat2/WN10-AC-xxxxxx.yml`, still ordered `LockoutBadCount` -> `LockoutDuration` ->
  `ResetLockoutCount`. That order is load bearing and the reason is recorded in the file: secedit
  rejects the other two before a bad-count threshold exists, and rejects a reset counter greater than
  the current duration.
- The lockout trio now also honours `win_skip_for_test`, in line with the rest of the ALD Windows fleet. Changing
  the lockout threshold and duration mid-run can lock out the account Ansible is connected as.
- `prelim.yml` reads `ansible_facts['windows_domain_member']` rather than the bare
  `ansible_windows_domain_member`. `INJECT_FACTS_AS_VARS` is deprecated and removed in ansible-core
  2.24, so the bare form stops resolving.

  **Not executed.** No Windows inventory and no domain joined host was available, so the
  domain-scoped behaviour is statically verified, not exercised against a host.

- `README.md`: the `Community Contribution` section still described the old open-contribution model -
  "We encourage you (the community) to contribute to this role" and "All community Pull Requests are
  pulled into the devel branch". That contradicted both `CONTRIBUTING.md`, which states pull requests
  come from approved contributors, and this README's own `Contributing` section a few screens above
  it. Replaced with the wording the Windows Fleet already carried: pull requests from
  approved contributors, issues welcome from everyone, and a pointer to `CONTRIBUTING.md` for
  onboarding and the commit signing requirements. The Windows Fleet now carries an identical
  section.

- `README.md`: removed four controller-side dependencies the role does not use. It declared
  `passlib`, `python-lxml`, `python-xmltodict` and `python-jmespath`, and a paragraph describing an
  OpenSCAP tool installation. This role calls only `ansible.windows`, `community.windows` and
  `ansible.builtin` modules: there is no `password_hash` filter, no `xml` module, no `json_query`
  filter and no OpenSCAP task anywhere in it. `pywinrm` is retained and now carries the note, since
  it is the controller-side connection library. The only XML in the role is PowerShell
  `Get-AppLockerPolicy -Effective -XML` running on the target, which implies nothing about
  controller packages.
- `defaults/main/main.yml`: the `win_skip_for_test` comment read "consist of winrmvi host controls".
  Corrected to "WinRM", matching the wording the Windows Fleet carries.

- FIXED: three tasks in `tasks/Cat2/WN10-AU-xxxxxx.yml` were tagged `NIST800-53A_AC-12.1_iv` where
  the control is `AU-12 c`. Each sits beside `CCI-000172` and `NIST800-53_AU-12_c`, and the other 38
  occurrences in this role already used `AU-12.1`, so these three were typos. Corrected to
  `NIST800-53A_AU-12.1_iv`, which is what the Windows Fleet already carried.

- `README.md`: refreshed the stale and incorrect content.
  - The Ansible requirement read "tested against Ansible version 2.10.1 and newer" while
    `meta/main.yml` declared `2.16.1` and a `Local Testing` block claimed `2.18.2`. It now states the
    floor the role actually asserts in `tasks/main.yml`, and the `Local Testing` block is removed,
    matching the Windows Fleet.
  - The NIST tag documentation claimed all references are prefixed `NIST SP`. No tag in this role
    uses that prefix; they all begin `NIST`.
  - The `800-53 Revision 4` conversion example rendered a non-R4 tag, and the `AU-12.1 (iv)` example
    reproduced the `AC-12.1` typo above. Both corrected, and the tag example is now an exact
    reproduction of `WN10-AU-000040`, all nineteen tags.
  - Copy-edits: "a auditing tool", "a clean install of the Windows 10", a lowercase "stig", a comma
    splice and a garbled sentence in "Coming From A Previous Release", "Make sure to Signed-off", and
    a stray double space.

- `README.md`: the `Release Tag` and `Closed Issues` badge URLs used a redundant double ampersand
  (`&&color=success`). Corrected to a single `&`, matching the Windows Fleet. Every role now carries an identical badge block.
  Also dropped the commented-out Ansible Galaxy Quality badge: it carried project ID `61846`, the
  same ID the Win-10 role used, so it was wrong in one of the two and rendered nothing either way.

- FIXED: `.yamllint` now ignores `.ansible/`, matching the Windows Fleet. An `ansible-lint` run
  installs `collections/requirements.yml` into `.ansible/collections/`, and `yamllint` then walked
  that tree and linted several hundred vendored `ansible.windows` and `community.windows` files
  against this role's style rules. Only dependency code is excluded: role YAML is still linted.

- `README.md`: restored the release and activity badges, which were dropped with the pipeline badges
  earlier in this cycle. `Release Branch`, `Release Tag`, `Release Date`, `Devel Branch Commits`,
  `Open Issues`, `Closed Issues` and `Pull Requests` are back, and the block now matches
  the Windows Fleet exactly. They report on the public mirror
  `ansible-lockdown/Windows-10-STIG`, which is where the releases and issues live. The pipeline
  status badges are deliberately **not** restored.

### Breaking changes

- BREAKING: the two security tunables that still carried the legacy `wn10stig_` prefix are renamed
  to `win10stig_`, so every role behaviour variable and security tunable in this role now shares one
  prefix, matching the Windows Fleet. An override left under the old name is **silently ignored** rather
  than raising an error, so rename it in inventory, group_vars or extra vars:

  - `wn10stig_internet_based_apps_to_check` becomes `win10stig_internet_based_apps_to_check`
  - `wn10stig_pass_age_administrator` becomes `win10stig_pass_age_administrator`

  Rule toggles are untouched and keep the `wn10_<control id>` form.

### Documentation and branding alignment

- replaced `CONTRIBUTING.rst` with `CONTRIBUTING.md`, carrying the current Ansible-Lockdown
  contributing guide. The Windows Fleet now ships a byte-identical file
- `README.md`: added a Contributing section pointing at `CONTRIBUTING.md`, normalized the social
  badge to the `X URL` form on `x.com`, and pointed the Discord link at
  `https://www.lockdownenterprise.com/discord`
- `README.md`: aligned the shared heading text and the benchmark banner format with the rest of the
  Windows fleet. Role-specific sections are unchanged
- `LICENSE`: corrected the copyright year to 2026 and the company to
  `MindPoint Group - A Quantum Sky Company`. It read `Mindpoint Group - A Tyto Athene Company`
- `meta/main.yml`: `author` is now `Ansible-Lockdown Team`, `company` carries the full company line,
  and `min_ansible_version` is `2.16.1`, matching the other four Windows roles
- renamed `ChangeLog.md` to `CHANGELOG.md` so the filename matches the rest of the fleet, and
  updated the README link that pointed at the old name
- `templates/banner.txt`: the console banner read `A TYTO ATHENE COMPANY`. It now reads
  `A QUANTUM SKY COMPANY`, matching the other four roles
- `README.md`: fixed the repeated word and the stray preposition in the security level sentence
  (`possible to to ... a particular for security level`), which now matches the other four roles

### Benchmark alignment: V3R4 -> V3R6

Regenerated against `U_MS_Windows_10_STIG_V3R6_Manual-xccdf.xml` (Release 6, benchmark date
05 Jan 2026, 267 rules). Rule coverage is 267 of 267. V3R5 was never released by ALD, so this
covers the cumulative V3R4 -> V3R6 changeset.

- ADDED: eight advanced audit policy subcategory controls, all CAT II - `WN10-AU-000581` and
  `WN10-AU-000582` (Object Access >> File System, failure and success), `WN10-AU-000583` and
  `WN10-AU-000584` (Object Access >> Handle Manipulation), `WN10-AU-000586` and `WN10-AU-000589`
  (Object Access >> Registry), and `WN10-AU-000587` and `WN10-AU-000588` (Privilege Use >>
  Sensitive Privilege Use). Each follows the existing `AuditPol` read-then-set pattern and is
  gated on its own `wn10_au_0005xx` toggle, all defaulting to true.
- ADDED: `WN10-00-000126`, blocking consumer Microsoft account user authentication via
  `HKLM:\SOFTWARE\Policies\Microsoft\MicrosoftAccount\DisableUserAuth`. Toggle
  `wn10_00_000126`.
- RENUMBERED: `WN10-00-000125` is now `WN10-00-000107`. DISA renumbered the Copilot control and
  kept the same underlying rule (`SV-268315`), SRG and CCI, so this is a rename rather than a new
  control. **BREAKING:** the toggle `wn10_00_000125` is renamed to `wn10_00_000107`; rename it if
  you override it, because the old name is no longer read.
- REMOVED: three controls withdrawn in V3R6 and absent from the benchmark - `WN10-00-000145`
  (Data Execution Prevention set to OptOut), `WN10-00-000220` (Bluetooth turned off when not in
  use) and `WN10-SO-000005` (built-in administrator account disabled). Each was confirmed
  withdrawn rather than renumbered by checking that its SV identifier no longer appears anywhere
  in V3R6. **BREAKING:** the toggles `wn10_00_000145`, `wn10_00_000220` and `wn10_so_000005` are
  gone, along with the now-orphaned `wn10stig_dep_value` and `wn10stig_dep_optout_apps`
  variables, which only `WN10-00-000145` read.
- CHANGED: 20 SV identifiers were carrying superseded revision numbers and were bumped to their
  V3R6 values.
- CHANGED: `WN10-00-000040` moves from `CCI-000366` to `CCI-003376`. Its NIST tag follows the CCI
  and is now the single tag `NIST800-53R4_SA-22_a`; the DISA CCI list maps `CCI-003376` only to
  NIST SP 800-53 Revision 4 `SA-22 a`, with no Revision 3 or 800-53A equivalent, so the previous
  `NIST800-53_CM-6_b` and `NIST800-53A_CM-6.1_iv` tags no longer apply.
- ADDED: the rev-5 CCI that V3R6 lists alongside the legacy CCI for twelve controls, which had
  been carrying only the legacy value: `CCI-004066` on `WN10-00-000090`, `WN10-AC-000025`,
  `WN10-AC-000030`, `WN10-AC-000035` and `WN10-SO-000280`; `CCI-004062` on `WN10-AC-000045` and
  `WN10-SO-000195`; `CCI-003938` on `WN10-AU-000045` and `WN10-AU-000050`; `CCI-003980` on
  `WN10-CC-000310` and `WN10-CC-000315`; and `CCI-003627` on `WN10-00-000065`. These have no
  Revision 4 mapping, so the existing `NIST800-53R4_` tags are unchanged.
- FIXED: two SRG identifiers. `WN10-CC-000350` was tagged `SRG-OS-000393-GPOS-00174` where V3R6
  assigns `SRG-OS-000393-GPOS-00173`, and `WN10-SO-000080` was tagged `SRG-OS-000228-GPOS-00088`
  where V3R6 assigns `SRG-OS-000023-GPOS-00006`.
- FIXED: two CAT tags disagreed with the benchmark severity. `WN10-00-000032` is CAT I in V3R6 but
  was tagged `CAT2`, and `WN10-CC-000080` is CAT III but was tagged `CAT2`. Both were already in
  the correct task file, so only the tag changed.
- FIXED: `WN10-00-000045` carried the tag `WIN10-00-000045`. The `WIN10-` prefix is not the rule
  ID, so `--tags WN10-00-000045` never selected this control.
- CHANGED: ten task titles were aligned with their V3R6 wording, including `WN10-00-000107`
  ("Copilot must be disabled"), `WN10-CC-000235` ("unverified files" rather than "malicious
  websites") and `WN10-CC-000050` (which now names the `\\*\SYSVOL` and `\\*\NETLOGON`
  shares). `WN10-CC-000235` already wrote the correct registry value
  (`PreventOverrideAppRepUnknown`), so only the title was stale.
- ADDED: `benchmark_version` (`v3.6.0`) and `benchmark` (`Windows-10-STIG`) to `defaults/main.yml`,
  matching the metadata the rest of the Windows fleet carries.
- `README.md`: the benchmark banner and download link now point at V3R6.

### Repository hygiene

- FIXED: **the pipeline workflows could hand repository secrets to a fork and could leave the Azure
  instance running.** `build-azure-windows` now requires the pull request to originate from a branch
  in this repository, because `pull_request_target` grants that job the repository secrets and its
  steps check out pull request head and execute it. `Tofu Destroy` was gated on
  `ENABLE_DEBUG == 'false'`, which is false for an unset variable, so the instance survived the run.
  It is now `!= 'true'`. Both jobs also declare least-privilege permissions, where neither declared
  any.
- FIXED: **three defects that stopped the pipelines working as written.**
  `if [ ${{ vars.IAC_BRANCH }} != '' ]` expands to `if [ != '' ]` when the variable is unset and is
  now the quoted `-n` form; the debug step echoed `$benchmark_type`, which is never defined, and now
  echoes `$TF_VAR_benchmark_type`; and `actions/first-interaction` tracked `@main` with the
  hyphenated `repo-token` and `pr-message` inputs it renamed after v1, so the welcome comment had
  silently stopped posting. It is pinned to `v3.1.0` with the current input names and the
  `issue_message` its runtime requires. `actions/checkout` moves to v7 and both workflows gain
  `workflow_dispatch`.
- REMOVED: **`update_galaxy.yml`.** It has never run, and no Windows STIG role is published on
  Ansible Galaxy, so it was not the mechanism keeping anything current.
- FIXED: `.gitignore` ignored `.github/` wholesale, which is why a workflow file had to be
  force-added and why any new one would have been silently untracked. Replaced with the narrow
  runtime-checkout patterns the rest of the Windows Fleet uses
  (`.github/workflows/github_windows_IaC/` and `.github/.ansible/`), which carry a comment warning
  against the wholesale form.
- `README.md`: removed the `Public Repository`, `Lint & Pre-Commit Tools`,
  `Community Release Information` and `Subscriber Release Information` sections and the 15 badges
  they held, including both pipeline-status badges and every
  `ansible-lockdown.github.io/github_windows_IaC` endpoint badge. The remaining badges are a single
  flat block matching the other four Windows roles: organisation and repository stars, forks,
  followers, X, Discord and licence. This role was the only one of the five carrying those four
  headings.

### Modernization to the fleet pattern

The role now matches the Windows Fleet structurally. Nothing below changes which controls
run or what they write, except where marked BREAKING or FIXED.

- CHANGED: the task files are split from flat `tasks/cat1.yml`, `cat2.yml` and `cat3.yml` into
  `tasks/Cat1`, `tasks/Cat2` and `tasks/Cat3`, one file per control-ID band, each imported by a
  `main.yml` index in its own directory. 19 family files replace 3 flat ones. Verified
  content-neutral: every one of the 268 top-level task blocks is byte-identical before and after,
  with none dropped, duplicated or altered, and the per-band counts unchanged at 28, 222 and 18.
- BREAKING: `win10stig_cat1_patch`, `win10stig_cat2_patch` and `win10stig_cat3_patch` are renamed
  to `win10stig_cat1_controls`, `win10stig_cat2_controls` and `win10stig_cat3_controls`, matching
  the Windows Fleet and the names the tasks read. Rename them if you override them; the old
  names are no longer read.
- BREAKING: `win10stig_min_ansible_version` is renamed to `min_ansible_version` and moved from
  `defaults/` to `vars/main.yml`, matching the rest of the Ansible Lockdown fleet. Rename it if you
  override it; the old name is no longer read.
- CHANGED: `defaults/main.yml` moved to `defaults/main/main.yml`, the directory layout the Linux
  roles have adopted. Ansible loads role defaults from the directory, so no variable name or value
  changed. Verified by loading the defaults tree through Ansible's role-defaults loader.
- FIXED: every rule toggle and operational switch used in a `when:` is now filtered through
  `| bool`, so they can be set from the command line, where an override arrives as a string.
  271 conditions.
- CHANGED: all 127 registers move from `wn10_<control id>_<description>` to
  `discovered_<control id>_<description>`, the fleet's register namespace. The control ID is kept
  because stripping it would collapse 127 names into a handful. Internal only; registers are not
  consumer-facing.
- ADDED: `skip_os_check`, defaulting to false. Setting it true bypasses the OS version and family
  assert for hosts the check misidentifies.
- ADDED: `create_benchmark_facts` and `ansible_facts_path`. The role now writes
  `compliance_facts.ps1` under `C:\ProgramData\ansible\facts.d`, recording the benchmark
  release, the run date and which CAT levels were enabled. Unlike Linux, Windows has no default
  local-facts directory, so the file is collected only when `ansible.windows.setup` is given a
  matching `fact_path`; it then surfaces as `ansible_compliance_facts`.
- ADDED: `company_title` and `file_managed_by_ansible` in `vars/main.yml`, the provenance header
  the fleet templates use. The `vars` file header also read `# vars file for .` and now names the
  role.
- FIXED: the role read 41 facts through the injected top-level names (`ansible_distribution` and
  friends). `INJECT_FACTS_AS_VARS` defaulting to true is deprecated and the behaviour is removed in
  ansible-core 2.24, at which point those names become undefined. All 41 now read
  `ansible_facts['...']`. `ansible_version` and `ansible_run_tags` are deliberately untouched:
  they are magic variables, not facts, and have no `ansible_facts` equivalent.
- FIXED: the OS check. It gated the distribution gather on `ansible_distribution is not defined`,
  which can never fire because `ansible.windows`'s `setup.ps1` assigns the fact unconditionally and
  only fills it when the connection is administrative, so the assert then read `None`. It also used
  `regex_search`, which returns `None` rather than false on a miss and raises outright when handed
  that `None`. The check now tests a falsy distribution as well as an undefined one, uses a
  substring test, reads `os_installation_type` to separate a client SKU from a Server one (Windows
  10 1607 and Windows Server 2016 both report build 10.0.14393, so no build range can do it), and
  defaults every fact it reads so a failure reports the reason instead of collapsing into a
  template error.
- REMOVED: the `MAIN | Get Disk Facts` task. It called `community.windows.win_disk_facts` on every
  run, but nothing consumed the result: `WN10-00-000050` and `WN10-00-000130` each run their own
  `win_shell`. It also carried no tags, so it ran even on tag-filtered invocations.
- CHANGED: `collections/requirements.yml` is pinned to `ansible.windows >= 3.7.0` and
  `community.windows >= 3.3.0` rather than tracking git HEAD, so a collection change cannot alter
  this role's behaviour between runs. `community.general` is removed from both the requirements
  file and `meta/main.yml`: no task in this role calls it. `ansible.windows` accounts for 386
  module calls and `community.windows` for 16.
- CHANGED: `.ansible-lint` skips the `complexity` rule, matching the Windows Fleet. The
  one-file-per-family layout puts 105, 135 and 133 tasks in three Cat2 files against the rule's
  hard-coded limit of 100. The limit is not settable from a config file and splitting the families
  further would break the naming convention shared across the fleet. This disables the rule
  repo-wide, so a new file crossing 100 tasks is not reported.
- CHANGED: single-entry `when:` and `tags:` are written inline rather than as one-item lists.
  Conditions whose single item is folded across several lines stay as blocks.

### Remediation defects fixed

Each of these predates this release and would have been reached on an ordinary run.

- FIXED: `WN10-AC-000005`, `WN10-AC-000010` and `WN10-AC-000015` were gated on
  `discovered_cloud_based_system` **positively**, the same condition as the dispatcher that imports
  `cat2_cloud_lockout_order.yml` in their place. The two were meant to be mutually exclusive. As
  written, a host detected as non-cloud ran none of the three at all, and a host detected as cloud
  ran each of them twice, the second pass in the order the companion file exists to prevent. The
  three inline blocks now carry `not`.
- FIXED: the inline lockout path applied `ResetLockoutCount` before `LockoutDuration`. `secedit`
  rejects a reset counter greater than the current duration, so on a host still at the Windows
  default duration of 10 the control fails with "Failed to import secedit.ini file ... The
  parameter is incorrect". Both values default to 15, so this would hit any host not already raised
  to 15 or above. The order is now BadCount -> Duration -> Reset on both paths.
- FIXED: cloud detection read `not virtualization_type == 'VMware' or (...)`. Because `not` binds to
  the comparison and the clauses are joined by `or`, the fact was raised on **every host that is not
  VMware**, including bare metal and a laptop running VirtualBox. Detection now requires positive
  evidence: a `system_vendor` in the new `win10stig_cloud_vendors` list and a virtualization type of
  `Hyper-V`, `kvm` or `xen`. Set `discovered_cloud_based_system` in inventory to override it.
- FIXED: `WN10-SO-000251` wrote four registry paths as `HKLM\SOFTWARE\...` with no colon after the
  hive. Both `win_regedit` and `win_reg_stat` reject a path that does not match
  `^HK(CC|CR|CU|LM|U):\\` and fail the task outright. The control is gated on
  `discovered_domain_joined`, which defaults to true, so it would abort on any domain-joined host.
  171 other paths in the role already carried the colon.
- FIXED: the non-persistent VDI test read `discovered_vdi_persistence`, a register created nowhere
  in this role. Guarded by `default('')` it evaluated as an empty string rather than erroring, so
  the non-persistent clause was always satisfied and the persistent fact could never be raised.
  Ten controls are gated on these facts, three of them CAT I, so ordinary virtual machines were
  silently skipping BitLocker and Credential Guard. Detection now requires positive evidence of a
  managed disposable desktop: an Azure Virtual Desktop cloud domain join, or FSLogix profile
  redirection. Disk media type is no longer consulted.
- FIXED: the non-persistent and persistent VDI status messages were transposed, so an operator
  reading the output was told the opposite of what the role had detected.
- FIXED: `WN10-00-000140` gated its auto-remediation log and part of its warn count on
  `win10stig_auto_remediate`, a variable defined nowhere. Guarded by `default(false)` it evaluated
  false permanently rather than erroring, so when auto-remediation did delete unauthorized inbound
  firewall rules the operator got no record of which rules were removed. The switch that actually
  gates the removal, and which the message text already names, is
  `win10stig_remove_unauthorized_hosts`; both references now read it.
- FIXED: `WN10-CC-000080` is CAT III in V3R6 but all four of its task names read `MEDIUM`. Its
  `CAT3` tag and task file were already correct, so a tag-only check passed it.

### Remediation defects fixed - second pass

Found by a full `/gpo-parity` sweep. Where the Windows Fleet had already fixed the same
defect, its fix was adopted rather than a new one written; the rest are fleet-wide and are noted as
such.

- FIXED: `WN10-CC-000391` removed Internet Explorer with `ansible.windows.win_feature`, which wraps
  `ServerManager` and is documented as unavailable on client operating systems, so it could never
  succeed in a role that asserts a client SKU. It now uses `win_optional_feature`, the module the
  role already uses for six other optional features. Adopted from Win-11.
- FIXED: all six `win_optional_feature` removals are now gated on the feature being present.
  `win_optional_feature` raises "Failed to find feature" rather than no-opping, and Windows removes
  optional features between releases. `prelim.yml` gains the optional-feature enumeration and the
  `discovered_feature_names` fact, both adopted from Win-11.
- FIXED: `WN10-00-000210` never wrote the value V3R6 checks. The benchmark specifies
  `HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device\Connectivity\AllowBluetooth` = 0; the
  role wrote a QuickActions value and disabled `bthserv` instead, so a V3R6 scan would still flag
  the control after a successful run. Adopted from Win-11.
- FIXED: `WN10-00-000010` carried four defects in one block. Its warning fired on the *compliant*
  TPM state because the conditions tested for the healthy values being present rather than absent.
  All four assertions read `stderr_lines`, where `wmic` writes nothing but its "No Instance(s)"
  diagnostic, so they could never match and the inversion was masked. The warn-count task's final
  `or` disjunct was unreachable, so a host with no TPM was never counted. And the probe used
  `wmic`, which is deprecated on Windows 10 and is a Feature-on-Demand that can be absent, leaving
  the register without `stdout`/`stderr` keys. The probe now uses `Get-CimInstance`, reads
  `stdout`, and warns when the TPM is *not* compliant. Adopted from Win-11, except that Win-10
  keeps its `discovered_domain_joined` and VDI gates: unlike the fleet equivalent, the V3R6 check
  text says "For standalone or nondomain-joined systems, this is NA" and grants the VDI exemption.
- FIXED: `WN10-00-000130` referenced a bare `filtered_certificate_files` at its alert task, a name
  defined nowhere and carrying no `default()`, so the task raised an undefined-variable error when
  reached. The set_fact eleven lines above defines
  `discovered_00_000130_filtered_certificate_files`. Reached whenever
  `win10stig_auto_remediate_files` is false, which the surrounding message itself tells the operator
  to do. **The same defect is still present elsewhere in the Windows Fleet.**
- FIXED: four `AUDIT`-labelled tasks performed writes. `--tags audit` is a real selection
  mechanism, so they handed ACL changes and registry writes to an operator who asked for a
  read-only pass. `WN10-00-000105` took ownership of `C:\Windows\System32\snmp.exe` and granted
  Administrators full control while also declaring `changed_when: false`, so it mutated a system
  binary and reported `ok`; it is now `PATCH` with `changed_when: true`. `WN10-AU-000083` and
  `WN10-AU-000107` ran `AuditPol /set` and are now `PATCH`, matching their own structural twins.
  **The same defect is still present elsewhere in the Windows Fleet.**
- FIXED: `WN10-AU-000585` also wrote `ProcessCreationIncludeCmdLine_Enabled`, which is
  `WN10-CC-000066`'s value and is already applied independently by that control. The duplicate
  write, filed under the wrong control and under an `AUDIT` label, is removed; `WN10-AU-000585`
  keeps only the `AuditPol` work its own check text asks for. **The same duplicate is still present
  elsewhere in the Windows Fleet.**
- REMOVED: three dead `prelim.yml` fact chains consumed nowhere in the role -
  `discovered_tpm_present` (with its `discovered_tpm_info` gather), `discovered_rdp_enabled` and
  `discovered_admin_accounts`. The TPM set_fact also called `from_json` on the output of a
  `failed_when: false` probe, which raises on an empty string. None of the three exists in Win-11.
  The role's remaining two unconsumed registers are the same two Win-11 carries.

### Defects fixed while aligning

These predate this release and were reachable on a normal run.

- FIXED: `WN10-PK-000015` referenced `wn10_pk_000015_dod_root_3_2022_check` in a `when:`, a
  register that is created nowhere in the role. Jinja's `or` short-circuits, so the undefined name
  was only evaluated when the left operand was false - that is, on a host where the DoD Root CA 3
  check passed. The task therefore failed on compliant hosts and passed on non-compliant ones.
  V3R6 checks exactly one certificate, so the operand was dead code and has been removed.
- FIXED: the same control's warn count was recorded against `WN10-PK-000010`, a different control
  covering the ECA Root CA certificates in the trusted root store. It now records
  `WN10-PK-000015`.
- FIXED: `WN10-00-000250` referenced `wn10_00_000250_session_timeout_value` in two `when:`
  clauses. That variable is defined nowhere; the value the rest of the block reads is
  `win10stig_non_persistent_max_session_timeout`. Because the reference sat first in a `when:`
  list there was no short-circuit, so the control failed outright on any non-persistent or
  persistent VDI host, where its enclosing condition is met.
- FIXED: the `WN10-00-000032` task name was missing its opening quote while carrying a closing
  one, so the rendered task name ended with a stray `"`.
- FIXED: three `WN10-UR-000035` task names contained `NT SERVICE\autotimesvc` inside a
  double-quoted scalar, where `\a` is YAML's bell escape. The rendered names carried a `0x07`
  control character and read `NT SERVICEutotimesvc`. The backslash is now escaped.
- FIXED: `.ansible-lint` declared `parseable: true`, a key current ansible-lint rejects, so every
  lint run aborted on a schema error and the role was effectively unlinted. With the key removed
  `ansible-lint` exits 0 with no rule findings.

## Based on STIG v3.4.0 - Release 2.1.1 - May 2025

General Updates
  - Updated Readme
  - Added New Workflows

## Based on STIG v3.4.0 - Release 2.1.0 - April 2025

General Updates
  - Updated Release To V3R4 STIG
  - Verified All NIST Tagging
  - Updated WN10-AU-000005
  - Rule IDs Updated For Changes
  - New Control WN10-SO-000110

## Based on STIG v3.3.0 - Release 2.0.0 - March 2025

General Updates
  - First Release for V3R3 STIG
  - Removed state: present from all win_regedit modules.
  - Added NIST Tagging
  - Added New Workflows

## Release 1.1.0

#### August 2023
  - Updated Workflows To Central Repo
    - Renamed them to better run across all repos.
  - Removed Templates & PR Template from repo and adjusted to Org level.
  - Updated Readme Layout to add new pipeline badges.
  - Cat2_Cloud moved from tasks/main and renamed to cat2_cloud_lockout_order and in cat2.yml workflow.
  - Updated Tags in tasks/main.

#### May 2023
  - Updated Pipelines For Testing
  - Added Banner
  - Added Skip For Testing to controls that will break in cloud.
  - Added Support for Azure for Controls that break.

## Based on STIG v2.5.0 - Release 1.0.0 - December 2022

  - Updated Readme
  - Added Changelog.md and updated.
  - Added Version 2 Release 3 changes during this update.
  - Added Version 2 Release 4 changes during this update.
  - Added Version 2 Release 5 changes during this update.
  - WN10-00-000030, WN10-00-000031, WN10-00-000032 - Changed Check text from
    “WVD” to “AVD” for Azure Virtual Desktop.
  - WN10-00-000040 - Updated Check and Fix: Windows servicing levels need to be updated.
  - WN10-AU-000550 - Removed requirement.
  - WN10-AU-000570 - Updated Fix text: Object Access >> “Audit Detailed File
    Share” with “Failure” selected.
  - WN10-CC-000007 - Updated Check and Fix registry label and settings with
    Value Name: Value; Value Data: Deny.
  - WN10-CC-000050, WN10-SO-000280 - Rule ID changed in data management system.
  - WN10-CC-000080 - Added requirement back to STIG per government and SHB.
  - WN10-CC-000327 - Rule ID changed due to reparenting SRG ID.
  - WN10-PK-000005, WN10-PK-000015 - Removed all deprecated DoD Root CA 2 references.
  - WN10-SO-000251 - Changed Check text: If the system is “not” a member of a domain, this
    is Not Applicable.
  - WN10-AU-000555 - Updated control because of removal of WN10-AU-000550
  - Update Rule ID's As Needed
  - Checked All Cat I, II, III Controls.
  - Removed WN10-EP Controls From Nov 1st 2021 Update
  - Added Version 2 Release 5 changes during this update.
  - WN10-00-000005, WN10-CC-000050 - Changed wording in the Check text from “standalone” to “standalone or nondomain- joined”.
  - WN10-00-000010, WN10-CC-000115, WN10-CC-000130, WN10-CC-000206, WN10-UR-000075, WN10-UR-000080 - Changed wording in the Check and Fix text
    from “standalone” to “standalone or nondomain-joined”.
  - WN10-00-000030, WN10-00-000031, WN10-00-000032, WN10-PK-000005, WN10- PK-000015 - Corrected CCIs.
  - WN10-00-000040 - Updated Check and Fix text regarding versioning.
  - WN10-CC-000055 - In Check and Fix text, set Minimize simultaneous connections to Enabled; set Minimize Policy Options to 3, Prevent Wi-Fi
    while on Ethernet.
  - Some Rule IDs and CCIs updated due to minor changes in content management system.
  - Added Warning Count For End Of Playbook
