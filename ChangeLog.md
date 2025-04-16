# Changelog

## Release 2.3.0

#### March 2025
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

## Release 1.0.0

#### December 2022
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
